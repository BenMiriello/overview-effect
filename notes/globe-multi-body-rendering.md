# Globe Multi-Body Rendering: Architecture & Lessons

## react-globe.gl Camera Ownership

react-globe.gl is earth-centric. Its camera management, tile engine, and coordinate system all assume the camera orbits (0,0,0). You cannot make it orbit a different body by tweaking properties — you must fully bypass its camera management for any non-earth view.

### Two Independent Animation Loops

The library runs two separate RAF loops that control the camera:

1. **`_animationCycle`** — calls `controls.update()` + `renderer.render()`. Stopped by `globeEl.pauseAnimation()`.

2. **TWEEN IIFE** — `(function onFrame() { requestAnimationFrame(onFrame); TWEEN.update(); })()`. Runs forever, cannot be paused. Processes camera tweens created by `pointOfView()` setter, which directly writes `camera.position.x/y/z` bypassing OrbitControls.

Patching `controls.update` alone is insufficient because the TWEEN loop bypasses it entirely.

### The `isOrbiting` / `pointOfView()` Trap

When orbiting stops, the `isOrbiting` effect calls `globeEl.pointOfView({...}, duration)` to snap north. This creates a TWEEN that animates the camera back to earth-centric coordinates. Any non-earth camera work must guard against this with `if (inMoonViewRef.current) return;` in the isOrbiting effect.

### Full Camera Takeover Pattern

For any non-earth-centered view, use the same pattern as close-mode:

1. **Pause the library**: `pauseAnimation()` + patch `controls.update = () => false` + snap `pointOfView` to current position (duration=0) to kill active tweens
2. **Own the rendering**: call `renderer.render(scene, camera)` yourself each frame
3. **Own the input**: attach mouse/touch/wheel handlers for orbit/zoom
4. **Store orbit state in a ref**: spherical coords (theta, phi, distance) relative to the target body
5. **Track body movement**: each frame, recompute camera position from the body's current world position + spherical offset
6. **Clean restore**: on exit, `resumeAnimation()` + restore `controls.update` + reset `controls.target` to (0,0,0)

## Three.js Render Order for Mixed depthTest Objects

### The Problem

Night tiles (GIBS city lights) require `depthTest: false` and `depthWrite: false` to avoid z-fighting with the day tiles on earth's surface (they're at scale 1.001, nearly coplanar). But `depthTest: false` means they render on top of everything, including closer objects like the moon.

### Why renderOrder Alone Doesn't Work

Three.js renders in this order, ignoring renderOrder across groups:

1. All opaque objects (sorted by renderOrder, then front-to-back)
2. All transmissive objects
3. All transparent objects (sorted by renderOrder, then back-to-front)

An opaque object with `renderOrder: 1` still renders BEFORE a transparent object with `renderOrder: 0`. So an opaque moon always renders before transparent night tiles, and the night tiles (depthTest: false) overwrite it.

### The Fix: Transparent + renderOrder

Mark the moon material as `transparent: true` (even though it outputs alpha=1.0) with `depthWrite: true`. This puts it in the transparent render pass. Then `renderOrder: 1` correctly places it after the night tiles (renderOrder 0 default). The moon draws last in those pixels and overwrites the night tile fragments.

```typescript
const material = new THREE.ShaderMaterial({
  // ...shaders and uniforms...
  transparent: true,  // enters transparent render pass
  depthWrite: true,   // still writes depth for correct occlusion
});
mesh.renderOrder = 1; // renders after night tiles in transparent pass
```

This generalizes to any body that needs to render in front of depthTest-disabled layers.

## Drag/Orbit Controls for Non-Earth Bodies

### Spherical Coordinate Convention

Use THREE.Spherical convention:
- theta: azimuthal angle in x-z plane from +z
- phi: polar angle from +y (0 = north pole, PI = south pole)
- Clamp phi to [0.1, PI-0.1] to avoid gimbal lock at poles

### Drag Direction for "Grab the Surface"

For surface-grab feel (drag moves the surface, not the camera):
- Horizontal drag (dx > 0, drag right): theta decreases (camera orbits left, surface appears to move right)
- Vertical drag (dy > 0, drag down): phi decreases (camera orbits up, surface appears to move down)

### Drag Inertia

Track velocity during drag using exponential smoothing, release on mouseup, decay each frame:

```typescript
// During drag:
smoothDTheta = smoothDTheta * 0.7 + (dTheta / dt) * 0.3;

// After release, each frame:
moonOrbitState.theta += velocity.dTheta * dt;
const decay = Math.exp(-3 * dt);
velocity.dTheta *= decay;
```
