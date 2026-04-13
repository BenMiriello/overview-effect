# Postmortem: 3D Fly-to Going to Wrong Location (4-Day Bug)

## Summary

Clicking "Go to Hotspot" in 3D (close) mode navigated the camera to the wrong geographic location — typically ~90° east of the intended target. 2D mode worked correctly. The bug took ~4 days to fix across dozens of iterations.

---

## Root Cause (Two Compounding Issues)

### Issue 1: `enterCloseMode` called mid-animation with a stale camera position

The camera system has two modes:
- **Far mode**: the library (three-globe + OrbitControls) controls the camera
- **Close mode**: we control the camera directly via `applyCameraState` / `closeModeState`

`enterCloseMode` is triggered by `checkThreshold`, which listens to OrbitControls `'change'` events. When a 2D fly-to animated via `pointOfView({}, 0)` each RAF frame, OrbitControls fired `'change'` events throughout. As the altitude passed through the 1.0 threshold, `checkThreshold` called `enterCloseMode` mid-animation.

At that moment, `camera.position` was at an intermediate animation frame — not the final target. `enterCloseMode` captured this intermediate position as the close-mode origin. Subsequent 3D fly-tos used this stale starting position.

**Fix**: add `if (flyToActiveRef.current) return` to `checkThreshold`, and keep `flyToActiveRef = true` throughout the entire 2D fly-to animation (not just at start).

### Issue 2: Coordinate frame mismatch between three-globe and our camera math

Three-globe internally applies `globeObj.rotation.y = -Math.PI / 2` to the globe mesh. Its `polar2Cartesian` function uses `theta = (90 - lng) * π/180`, while our `latLngToCartesian` uses `theta = (lng + 180) * π/180`. These produce different world-space positions for the same geographic `(lat, lng)`.

The result:

```
our_frame_lng = library_frame_lng - 90°
```

- `pointOfView()` returns coordinates in the **library frame**
- `cartesianToLatLng(camera.position)` returns coordinates in **our frame**
- `latLngToCartesian(lat, lng)` positions the camera in **our frame**

These are internally consistent: `latLngToCartesian` and `cartesianToLatLng` are exact inverses, and the close-mode rendering loop uses them consistently. The visual result is always correct because both the camera position and the globe surface points share the same coordinate frame.

The problem appeared in the **fly-to calculation**:

```ts
// flyTo.lng comes from the server (real geography = library frame)
// startLng comes from closeModeState (our frame = library - 90°)
const lngDelta = flyTo.lng - startLng;  // off by 90°
```

The fly-to target (`flyTo.lng`) is real geographic data in the library frame. The starting position (`closeModeState.targetLng`) is in our frame. The 90° mismatch caused the fly-to to land at the wrong destination.

**Fix**: convert `flyTo.lng` to our frame before computing the delta:

```ts
const flyToLng_ours = ((flyTo.lng - 90) + 540) % 360 - 180;
const lngDelta = (((flyToLng_ours - startLng) % 360) + 540) % 360 - 180;
```

---

## Why It Was So Hard to Find

### 1. The errors masked each other in normal operation

`cartesianToLatLng` and `latLngToCartesian` are inverses. The -90° error introduced on read was exactly canceled by the +90° implicit in the write. Close mode *looked correct visually* in all cases except the fly-to, because rendering only cares about the final camera world position, not the intermediate coordinate values.

### 2. The coordinate mismatch was invisible in logs

Log lines like `[enterCloseMode] camera=(22.09, -128.10)` showed values 90° off from the expected position. But since close mode *rendered correctly*, the logs looked like a coordinate system issue — not a timing/stale-state issue. This sent investigation down the wrong path repeatedly.

### 3. The two bugs compounded in unpredictable ways

Bug 1 (stale position) produced a wrong `startLng`. Bug 2 (frame mismatch) then shifted the destination by another 90°. The combined error varied depending on where the camera happened to be when `enterCloseMode` fired mid-animation, making the wrong destination inconsistent and hard to reproduce precisely.

### 4. Fixing one bug revealed the other

When Bug 1 was fixed (flyToActiveRef guard), `enterCloseMode` started capturing the correct position — but the fly-to still went to the wrong place because Bug 2 was now exposed cleanly. Before the fix, the stale position and the frame offset produced the *same wrong destination* coincidentally in many test cases, making it look like a single issue.

### 5. Wrong diagnostic direction: changing coordinate functions

Multiple attempts tried to fix the bug by changing `enterCloseMode` to use `pointOfView()` instead of `cartesianToLatLng`. This felt correct (use the library's own function), but it broke close-mode entry by introducing a 90° snap: `pointOfView()` is library frame, `latLngToCartesian` is our frame, so the stored coordinates and the camera positioning function were now mismatched.

---

## Key Insight

**When a system has two coordinate frames, the danger is not using the wrong frame — it's mixing them silently.** Here, the close-mode system was internally consistent in its own frame. The fly-to target was in a different frame. The fix was minimal: convert at exactly one boundary (the fly-to target input), not refactor the entire coordinate system.

**Checklist for debugging coordinate frame bugs:**
1. Identify every place a coordinate value is *produced* and every place it is *consumed*
2. Label each with its frame explicitly (library frame vs. our frame, world vs. local, etc.)
3. Find the boundary crossing — the one place where a value crosses frames without conversion
4. Fix only that boundary; do not "fix" the consistent internal system

---

## Broader Extrapolations

### Cancellation and frame consistency are the same class of bug

The mid-animation `enterCloseMode` problem and the coordinate frame mismatch are both versions of the same issue: a value produced in context A is consumed in context B without conversion. Here "context" meant both temporal (mid-animation state vs. final state) and spatial (library frame vs. our frame). Any time a system produces values and another consumes them, the bug lives at the boundary — not inside either system.

### Visual correctness is a false signal

We chased this for days partly because close mode *looked right*. When two errors cancel, the system appears correct until a third path is introduced that only goes through one of them. The fly-to was that third path. When something looks right but behaves wrong in a specific scenario, look for paired errors that cancel — not a single broken thing.

### The moon coordinate system has the same latent risk

The moon camera (`moonPolar2Cartesian`, `moonCartesian2Polar`, `applyCameraStateMoon`) is a separate coordinate frame. If a server-pushed fly-to for the moon is ever added (e.g., navigate to a crater), the same frame-mismatch bug would appear. The fix pattern is already documented — convert incoming coordinates at the boundary.

### The close-mode/far-mode transition boundary is fragile

The instability wasn't the coordinate math — it was the *transition*. `enterCloseMode` and `exitCloseMode` are called from many paths (threshold crossing, toggle, fly-to, wheel scroll, touch), and each path has different camera state when it fires. Any future feature that adds another entry/exit path needs to ask: what frame is the camera in when this fires, and who holds `flyToActiveRef`?

### Diagnostic logs that are frame-aware cut days off debugging

The logs showed `camera=(22.09, -128.10)` which looked like a wrong value. The question that should have been asked earlier: *wrong relative to what frame?* If the logs had printed both `cartesianToLatLng` and `pointOfView()` side by side from the start, the 90° discrepancy would have been immediately visible. When debugging coordinate systems, always log both representations.

### The general rule

Any system in this codebase that crosses a coordinate or reference-frame boundary — shader UV space vs. world space, tile coordinates vs. geographic, moon local vs. scene world — carries this same latent risk. The fix is always the same: find the boundary, convert once, and leave the consistent internal systems alone.

---

## What We Changed

| File | Change |
|------|--------|
| `GlobeComponent.tsx` | `checkThreshold`: return early if `flyToActiveRef.current` |
| `GlobeComponent.tsx` | 2D fly-to animation: keep `flyToActiveRef = true` throughout, clear at t=1 and on cancel |
| `GlobeComponent.tsx` | 3D fly-to: convert `flyTo.lng` from library frame to our frame (`lng - 90°`) before computing `lngDelta` |
| `GlobeComponent.tsx` | `animTick` (exit animation): abort if `flyToActiveRef` becomes true mid-animation |
| `GlobeComponent.tsx` | `onCloseWheel` / `onCloseTouchStart`: guard against interrupting an active fly-to |
| `GlobeComponent.tsx` | `flyToCompletedAtRef`: 600ms cooldown in `onCloseWheel` after fly-to ends (trackpad inertia protection) |
