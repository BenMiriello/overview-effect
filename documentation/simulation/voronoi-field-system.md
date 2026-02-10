# Voronoi Field System

The Voronoi field system provides smooth scalar fields for atmospheric properties. It's the underlying data structure for charge distribution, and could be extended for moisture and other properties.

---

## Concept

A Voronoi-based scalar field consists of multiple **cells**, each contributing to the total field value at any point. Cells blend smoothly where they overlap.

Traditional Voronoi diagrams have hard boundaries between cells. Our system uses **smooth falloff** for natural-looking fields.

---

## Cell Structure

```typescript
interface VoronoiCell {
  center: Vec3;           // Peak location
  intensity: number;      // Value at center (0-1 typical)
  falloffRadius: number;  // Distance where value reaches zero
}
```

Each cell represents a "pocket" of the property (charge, moisture, etc.).

---

## Sinusoidal Falloff

### The Formula

```
value(point) = intensity * (cos(distance/radius * PI) + 1) / 2
```

Where:
- `distance` = distance from point to cell center
- `radius` = cell's falloffRadius

### Value at Key Points

| Distance | Normalized (t = dist/radius) | cos(t*PI) | (cos+1)/2 | Value |
|----------|------------------------------|-----------|-----------|-------|
| 0 | 0 | 1 | 1.0 | intensity * 1.0 |
| radius/4 | 0.25 | 0.707 | 0.854 | intensity * 0.854 |
| radius/2 | 0.5 | 0 | 0.5 | intensity * 0.5 |
| 3*radius/4 | 0.75 | -0.707 | 0.146 | intensity * 0.146 |
| radius | 1.0 | -1 | 0 | 0 |
| > radius | > 1 | N/A | 0 | 0 |

### Why Sinusoidal?

Compared to linear falloff `(1 - distance/radius)`:
- **No discontinuity** at the edge
- **S-curve shape** matches natural diffusion patterns
- **Smooth derivative** everywhere
- **Visual quality** when rendered as gradient

---

## Multiple Cell Blending

When multiple cells overlap, values are **additive**:

```typescript
getValue(point: Vec3): number {
  let total = 0;
  for (const cell of this.cells) {
    const dist = this.distance2D(point, cell.center);
    if (dist < cell.falloffRadius) {
      const t = dist / cell.falloffRadius;
      const falloff = (Math.cos(t * Math.PI) + 1) * 0.5;
      total += cell.intensity * falloff;
    }
  }
  return total;
}
```

Total values can exceed 1.0 where cells overlap strongly.

---

## 2D vs 3D Modes

### 2D Mode (Charge Planes)

For ceiling and ground charge, the field is constrained to a plane:

```typescript
constructor(cells, { is2D: true, fixedY: ceilingY })
```

Distance calculation ignores Y:
```typescript
private distance2D(a: Vec3, b: Vec3): number {
  const dx = a.x - b.x;
  const dz = a.z - b.z;
  return Math.sqrt(dx * dx + dz * dz);
}
```

### 3D Mode (Volumetric)

For atmospheric charge or moisture (future):

```typescript
constructor(cells, { is2D: false })
```

Full 3D distance:
```typescript
private distance3D(a: Vec3, b: Vec3): number {
  const dx = a.x - b.x;
  const dy = a.y - b.y;
  const dz = a.z - b.z;
  return Math.sqrt(dx * dx + dy * dy + dz * dz);
}
```

---

## Gradient Computation

For path selection, we sometimes need the gradient (direction of increasing value):

```typescript
getGradient(point: Vec3): Vec3 {
  const eps = 0.001;

  const dx = this.getValue({ x: point.x + eps, ...}) -
             this.getValue({ x: point.x - eps, ...});
  const dy = this.is2D ? 0 :
             this.getValue({ y: point.y + eps, ...}) -
             this.getValue({ y: point.y - eps, ...});
  const dz = this.getValue({ z: point.z + eps, ...}) -
             this.getValue({ z: point.z - eps, ...});

  const len = Math.sqrt(dx*dx + dy*dy + dz*dz);
  if (len < 1e-10) return { x: 0, y: 0, z: 0 };

  return { x: dx/len, y: dy/len, z: dz/len };
}
```

Returns a unit vector pointing toward higher field values.

---

## Local Maxima Detection

For starting point selection, we find field peaks:

```typescript
getLocalMaxima(): Vec3[] {
  // Simple implementation: cell centers sorted by intensity
  return [...this.cells]
    .sort((a, b) => b.intensity - a.intensity)
    .map(cell => ({ ...cell.center }));
}
```

A more sophisticated implementation could search for true local maxima (including overlap regions), but cell centers are sufficient for charge-based leader spawning.

---

## Grid Sampling (Debug/Visualization)

```typescript
sampleGrid(bounds, resolution): { position: Vec3, value: number }[] {
  // Returns array of (position, value) pairs
  // Useful for debugging or texture generation
}
```

---

## Scale Reference

| Property | Min Radius | Max Radius | Real Size |
|----------|------------|------------|-----------|
| Charge pocket | 0.08 | 0.24 | 500m - 1.5km |
| Moisture region | 0.08 | 0.32 | 500m - 2km |
| Ionization seed | 0.002 | 0.008 | 12m - 50m |

---

## Shader Implementation

For visualization, the same falloff is implemented in GLSL:

```glsl
uniform vec3 cellCenters[16];
uniform float cellIntensities[16];
uniform float cellRadii[16];
uniform int cellCount;

float getChargeValue(vec2 pos) {
  float value = 0.0;
  for (int i = 0; i < 16; i++) {
    if (i >= cellCount) break;
    float dist = distance(pos, cellCenters[i].xz);
    if (dist < cellRadii[i]) {
      float t = dist / cellRadii[i];
      float falloff = (cos(t * 3.14159) + 1.0) * 0.5;
      value += cellIntensities[i] * falloff;
    }
  }
  return clamp(value, 0.0, 1.5);
}
```

Note: Loop bounds must be compile-time constant in GLSL ES 2.0.

---

## Performance Considerations

### Cell Count
- 5-8 cells for charge distributions: trivial
- 20-100 cells for ionization seeds: still fast
- Maximum practical: ~16 for shader (uniform array limit)

### Query Frequency
- Per-candidate field evaluation: ~16 per step
- ~200 steps = ~3200 queries
- Each query loops through all cells
- Total: ~16000 cell evaluations (trivial)

### Optimization (If Needed)
- Spatial grid for cell lookup
- GPU texture sampling instead of CPU computation
- Not currently necessary

---

## API Summary

```typescript
class VoronoiField {
  readonly cells: VoronoiCell[];

  constructor(cells: VoronoiCell[], options?: { is2D?: boolean, fixedY?: number });

  getValue(point: Vec3): number;
  getGradient(point: Vec3): Vec3;
  getLocalMaxima(): Vec3[];
  sampleGrid(bounds, resolution): Sample[];
}
```
