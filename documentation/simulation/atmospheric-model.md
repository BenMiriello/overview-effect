# Atmospheric Model

The atmospheric model captures the spatial distribution of charge that influences lightning path selection. It consists of ceiling charge (cloud base), ground charge (induced), and their coupling.

---

## Overview

Real lightning doesn't travel through uniform air. Charge concentrates in pockets driven by convection, and the ground beneath develops induced charge. Our atmospheric model captures these effects.

---

## Model Components

### Ceiling Charge (Cloud Base)

Represents the main negative charge region at cloud base.

**Physical basis:**
- Ice-graupel collisions create charge separation
- Negative charge concentrates in pockets at -10 to -25 deg C isotherm
- Typically 5-8 km altitude

**Model:**
- 2D Voronoi field at y = ceilingY (0.5 in normalized units)
- 3-8 cells with random positions
- Intensity range: 0.5 to 1.0
- Falloff radius: 0.08 to 0.24 units (500m to 1.5km)

### Ground Charge (Induced)

Represents positive charge induced on ground by electrostatic induction.

**Physical basis:**
- Negative cloud charge repels ground electrons
- Ground surface becomes positively charged
- Distribution mirrors cloud charge (approximately)

**Model:**
- 2D Voronoi field at y = groundY (-0.5 in normalized units)
- Cells correlated with ceiling cells + jitter
- Plus 1-2 independent cells for local features

### Coupling: Smeared Mirror

Ground charge is derived FROM ceiling charge, not independent:

```typescript
function generateGroundCharge(ceilingCharge, rng, groundY, config) {
  const cells = [];

  // Induced cells (correlated with ceiling)
  for (const ceilingCell of ceilingCharge.cells) {
    const jitterX = (rng.next() - 0.5) * 0.1;  // +/- 0.05 units
    const jitterZ = (rng.next() - 0.5) * 0.1;

    cells.push({
      center: {
        x: ceilingCell.center.x + jitterX,
        y: groundY,
        z: ceilingCell.center.z + jitterZ,
      },
      intensity: ceilingCell.intensity * (0.7 + rng.next() * 0.3),
      falloffRadius: ceilingCell.falloffRadius * (0.8 + rng.next() * 0.4),
    });
  }

  // Independent cells (local conductivity features)
  const extraCells = 1 + Math.floor(rng.next() * 2);
  for (let i = 0; i < extraCells; i++) {
    // Random position within bounds
    // Random intensity and radius from ground config ranges
  }

  return new VoronoiField(cells, { is2D: true, fixedY: groundY });
}
```

**Why this approach:**
- Preserves physical causality (ground charge is induced BY ceiling charge)
- Jitter creates path variety (not perfectly straight cloud-to-ground)
- Independent cells represent metal structures, water bodies, wet soil
- Leaders flow from ceiling peaks toward corresponding ground peaks

---

## Starting Point Selection

Leaders originate from ceiling charge peaks, not arbitrary positions.

```typescript
function deriveStartingPoints(ceilingCharge, minThreshold = 0.3) {
  const maxima = ceilingCharge.getLocalMaxima();
  return maxima.filter(pos =>
    ceilingCharge.getValue(pos) >= minThreshold
  );
}
```

This creates **multi-leader competition**:
- Number of leaders emerges from charge distribution
- Not a configurable parameter
- Authentic physics: lightning starts where charge is concentrated

---

## Ground Charge Influence

Ground charge only affects paths when close to ground:

```typescript
const GROUND_CHARGE_RANGE = 0.08;  // ~500m
const GROUND_CHARGE_WEIGHT = 0.5;

if (groundDist > 0 && groundDist < GROUND_CHARGE_RANGE) {
  const proximityFactor = 1 - groundDist / GROUND_CHARGE_RANGE;
  const chargeValue = groundCharge.getValue({ x: point.x, y: groundY, z: point.z });
  field += chargeValue * proximityFactor * GROUND_CHARGE_WEIGHT;
}
```

**Why distance-modulated:**
- At high altitude, ground charge is too far to matter
- Influence increases as leader descends
- Final termination is attracted toward charge peaks
- Creates natural "seeking" behavior near ground

---

## Voronoi Field System

Each charge layer uses the same underlying field system.

### Cell Structure

```typescript
interface VoronoiCell {
  center: Vec3;          // Position of peak
  intensity: number;     // 0-1, value at center
  falloffRadius: number; // Distance to zero
}
```

### Sinusoidal Falloff

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

**Why sinusoidal (not linear):**
- Smooth S-curve transition
- No discontinuities at boundaries
- Matches natural diffusion/convection patterns
- Looks natural when visualized

---

## Configuration

```typescript
interface AtmosphericConfig {
  // Ceiling charge
  ceilingChargeCellCount: number;       // 3-8 typical
  ceilingChargeIntensityRange: [number, number];
  ceilingChargeRadiusRange: [number, number];

  // Ground charge
  groundChargeCellCount: number;        // (derived from ceiling)
  groundChargeIntensityRange: [number, number];
  groundChargeRadiusRange: [number, number];

  // Spatial bounds
  boundsRadius: number;  // 0.4 default
}
```

**Default values:**
```typescript
{
  ceilingChargeCellCount: 5,
  ceilingChargeIntensityRange: [0.5, 1.0],
  ceilingChargeRadiusRange: [0.08, 0.24],  // 500m to 1.5km

  groundChargeCellCount: 4,  // (actually derived from ceiling + extras)
  groundChargeIntensityRange: [0.3, 0.8],
  groundChargeRadiusRange: [0.08, 0.24],

  boundsRadius: 0.4,
}
```

---

## Integration with Simulation

The atmospheric model is created once per simulation:

```typescript
function simulateBolt(input) {
  const rng = createSeededRNG(seed);
  const atmoRng = rng.fork();

  // Create atmospheric model
  const atmosphere = createAtmosphericModel(atmoRng, ceilingY, groundY);

  // Pass to field context
  const fieldCtx = createFieldContext(groundY, fieldConfig, useSpatialGrid, atmosphere);

  // Spawn leaders from charge peaks
  const initialHeads = createInitialHeads(atmosphere, target, rng);

  // ... simulation loop uses fieldCtx.atmosphere for ground charge influence
}
```

---

## Visualization (Showcase Mode Only)

```typescript
class ChargeFieldRenderer {
  setChargeField(atmosphere, worldStart, worldEnd) {
    // Create ceiling plane with shader
    // Create ground plane with shader
    // Shader samples Voronoi cells for intensity
  }

  setVisible(visible) { /* toggle */ }
  dispose() { /* cleanup */ }
}
```

**Shader approach:**
- Pass cell centers, intensities, radii as uniforms (max 16 cells)
- Fragment shader samples field at each point
- Intensity maps to opacity
- Ceiling: blue-white, Ground: warm orange

---

## Future Extensions (Not Implemented)

### 3D Atmospheric Charge
Volumetric charge distribution affecting path mid-journey.

### Moisture Layer
Humidity variation lowering breakdown threshold.

### Pre-ionization Seeds
Cosmic ray tracks creating local attraction points.

### Upward Leaders
Ground-initiated leaders meeting descending leaders.

---

## References

- Rakov & Uman (2003) - Cloud electrification, charge structure
- Williams (1989) - Tripole model of thunderstorms
- Image charge principle in electrostatics
