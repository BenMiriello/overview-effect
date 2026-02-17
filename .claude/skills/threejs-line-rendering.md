# Three.js Line Rendering Gotchas

Non-obvious issues discovered while implementing lightning rendering.

## LineMaterial vertexColors

**The Bug**: Animation updates vertex colors but nothing changes visually.

**Root Cause**: `LineMaterial` ignores vertex colors by default. Must set:

```javascript
const mat = new LineMaterial({
  color: 0xffffff,
  linewidth: 4,
  vertexColors: true,  // REQUIRED for vertex color animation
  // ...
});
```

Without `vertexColors: true`, the material uses its fixed `color` property for all vertices, ignoring the per-vertex colors set via geometry.

**Symptoms**:
- Animation logic runs correctly (logs show progress)
- But visual appearance never changes
- All segments visible at full brightness immediately

## LineMaterial Resolution

**The Bug**: Lines render with wrong width or not at all.

**Root Cause**: `LineMaterial.resolution` must be set to canvas size:

```javascript
material.resolution.set(window.innerWidth, window.innerHeight);
```

**Must update on resize**:
```javascript
useEffect(() => {
  material.resolution.set(width, height);
}, [width, height]);
```

## LineSegmentsGeometry Color Updates

After modifying colors array, must flag attributes for update:

```javascript
geometry.setColors(colors);
geometry.getAttribute('instanceColorStart').needsUpdate = true;
geometry.getAttribute('instanceColorEnd').needsUpdate = true;
```

Both `instanceColorStart` AND `instanceColorEnd` must be flagged.

## LineSegments2 vs Line2

- `LineSegments2`: For disconnected line segments (pairs of points)
- `Line2`: For connected polylines (continuous path)

Lightning segments are disconnected (each segment is independent), so use `LineSegments2`.

## Render Order for Glow

When using additive blending for glow effect:

```javascript
glowLine.renderOrder = 999;      // Render glow first
mainLine.renderOrder = 1000;     // Main line on top
```

Lower renderOrder = rendered first (behind).

## depthWrite for Transparency

For transparent/glowing lines:

```javascript
const mat = new LineMaterial({
  transparent: true,
  depthWrite: false,  // Prevents z-fighting with overlapping transparent lines
  blending: THREE.AdditiveBlending,  // For glow effect
});
```

## Performance: Material Pooling

Don't create new materials per segment. Create materials by depth tier:

```javascript
const materials = [
  new LineMaterial({ linewidth: 4, ... }),  // depth 0
  new LineMaterial({ linewidth: 3, ... }),  // depth 1
  new LineMaterial({ linewidth: 2, ... }),  // depth 2
  new LineMaterial({ linewidth: 1.5, ... }), // depth 3+
];
```

Share materials across all segments of same depth.
