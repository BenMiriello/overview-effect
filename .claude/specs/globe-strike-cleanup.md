---
status: active
priority: critical
area: globe
---

# Globe View: Strike Animations Not Cleaning Up

## Problem

Strike animations on the globe accumulate over time and never fully clean up. Some go away but many linger. After a few minutes, the accumulated stale strikes cause severe performance degradation -- the page becomes laggy and eventually unusable.

## Root Cause Investigation — Completed

**Three.js disposal is structurally correct.** Full trace:
- `LightningLayer.update()` runs every frame via `GlobeLayerManager` animation loop
- Completed bolt effects (`AnimationPhase.COMPLETE`, ~710ms for GLOBE detail) → `effect.terminate()` → `BoltRenderer.dispose()` removes THREE.Group from scene, disposes all LineSegmentsGeometry and LineMaterial objects
- Completed markers (after `maxAge: 60000ms`) → `effect.terminate()` → `scene.remove(mesh)`, geometry + material disposed
- Bounds enforced: max 10 active bolt effects, max 256 markers

**Actual causes of performance degradation:**

1. **`console.log` per strike** — `addData()` logged the full strike object on every incoming strike (1–10/sec). DevTools keeps all logged objects alive indefinitely → unbounded memory growth in DevTools. **Fixed: removed.**

2. **O(n log n) sort per strike** — `enforceMaxDisplayedStrikes()` sorted all markers on every single strike arrival. With 256 markers at steady state, this was ~2000 sort comparisons per strike. **Fixed: replaced with O(1) FIFO using Map insertion order.**

3. **Large simulation data held post-terminate** — `LightningBoltEffect.terminate()` disposed GPU resources but left `this.animator` (containing large segment Maps/Sets from `simulateBolt`) alive until GC. **Fixed: null out animator, transform, and path data on terminate.**

## Changes Made

- `LightningLayer.ts`: removed 2 `console.log` calls from `addData()`
- `LightningLayer.ts`: `enforceMaxDisplayedStrikes()` now uses `Map.entries().next()` (O(1)) instead of full sort
- `LightningBoltEffect.ts`: `terminate()` nulls `this.animator`, `this.transform`, `this.mainChannelPath`, `this.mainChannelIds` after disposing GPU resources

## Acceptance Criteria

- [ ] Globe maintains stable FPS after 5+ minutes of continuous strikes
- [ ] Three.js object count does not grow unbounded (check renderer.info)
- [ ] No visual remnants of old strikes after their animation completes
- [ ] Memory usage stays stable over time
