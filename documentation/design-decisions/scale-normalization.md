# Scale Normalization

The simulation operates in normalized coordinate space rather than real-world meters. This document explains the scale system and conversion factors.

---

## The Problem

Real lightning involves:
- Cloud-to-ground distance: 5-8 km
- Step length: 50 m
- Charge pocket radius: 500-1500 m
- Breakdown field: megavolts per meter

Using real units would cause:
- Large floating-point numbers
- Numerical precision issues
- Harder-to-read configuration
- Scale-dependent algorithms

---

## The Solution: Normalized Space

### Coordinate System

| Position | Normalized Y | Real Altitude |
|----------|--------------|---------------|
| Cloud base (start) | 0.5 | ~6 km |
| Ground (end) | -0.5 | 0 km |
| Mid-air | 0.0 | ~3 km |

Total height: 1.0 unit = ~6.25 km

### Origin

The origin (0, 0, 0) is the midpoint between cloud and ground. This places the bolt centered in the coordinate system.

### Scale Factor

```
1 simulation unit = 6250 meters
1 meter = 0.00016 simulation units
```

Derived from:
- Real cloud base: ~5-8 km → use 6.25 km as reference
- Normalized height: 1.0 unit
- 6250 m / 1.0 unit = 6250 m/unit

---

## Common Conversions

| Real World | Simulation Units | Notes |
|------------|------------------|-------|
| 50 m (step) | 0.008 units | Stepped leader step length |
| 500 m | 0.08 units | Minimum charge pocket radius |
| 1.5 km | 0.24 units | Maximum charge pocket radius |
| 300 m | 0.05 units | Typical jitter for ground charge |
| 6.25 km | 1.0 units | Full cloud-to-ground distance |

---

## The Constants File

```typescript
// simulation/constants.ts

export const SCALE = {
  METERS_PER_UNIT: 6250,
  UNITS_PER_METER: 1 / 6250,

  STEP_LENGTH_METERS: 50,
  STEP_LENGTH_UNITS: 0.008,

  CHARGE_POCKET_RADIUS: {
    MIN: 0.08,   // ~500m
    MAX: 0.24,   // ~1.5km
  },

  MOISTURE_REGION_RADIUS: {
    MIN: 0.08,
    MAX: 0.32,
  },

  IONIZATION_SEED_RADIUS: {
    MIN: 0.002,  // ~12m
    MAX: 0.008,  // ~50m
  },
} as const;

export function metersToUnits(meters: number): number {
  return meters * SCALE.UNITS_PER_METER;
}

export function unitsToMeters(units: number): number {
  return units * SCALE.METERS_PER_UNIT;
}
```

---

## Why These Specific Values?

### Step Length: 0.008 units

Real stepped leaders have ~50m steps. With 6250 m/unit:
```
50 m / 6250 m/unit = 0.008 units
```

This gives ~125 steps for SHOWCASE (1.0 unit / 0.008 units/step).

For GLOBE, we use 0.02 units (~125m real) for fewer segments.

### Charge Pocket Radius: 0.08-0.24 units

Real charge pockets: 500m - 1.5km
```
500m / 6250 = 0.08 units
1500m / 6250 = 0.24 units
```

### Ground Charge Range: 0.08 units

Ground charge should only affect the last ~500m of descent:
```
500m / 6250 = 0.08 units
```

When leader is within 0.08 units of ground, ground charge starts influencing path.

---

## Coordinate Transform

The rendering layer transforms normalized → world coordinates:

```typescript
// In BoltRenderer
setGeometry(geometry, worldStart, worldEnd) {
  const worldLength = distance(worldStart, worldEnd);
  const worldDir = normalize(subtract(worldEnd, worldStart));

  this.worldTransform = {
    origin: midpoint(worldStart, worldEnd),
    scale: worldLength,       // Normalized 1.0 → worldLength
    rotation: quaternionFromDirection(worldDir)
  };
}

toWorldSpace(normalized: Vec3): Vec3 {
  const scaled = scale(normalized, this.worldTransform.scale);
  const rotated = applyQuaternion(scaled, this.worldTransform.rotation);
  return add(rotated, this.worldTransform.origin);
}
```

For Globe view: worldLength is the actual globe distance
For Showcase view: worldLength is the scene-appropriate size

---

## Benefits

### Numerical Stability

All calculations happen in [-0.5, 0.5] range. No megavolt field values causing overflow.

### Readable Configuration

```typescript
// Easy to read and understand
stepLength: 0.008,         // vs stepLength: 50 (meters? pixels?)
coneHalfAngle: Math.PI/6,  // 30 degrees, scale-independent
```

### Reusable Algorithms

Same algorithm works for:
- Globe view (km-scale real world)
- Showcase view (meter-scale scene)
- Any other context

Just change the world transform, not the simulation.

### Easy Debugging

Can log normalized positions and immediately understand "0.2 means 20% down from ceiling."

---

## Trade-offs

### Extra Mental Layer

Must think in normalized units and convert mentally to real-world.

Mitigation: Constants file has both values documented.

### Potential Precision Loss

Conversion functions introduce floating-point operations.

Mitigation: For our precision needs, insignificant.

---

## Historical Note

Early implementation mixed:
- Math.random() (0-1 range)
- Real-ish field values
- Arbitrary position coordinates

This caused confusion and bugs. Normalization standardized everything.
