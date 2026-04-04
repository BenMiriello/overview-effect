# Globe Camera Perspective: 2D/3D Tilt and Orbit Mode

How the globe view handles tilt, perspective, and orbit. This document covers the camera control system, the tilt mechanism, orbit behavior, UI controls, and open design questions.

*Status: work in progress. Core implementation is in place and working, but tilt parameters need visual tuning and several design questions remain open.*

---

## Motivation

The default globe view always points the camera at the Earth's center regardless of zoom level. At high altitude this reads as a clean overhead map. At low altitude (zoomed into a storm cell), it still reads flat — you see the ground from above, not the horizon. Lightning strikes appear as vertical sticks rather than dramatic branching arcs.

The goal is to give users a perspective that changes as they zoom in: staying flat at global scale, tilting progressively as they approach the surface, so strikes appear the way they do visually — angled against the sky with the horizon visible.

A secondary goal: orbit mode. When zoomed in close, orbit around the surface point at center of view (like circling a storm). When zoomed out, orbit the planet at ISS speed.

---

## How Globe.gl Camera Controls Work

The globe uses **react-globe.gl** (v2.22.0), which wraps **globe.gl**, which wraps **three-globe**, which uses **THREE.js** (v0.176.0) with standard `OrbitControls`.

```
react-globe.gl
  └── globe.gl
        └── three-globe (THREE.Group)
              └── OrbitControls (from three/examples/jsm/controls/OrbitControls.js)
```

`globeEl.current.controls()` returns the live `OrbitControls` instance. All camera control flows through it.

**Key OrbitControls properties used:**

| Property | Effect |
|---|---|
| `controls.target` | The point the camera orbits around. Default: `(0,0,0)` = globe center |
| `controls.autoRotate` | Enables continuous rotation |
| `controls.autoRotateSpeed` | Rotation speed in degrees/frame |
| `controls.minDistance` | Minimum camera distance from target (not from globe surface) |
| `controls.maxDistance` | Maximum camera distance |

**`pointOfView({ lat, lng, altitude }, transitionMs)`** — globe.gl's API for positioning the camera. Always places the camera in canonical north-at-top orientation looking at the globe center. `altitude` is in units of globe radii (globe radius = 100 scene units). Does not accept pitch or heading.

---

## The Tilt Mechanism

The standard camera always points toward `controls.target = (0,0,0)`. If we shift the target to a point on or near the globe surface beneath the camera, the camera is now pointing at the horizon instead of the core. This is the mechanism used by Google Earth and Cesium — no quaternion math required.

### Coordinate conversion

Converting lat/lng to a world-space surface point:

```
phi   = (90 - lat) × π/180
theta = (lng + 180) × π/180

x = -(R × sin(phi) × cos(theta))
y =  R × cos(phi)
z =  R × sin(phi) × sin(theta)
```

This is the inverse of globe.gl's internal `getCoords()` function (confirmed by reading the three-globe source).

### Target interpolation

`controls.target` is interpolated between globe center `(0,0,0)` and the surface point based on zoom level:

```
t = clamp((TILT_THRESHOLD_ENTER - altitude) / (TILT_THRESHOLD_ENTER - MIN_ALTITUDE), 0, 1)
tiltFraction = TILT_FRACTION_CLOSE + (TILT_FRACTION_MAX - TILT_FRACTION_CLOSE) × t
controls.target = lerp(origin, surfacePoint, tiltFraction)
```

This runs in a listener on `controls.change` — fires on every camera movement. **Important:** calling `controls.update()` inside this handler causes a synchronous infinite event loop. OrbitControls picks up the modified `controls.target` on its next internal frame without needing an explicit `update()` call.

### Current tilt parameters

```typescript
const TILT_THRESHOLD_ENTER = 1.0;   // altitude at which tilt begins
const TILT_THRESHOLD_EXIT  = 1.15;  // altitude at which tilt disengages (hysteresis)
const MIN_ALTITUDE         = 0.15;  // max practical zoom-in
const TILT_FRACTION_CLOSE  = 0.45;  // target shift at threshold (~30° from top-down)
const TILT_FRACTION_MAX    = 0.80;  // target shift at max zoom (~60° from top-down)
```

These are **not visually validated yet**. The 0.45 → 0.80 range was chosen by calculation, not by looking at the result across many lat/lng combinations. **Tuning required.**

---

## Close Mode vs. Far Mode

A single altitude threshold divides behavior into two modes:

| | Far mode (altitude ≥ 1.15) | Close mode (altitude < 1.0) |
|---|---|---|
| Tilt | None — target stays at globe center | Active — target shifts toward surface |
| Orbit target | Entire planet | Surface point beneath camera |
| Auto-engage | On page load | When user zooms past threshold |
| 3D button | Hidden | Visible |

A **hysteresis band** (enter at 1.0, exit at 1.15) prevents rapid toggling when hovering near the boundary.

---

## Orbit Behavior

OrbitControls `autoRotate` always orbits around `controls.target`. This means the orbit target follows the tilt target automatically — no special-casing needed.

- **Far mode**: `controls.target = (0,0,0)`, so `autoRotate` orbits the planet
- **Close mode**: `controls.target = surface point`, so `autoRotate` orbits that location

**Orbit speeds:**

```typescript
const ORBIT_SPEED_PLANET  = 0.067;  // ISS-speed planet orbit (~92 min period)
const ORBIT_SPEED_SURFACE = 4.1;    // Surface-point orbit (~90 sec period)
```

The large jump in speed between modes is intentional — the orbit radius is vastly different. However, the surface speed value (4.1) is empirical and needs validation against actual 90-second timing. **Tuning required.**

### Auto-engage and user override

Orbit auto-engages when the altitude threshold is crossed inward. Any mousedown, wheel, or touchstart event stops orbit and turns off the orbit toggle.

### North-at-top on orbit stop

When orbit is turned off, the camera may be pointing in any direction (camera has drifted from north-up). `globe.gl`'s `pointOfView()` always positions the camera in canonical north-up orientation. Calling it with the current lat/lng/altitude and a transition duration of 1200ms eases the view back to north-at-top.

During this animation, the tilt logic is paused via a `tiltPausedRef` flag to prevent the `controls.change` listener from fighting the `pointOfView()` transition.

---

## UI Controls

Two circular buttons positioned in a horizontal row above the StatusBar (bottom-left corner):

```
[ ↺ orbit ]  [ 2D / 3D ]
─────────────────────────
 Connected | ... status ...
```

| Button | Always visible? | Active state |
|---|---|---|
| Orbit (RotateCcw icon) | Yes | Brightened border + white icon |
| 2D/3D text | Only in close mode | Same |

The 2D/3D button is hidden above the altitude threshold — it has no effect in far mode and showing it would be confusing.

**Styling:** Matches existing nav icons — 40px circle, `rgba(0,0,0,0.3)` background, `backdrop-filter: blur(4px)`, 1px border, Roboto font. Scale 1.2× on hover.

---

## Implementation Files

| File | Role |
|---|---|
| `client/src/components/GlobeComponent.tsx` | All camera/tilt/orbit logic. The `controls.change` listener, mode detection, prop-driven effects |
| `client/src/pages/GlobePage.tsx` | Owns `is3D`, `isOrbiting`, `isCloseMode` state. Passes to both GlobeComponent and GlobeControls |
| `client/src/components/GlobeControls.tsx` | Button UI. Conditionally renders 3D button based on `isCloseMode` |
| `client/src/components/GlobeControls.css` | Button styling |

### State flow

```
GlobePage
  ├── is3D, isOrbiting, isCloseMode  (useState)
  │
  ├── GlobeComponent (drives controls)
  │     ├── reads: is3D, isOrbiting
  │     └── writes: onIs3DChange, onIsOrbitingChange, onCloseModeChange
  │
  └── GlobeControls (drives buttons)
        └── reads: is3D, isOrbiting, isCloseMode
```

---

## Design Decisions

### Why `controls.target` shift instead of camera quaternion

Manipulating camera orientation directly (quaternion, `lookAt()`) fights OrbitControls on every frame — OrbitControls recomputes camera position from its internal spherical state and overwrites whatever you set. The target shift works *with* OrbitControls, not around it.

### Why not a slider UI

A slider would require always-visible UI, is harder to associate with a specific camera state, and offers false precision (users don't think in tilt degrees). The auto-tilt at threshold is more Google Earth-like and more intuitive.

### Why separate `controls.change` listener vs. rAF loop

The tilt needs to update on every camera movement, not every frame. Using `controls.change` means it fires exactly when needed. A per-frame loop in `GlobeLayerManager` would also work but is less precise and harder to clean up.

---

## Known Issues / Open Questions

### Tilt values not visually validated

The `TILT_FRACTION_CLOSE` and `TILT_FRACTION_MAX` values (0.45 and 0.80) were computed from approximate trig, not verified visually across real use cases. Need to zoom into several storms at different lat/lng values and verify the tilt looks natural.

### Polar regions untested

`getSurfacePoint()` uses standard spherical coordinate math. At latitudes above ~80°, the surface point may produce unexpected tilt directions as theta loses significance. Not tested.

### Orbit speed empirical

`ORBIT_SPEED_SURFACE = 4.1` was chosen to approximate a 90-second period but hasn't been timed against a clock. OrbitControls `autoRotateSpeed` units are degrees per second at 60fps in some versions; in others they're raw delta applied per frame. Verify by timing one full rotation.

### No smooth orbit speed transition between modes

When crossing the altitude threshold while orbiting, `autoRotateSpeed` jumps from 0.067 to 4.1 (or back). This is currently a hard cut. Smooth interpolation over a few frames would feel better.

### Manual 3D disable while zoomed in

If the user turns off 3D while close (via button), the `controls.change` listener won't re-enable it because `is3DRef.current` is false. But if they zoom out and back in, it will auto-enable again. This is acceptable behavior but should be verified: user expectation may be that "I turned it off, keep it off." A separate `userOverrode3D` flag could gate the auto-trigger.

### `pointOfView()` fighting tilt target

During the north-snap animation, `pointOfView()` drives the camera toward a north-up position while `controls.change` events also fire (because the camera is moving). The `tiltPausedRef` flag suppresses the tilt logic during this window, but if the animation duration mismatches the setTimeout guard (currently `NORTH_SNAP_DURATION + 100ms`), there may be a brief conflict. Not confirmed as a real issue yet.

---

## Research Sources

These resources informed the approach:

- **react-globe.gl source / type definitions**: `client/node_modules/globe.gl/dist/globe.gl.d.ts` — confirmed `controls()` returns `OrbitControls`, `pointOfView()` signature
- **OrbitControls source**: `client/node_modules/three/examples/jsm/controls/OrbitControls.js` — confirmed `controls.target`, `minPolarAngle/maxPolarAngle`, the `change` event
- **three-globe source**: `client/node_modules/three-globe/dist/three-globe.mjs` — confirmed `GLOBE_RADIUS = 100`, coordinate conversion formula
- **Google Earth**: Reference UX for how auto-tilt and 2D/3D toggle should feel. Their approach: tilt engages automatically as you zoom in; discrete 2D/3D button appears when zoomed in enough; orbit feature (Tour) is a separate control. We are following this pattern.
- **Cesium JS documentation**: Cesium also uses a target-shift approach for tilt. Their `camera.lookAtTransform` sets the orbit target to a surface point, which is the same mechanism we use.

---

## TODO

- [ ] Visually validate tilt fractions at 5+ real storm locations
- [ ] Time the surface orbit speed and correct `ORBIT_SPEED_SURFACE` if needed
- [ ] Smooth the autoRotateSpeed transition when crossing the altitude threshold
- [ ] Test at polar latitudes (>80°N/S)
- [ ] Decide policy on "user manually disabled 3D while close — should auto-re-enable suppress?"
- [ ] Verify north-snap doesn't stutter (check `tiltPausedRef` timing)

---

*Created: 2026-03-31*
