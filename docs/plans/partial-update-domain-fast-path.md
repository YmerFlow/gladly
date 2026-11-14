# Plan: Partial-update fast path for domain-only config changes

## Overview

`Plot.update({ config })` is all-or-nothing. If the new config is **byte-identical** to the current
one it is a no-op (`src/core/Plot.js:283`, `JSON.stringify(config) === JSON.stringify(this.currentConfig)`);
otherwise it runs a full `_initialize()` — new `AxisRegistry`, re-run transforms, re-create every
layer, re-upload GPU buffers. There is no middle ground between "identical → nothing" and "any
difference → rebuild everything".

This is wasteful for the most common config change of all: an **axis domain** (`min` / `max`). A
zoom or pan writes its new range into `currentConfig.axes` via `setAxisDomain`
(`src/core/Plot.js:772-775`). In a typical host app the config is held in external state and
re-applied whenever it changes, so that range round-trips back through `update()` — and a plain zoom
pays for a complete teardown and rebuild of the plot, despite nothing structural having changed.

This plan adds a third case to the update path: when the only differences between the old and new
config are axis domain bounds, apply them in place via `setAxisDomain` and skip the rebuild.

---

## Cost

The compare is cheap — cheaper than the `JSON.stringify` already on this path. The config is small
(a few layers, ≤4 spatial axes plus a few color/filter axes, some colorbars/transforms). The
**data** is not involved: it is already compared by identity (`data === this._lastRawDataArg`,
`src/core/Plot.js:284`), not serialized. The work avoided — a full `_initialize()` — costs
milliseconds to tens of ms in GPU uploads and draw-command construction; the compare costs
microseconds. The real cost is **correctness of the carveout list**, not CPU.

---

## Design

### Classification: structural vs. in-place

Compare old and new config. A difference is **in-place** only if it is confined to carved-out paths;
otherwise it is **structural** and forces a full rebuild.

**Carved-out (in-place) — start with exactly one field:**
- `axes[id].min` and `axes[id].max`, **numeric → numeric** only.

**Everything else is structural** and must rebuild: `quantity_kind` (registry slot), `scale`
(linear↔log → new d3 scale), adding/removing an axis key, `layers`, `transforms`, `colorbars`,
`interactions`, and any other axis field.

### The present↔absent subtlety (important)

A `min`/`max` going **present → absent** means "revert to auto-domain", which requires
`applyAutoDomainsFromLayers` (`src/core/Plot.js:1107`) — not a plain `setDomain`. So the carveout is
**numeric → numeric only**: an in-place change is allowed for an axis only if *both* old and new
configs have *both* bounds defined as numbers. Any transition that adds or removes a bound is treated
as structural (rebuild), which is always safe.

### Recommended implementation — strip-and-stringify

Do not hand-roll a recursive differ; reuse the existing stringify on a reduced copy.

```js
// Remove the carved-out fields from every axis, then compare the remainder.
const strip = c => ({
  ...c,
  axes: Object.fromEntries(
    Object.entries(c.axes ?? {}).map(([k, a]) => {
      const { min, max, ...rest } = a ?? {}
      return [k, rest]
    })
  ),
})
const structuralSame =
  JSON.stringify(strip(newConfig)) === JSON.stringify(strip(this.currentConfig))
```

- `structuralSame === false` → full `_initialize()`, exactly as today.
- `structuralSame === true` and the configs are otherwise equal → current no-op path
  (`scheduleRender()`).
- `structuralSame === true` but they differ (only `min`/`max` changed) → **in-place path**: for each
  axis whose numeric bounds changed *and* satisfy the numeric→numeric rule, call
  `setAxisDomain(axisId, [min, max])`; set `this.currentConfig = newConfig`; `scheduleRender()`. If
  any changed axis fails the numeric→numeric rule, fall back to the full rebuild.

> **Decision point:** strip-and-stringify (above) vs. an explicit recursive walk that returns the set
> of differing leaf paths. Strip-and-stringify is ~10 lines, reuses existing machinery, and the
> carveout is one list of fields to omit; the recursive walk is more general (per-path
> classification) but more code. Recommended: strip-and-stringify until a second carved-out field with
> different semantics forces the general version.

### Placement — `_applyUpdate`, not `update`

The existing short-circuit lives in `update()` (`src/core/Plot.js:283-288`), but `PlotGroup.update`
calls `_applyUpdate` directly (`src/core/PlotGroup.js:85`), bypassing it — so group updates already
always rebuild. Put the classification at the **top of `_applyUpdate`** (`src/core/Plot.js:239`), so
both direct `update()` and `PlotGroup.update` benefit, and so the no-op and in-place fast paths are
decided in one place before `currentConfig` is overwritten.

Sketch:
```js
async _applyUpdate({ config, data } = {}) {
  const dataChanged = data !== undefined && data !== this._lastRawDataArg
  if (config !== undefined && !dataChanged && this.currentConfig && this._rawData) {
    const decision = this._classifyConfigChange(config)   // 'noop' | 'domains' | 'rebuild'
    if (decision === 'noop')    { this.currentConfig = config; this.scheduleRender(); return }
    if (decision === 'domains') { this._applyDomainDeltas(config); this.currentConfig = config; this.scheduleRender(); return }
  }
  // ... existing full-rebuild body unchanged ...
}
```
(The exact short-circuit in `update()` can then delegate to this, or be left as a cheap pre-filter.)

### In-place application details

- `setAxisDomain` already updates both the registry and `currentConfig.axes`
  (`src/core/Plot.js:767-782`); after applying deltas, overwrite `this.currentConfig = config` so the
  stored config matches exactly what the caller passed (avoids drift in untouched fields).
- `setAxisDomain` is synchronous, so the in-place path completes atomically within the call.
- Linked axes: `setAxisDomain` → registry `setDomain`; whether a domain delta from a config
  round-trip should propagate to linked plots depends on existing link semantics. **Decision point:**
  confirm the intended behaviour. If the host app is the source of truth and already drove the link,
  re-propagating could double-apply; if not, it should propagate. Match whatever `setAxisDomain` does
  today and document it.

---

## Implementation steps

1. Add `_classifyConfigChange(newConfig)` → `'noop' | 'domains' | 'rebuild'` using strip-and-stringify
   plus the numeric→numeric check per changed axis.
2. Add `_applyDomainDeltas(newConfig)` that diffs `axes[*].{min,max}` against `this.currentConfig` and
   calls `setAxisDomain` for each numeric→numeric change.
3. Hook both at the top of `_applyUpdate` (`src/core/Plot.js:239`), before sizing / `_initialize`.
4. Leave `update()`'s existing equality short-circuit in place (or thin it to defer to
   `_classifyConfigChange`).
5. No change to `_initialize` or the registry.

---

## Testing

- **Domain-only delta skips rebuild:** spy on `_initialize` (or on a layer's buffer creation); call
  `update({ config })` with only `axes.xaxis_bottom.{min,max}` changed; assert `_initialize` was not
  called, the registry domain updated, and a render was scheduled.
- **Structural change still rebuilds:** change `axes.xaxis_bottom.scale` linear→log (and separately:
  add a layer, change a `quantity_kind`); assert `_initialize` ran.
- **present→absent falls back to rebuild:** remove `max` from an axis that had one; assert
  `_initialize` ran (auto-domain recompute), not an in-place `setDomain`.
- **absent→present falls back to rebuild:** add a `max` to an axis that had none; assert rebuild.
- **No-op unchanged:** byte-identical config still short-circuits to a plain render, no `_initialize`.
- **Data change forces rebuild:** a new-but-equal data object (identity differs) rebuilds regardless
  of config.
- **Mixed delta:** `min/max` plus a structural field changed together → rebuild.
- **Zoom round-trip does not rebuild:** simulate the host pattern — zoom (writes `currentConfig.axes`
  via `setAxisDomain`), read `getConfig()`, feed it back through `update({ config })`; assert no
  `_initialize`.

Update `docs/configuration/PlotConfiguration.md` to document that `update()` applies axis `min`/`max`
changes in place (no rebuild), while any other config change rebuilds.

---

## Risks and trade-offs

- **Carveout correctness is the whole risk.** Marking a field in-place that actually needs a rebuild
  produces stale renders. Mitigation: ship with `min`/`max` only; widen the list one verified field at
  a time (candidates to evaluate later, each needing a check that `_initialize` is not the sole
  applier: `axes[id].visible`, `axes[id].colorscale`).
- **`getConfig()` round-trip fidelity.** The host must feed back a config whose non-carved fields are
  stable; float-formatting / key-order differences in the *stripped* portion will (correctly) force a
  rebuild. This is a host-app memoization concern, documented, not fixed here.
- **Small surface, low blast radius.** The change is additive — a pre-filter in front of the existing
  rebuild. The fallback is always "rebuild", so the worst failure of a *too-conservative* classifier
  is the current behaviour (an unnecessary rebuild), never a stale plot.
