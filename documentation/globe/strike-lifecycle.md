# Globe Strike Lifecycle

How lightning strikes are created, animated, and cleaned up in the globe view. This document covers the layer system, effect lifecycle, Three.js resource management, and the performance investigation that informed our current implementation.

*Status: work in progress. We have a reasonable understanding of the lifecycle and resource disposal, but some performance questions remain open.*

---

## Overview

The globe view renders real-time strikes from Blitzortung as they arrive over WebSocket. Each strike goes through a lifecycle:

```
WebSocket strike arrives
        │
        ▼
LightningLayer.addData(strike)
        │
        ├──► createLightningBoltEffect()   [if showLightningBolt=true]
        │         └── LightningBoltEffect (GPU geometry, animation)
        │
        └──► createMarkerEffect()
                  └── PointMarkerEffect (persistent dot on globe)
```

Both effects are updated every animation frame, clean themselves up when done, and are bounded by configurable limits. The globe view is designed to run indefinitely without accumulation.

---

## System Components

### GlobeLayerManager

Central hub. Owns the RAF animation loop and all layer instances.

```
GlobeLayerManager
  ├── startAnimationLoop()     // RAF loop calling updateAllLayers()
  ├── layers: Map              // id → Layer instances
  ├── createLayer()
  ├── removeLayer()
  └── dispose()                // stops loop, clears all layers
```

The loop runs `updateAllLayers(Date.now())` every frame, which calls `layer.update(currentTime)` for each visible layer. If a layer is hidden, it is skipped entirely — importantly, this means effects in hidden layers do NOT get cleaned up until the layer is visible again.

### LightningLayer

The main lightning layer. Manages two collections of effects:

```
LightningLayer
  ├── lightningBoltEffects: Map<id, LightningBoltEffect>  // max 10
  ├── markerEffects: Map<id, PointMarkerEffect>           // max 256
  └── activeEffects: { id, timestamp }[]                 // tracks order for eviction
```

On every `update()`:
1. Iterates all bolt effects → terminates completed ones
2. Iterates all marker effects → terminates completed ones

On `addData(strike)`:
1. Creates bolt effect (if enabled)
2. Creates marker effect
3. Calls `enforceMaxDisplayedStrikes()` to evict oldest if over limit
4. Calls `ensureMaxActiveEffects()` to evict oldest bolt if over limit

### LightningBoltEffect

Wraps the GPU representation of one animated lightning bolt.

```
LightningBoltEffect
  ├── BoltRenderer           // THREE.Group with LineSegments2 objects
  ├── BoltAnimator           // timing state machine
  ├── CoordinateTransform    // normalized sim space ↔ world space
  └── LightningMaterials     // LineMaterial instances (per depth bucket)
```

**Lifecycle:**
1. Constructor: runs `simulateBolt()` synchronously, adds THREE.Group to scene
2. `update(t)`: advances animator, renders segment brightness
3. Complete: `AnimationPhase.COMPLETE` reached (~710ms for GLOBE detail)
4. `terminate()`: calls `BoltRenderer.dispose()`, nulls animator/transform/path data

**GLOBE animation phases and durations:**

| Phase | Duration |
|-------|----------|
| Leader stepping | 250ms |
| Connection pause | 20ms |
| Return stroke | 40ms |
| Stroke hold | 60ms |
| Fade | 300ms |
| Interstroke + second stroke | 440ms |
| **Total** | **~710ms** |

### PointMarkerEffect

A persistent dot shown at each strike location. Survives much longer than the bolt animation.

```
PointMarkerEffect
  ├── CircleGeometry or RingGeometry
  ├── MeshBasicMaterial (transparent, DoubleSide)
  └── Mesh (added to scene on initialize())
```

**Lifecycle timing:**
- `fadeInDuration`: 1500ms (waits for bolt animation to begin)
- `fadeStartAge`: 10000ms (full opacity for 10s)
- `maxAge`: 60000ms (complete fade-out and cleanup at 60s)

When `age > maxAge`, `markComplete()` is called. The update loop in `LightningLayer` sees `isComplete()=true` and calls `terminate()`, which removes the mesh from the scene and disposes geometry + material.

---

## Three.js Resource Management

### What gets created per bolt effect

- 1 `THREE.Group` added to scene
- N `LineSegmentsGeometry` (one per depth bucket, typically 5–15)
- N `LineSegments2` objects inside the group
- 1 glow group (`LineSegmentsGeometry` + `LineSegments2`)
- 1 `LightningMaterials` instance containing:
  - 1 `LineMaterial` base
  - 1 `LineMaterial` glow
  - M `LineMaterial` per depth bucket (created lazily, cached)

### Disposal path for bolt effects

```
LightningBoltEffect.terminate()
  └── BoltRenderer.dispose()
        ├── clear()
        │     ├── for each DepthGroup:
        │     │     ├── geometry.dispose()       // frees GPU buffer
        │     │     └── group.remove(line)       // removes from THREE.Group
        │     └── glowGroup.geometry.dispose()
        │
        ├── group.parent.remove(group)            // removes from scene
        └── materials.dispose()                   // disposes all LineMaterials
```

After `terminate()`, `this.animator` and `this.transform` are nulled to release the large simulation data structures (segment Maps/Sets) for GC.

### What gets created per marker effect

- 1 `CircleGeometry` (25-segment disc)
- 1 `MeshBasicMaterial`
- 1 `THREE.Mesh` added to scene

### Disposal path for marker effects

```
BaseEffect.terminate()
  ├── scene.remove(mesh)           // removes from scene
  └── dispose()
        ├── geometry.dispose()    // frees GPU buffer
        └── material.dispose()   // frees GPU material
```

---

## Bounds and Eviction

| Resource | Limit | Config key |
|----------|-------|------------|
| Active bolt animations | 10 | `layers.lightning.maxActiveAnimations` |
| Persistent markers | 256 | `layers.lightning.maxDisplayedStrikes` |

**Bolt eviction** (`ensureMaxActiveEffects`): When a new bolt arrives and 10 are already active, the oldest (by timestamp) is immediately terminated.

**Marker eviction** (`enforceMaxDisplayedStrikes`): When markers exceed 256, oldest (Map insertion order, which equals creation order) are evicted one by one.

---

## Performance Investigation (2026-04-01)

### Symptom

After a few minutes of running, the page became severely laggy and eventually unusable. Initial hypothesis was a Three.js memory/object leak — effects not being cleaned up.

### What we found

**The Three.js cleanup was structurally correct.** Every code path that terminates an effect properly removes objects from the scene graph and disposes GPU resources. Effects complete and get cleaned up on schedule. Object count does not grow unbounded.

**The actual causes of degradation were:**

**1. Console.log per strike (most impactful)**

`LightningLayer.addData()` had two `console.log` calls that fired on every incoming strike:
```js
console.log('LightningLayer: Adding strike', strike);
console.log('showLightningBolt config:', ...);
```

At 1–10 strikes/second, this creates thousands of logged objects per minute. Browser DevTools keeps all objects passed to `console.log` alive in memory indefinitely (as long as the console pane is open). This was the primary driver of memory growth. *Fixed: removed both calls.*

**2. O(n log n) sort on every strike**

`enforceMaxDisplayedStrikes()` sorted all marker entries on every single strike arrival:
```js
// old: sort all 256 markers on every strike
const markers = Array.from(this.markerEffects.entries())
  .sort((a, b) => aTime - bTime);
```

With 256 markers at steady state, this was ~2000 comparison operations per strike. At 5 strikes/second that's 10,000 comparisons/second just for eviction. *Fixed: JS `Map` preserves insertion order, so the oldest marker is always `entries().next().value` — O(1).*

**3. Large simulation data not released promptly**

`LightningBoltEffect.terminate()` disposed GPU resources via `BoltRenderer.dispose()`, but `this.animator` — which holds the full `BoltGeometry` and several large Maps of segment data — was not nulled. The `LightningBoltEffect` object itself was eligible for GC once removed from `lightningBoltEffects`, but any external reference (e.g., a temporary local variable during the update loop) would delay collection of the entire structure. *Fixed: null out `animator`, `transform`, `mainChannelPath`, `mainChannelIds` in `terminate()`.*

### What we still don't know

- Whether GPU memory stays truly stable over very long sessions (30+ minutes). We haven't measured `renderer.info.memory` over time.
- How `simulateBolt()` cost scales at very high strike rates (>10/second, e.g. direct cell observation). Each bolt computes synchronously on the main thread; at high rates with 10 active slots, the CPU budget is under pressure.
- Whether the hidden-layer update gap (effects not cleaned up while layer is hidden) is a real issue in practice. Currently the layer is never hidden.
- The full cost breakdown per frame. We have no profiler captures.

### What we measured / verified

- The hotspot navigation feature (globe auto-positions to recent strike centroid on load) was verified working via console output: `[GlobePage] hotspot: 1437 strikes at lat=31.54, lng=-73.76` and `[GlobeComponent] intro animation → lat=32.04, lng=-75.37 (hotspot)`.
- The race condition fix (animation now waits for hotspot fetch to settle before starting) was confirmed via `targetPositionReady` gating.

---

## Globe Camera Positioning on Load

Related to strike lifecycle: on load the globe animates to the geographic centroid of recent strikes.

**Backend** (`server/server.js`): `/api/hotspot` computes a time-weighted centroid of strikes in the last 5 minutes. Weight decays linearly with age (newest = weight 1.0, 5-minute-old = weight 0).

**Frontend** (`GlobePage.tsx`): fetches `/api/hotspot` on mount, sets `hotspotReady=true` when the fetch settles (success or error). Passes both `targetPosition` and `targetPositionReady` to `GlobeComponent`.

**Animation** (`GlobeComponent.tsx`): a dedicated `useEffect` with deps `[isGlobeReady, targetPosition, targetPositionReady]` starts the intro camera sweep only after both the globe is initialized and the hotspot fetch is settled. Falls back to `DEFAULT_TARGET = { lat: 20, lng: -55 }` if no hotspot data.

---

## Open Questions / TODO

- [ ] Profile actual GPU memory usage over a 30-minute session (add `renderer.info` monitoring)
- [ ] Measure main-thread time cost of `simulateBolt()` per call at realistic strike rates
- [ ] Evaluate whether `maxActiveAnimations: 10` is the right cap given CPU budget
- [ ] Decide if hidden-layer cleanup gap needs addressing (terminate effects when layer is hidden)
- [ ] Consider moving `simulateBolt()` off the main thread for globe-mode strikes (separate from the showcase worker, which has different constraints)

---

*Created: 2026-04-01*
