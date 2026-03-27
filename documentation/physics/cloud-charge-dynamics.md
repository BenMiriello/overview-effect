# Cloud Charge Dynamics

This document describes the physics of charge distribution and dynamics within thunderstorm clouds, focusing on aspects relevant to lightning simulation.

---

## Cumulonimbus Structure

### Typical Dimensions

| Parameter | Value |
|-----------|-------|
| Base altitude | 200-4,000 m (varies by climate) |
| Top altitude | 10-12 km (mid-latitudes), up to 18-20 km (tropics) |
| Horizontal diameter | 5-30 km per cell, ~24 km average |
| Anvil extent | 100+ km downwind |

### Internal Structure

```
20km  |                         [Anvil / cirrus outflow - positive charge]
      |
12km  |              [Upper positive charge region: ~40-50 C]
      |           _______________
10km  |          |               |
      |          |  UPDRAFT CORE |
8km   |          |               |
      |          | [-] MAIN NEG  |
6km   |          | CHARGE REGION |
      |          |               |
4km   |          | [+] LOWER POS |
      |           _______________
2km   |
      |   [CLOUD BASE]
0km   |_________________________________________________
```

Key regions:
- **Updraft core**: 5-15 m/s typical, up to 40 m/s in supercells
- **Stratiform/anvil**: Ice crystals advected by upper-level winds
- **Precipitation shaft**: Falling graupel/hail from negative region

---

## The Tripole Model

The dominant model for thunderstorm charge structure:

### Upper Positive Region
- **Magnitude**: ~40-50 Coulombs
- **Altitude**: 8-12 km
- **Carrier**: Small ice crystals (lofted by updraft)
- **Size**: ~3-5 km radius

### Main Negative Region
- **Magnitude**: ~40-100 Coulombs (dominant driver of CG lightning)
- **Altitude**: 5-8 km (at -10 to -25 deg C isotherm)
- **Carrier**: Graupel particles
- **Size**: ~3-8 km radius

### Lower Positive Region
- **Magnitude**: ~3-10 Coulombs (much weaker)
- **Altitude**: 2-5 km
- **Carrier**: Large graupel/rain in melting layer
- **Size**: ~2-4 km radius

---

## Temperature and Charge

| Altitude | Temperature | Charge Significance |
|----------|-------------|---------------------|
| 12+ km | < -50 C | Ice only; anvil outflow |
| 8-12 km | -20 to -50 C | Upper positive (ice crystals) |
| 5-8 km | -10 to -25 C | **Main negative** (reversal zone) |
| 3-5 km | 0 to -15 C | Lower positive |
| 0-3 km | +5 to 0 C | Liquid water, warm base |

The **reversal temperature** (~-10 to -20 C) is critical: this is where ice crystal-graupel collisions switch charge transfer polarity.

---

## Charge Separation Mechanism

### Non-Inductive Ice-Graupel Process (Primary Mechanism)

1. Supercooled water droplets coexist with ice crystals and graupel at -10 to -25 C
2. Rising ice crystals (small, light) collide with falling graupel (large, dense)
3. During each rebounding collision, ~100,000 electrons transfer from ice to graupel
4. Collision rate: ~5 x 10^13 collisions per minute per km^3 of active updraft

### Charge Transfer Polarity

Above the reversal temperature (normal polarity case):
- Graupel charges **negative**
- Ice crystals charge **positive**

Result: Graupel (heavy, falls) carries negative charge down; ice (light, lofted) carries positive charge up.

### Role of Updraft

The updraft is the engine of charge separation:
- Strong updrafts (>5-10 m/s) required to loft ice crystals
- Updraft speed determines collision rate
- Charge buildup roughly proportional to updraft intensity squared

---

## Charge Pocket Geometry

Charge regions are NOT uniform spheres:

- **Horizontally elongated**: Wider than tall (ellipsoidal)
- **Concentrated near updraft**: Strongest density adjacent to active updraft
- **Gradient boundaries**: Density falls off steeply at edges
- **Co-located with precipitation**: Negative overlaps graupel; positive overlaps rain/melting

### Mathematical Model (Gaussian Ellipsoid)

```typescript
interface ChargeRegion {
  center: Vec3;      // Position
  magnitude: number; // Coulombs (negative or positive)
  sigmaH: number;    // Horizontal spread (km)
  sigmaV: number;    // Vertical spread (km)
}

// Example tripole in normalized coordinates
const upperPositive = { center: [0, 0.85, 0], magnitude: +40, sigmaH: 4.0, sigmaV: 1.5 };
const mainNegative  = { center: [0, 0.55, 0], magnitude: -80, sigmaH: 5.0, sigmaV: 2.0 };
const lowerPositive = { center: [0, 0.30, 0], magnitude: +8,  sigmaH: 2.5, sigmaV: 1.0 };
```

---

## Thunderstorm Types and Flash Rates

| Type | Flash Rate | Character |
|------|-----------|-----------|
| Single cell | 1-10/min | Brief, disorganized |
| Multicell | 10-60/min | Pulsing, multiple active cells |
| Squall line | Continuous | Spread along linear front |
| Supercell | 50-200+/min | Rotating updraft, highest rates |

---

## Charge Dynamics During Lightning

### Flash Timescales

| Phase | Duration | Charge Effect |
|-------|----------|---------------|
| Stepped leader | 10-200 ms | ~5 C descends along channel |
| Return stroke | 50-100 microseconds | ~5 C transferred; 30,000 K |
| Post-stroke pause | 40-80 ms | Partial channel recombination |
| Dart leader | ~1-2 ms | ~1 C additional transfer |
| Total flash | 200-500 ms | 5-20 C total transfer |

### How Lightning Affects Cloud Charge

- A CG flash removes 5-20 C from the main negative region
- **Localized**: Neutralizes cylindrical volume ~100-500 m radius around channel
- Surrounding charge largely unaffected on flash timescale
- Recovery: 20-100 seconds via continued electrification

### Multiple Flash Effects

- Successive flashes to same region deplete charge faster than rebuild
- If flash rate > ~1/minute locally, region becomes "drained"
- Subsequent flashes shift to other regions or become intracloud
- Creates observed clustering and spatial migration of activity

---

## Lightning Initiation

### Where Initiation Occurs

CG lightning initiates at the **lower boundary of the main negative region** (~5-7 km):
- Between main negative and lower positive charge
- Local field reaches ~200-500 kV/m (exceeds breakdown threshold)

IC lightning initiates at the **upper boundary** (between main negative and upper positive).

Observed initiation altitudes cluster at:
- ~4,730 m (-7 C): CG-initiating breakdown
- ~9,150 m (-41 C): IC-initiating breakdown

### Trigger Mechanism

- Hydrometeors (ice, graupel) act as field-concentrating tips
- Clusters create local fields 10-100x background
- Once small plasma region forms (~meter scale), it self-propagates

### Bidirectional Leaders

Lightning initiation is **bidirectional**:
- Positive leader: Propagates upward (smooth, continuous)
- Negative leader: Propagates downward (stepped, ~50m steps)
- Multiple branches compete; one wins based on field strength

---

## Effect of Wind on Cloud Charge

### Charge Advection

Charge is carried on hydrometeors and moves with airflow:

| Charge Region | Advection Behavior |
|---------------|-------------------|
| Upper positive | Strongly advected by upper winds; spreads into anvil |
| Main negative | Partially resists (graupel is large/dense); tied to updraft |
| Lower positive | Moves least (heaviest particles) |

Net effect: Wind shear **horizontally separates charge layers**.

### Shear Effects by Storm Type

| Shear Level | Charge Configuration |
|-------------|---------------------|
| Low (single cell) | Vertically stacked, symmetric tripole |
| Moderate (multicell) | Tilted; CG lightning on downshear side |
| High (supercell) | Severe tilt; anvil positive displaced 20-50 km |

---

## Implementation for Simulation

### Charge Region Representation

```typescript
// Each region: Gaussian ellipsoid with falloff
interface ChargeRegion {
  center: Vec3;           // Normalized coordinates
  magnitude: number;      // Relative charge (positive or negative)
  horizontalRadius: number;
  verticalRadius: number;
}

const tripole: ChargeRegion[] = [
  { center: { x: 0, y: 0.85, z: 0 }, magnitude: +40, horizontalRadius: 0.3, verticalRadius: 0.1 },
  { center: { x: 0, y: 0.55, z: 0 }, magnitude: -80, horizontalRadius: 0.4, verticalRadius: 0.15 },
  { center: { x: 0, y: 0.30, z: 0 }, magnitude: +8,  horizontalRadius: 0.2, verticalRadius: 0.08 },
];
```

### Lightning Initiation Logic

```typescript
function selectInitiationPoint(chargeField: ChargeField): Vec3 {
  // Sample candidates weighted by |E-field| * P(altitude)
  // P peaks at ~0.45 normalized (5-7 km actual)
  // Add random perturbation for natural variation
}
```

### Post-Flash Charge Depletion

```typescript
function depleteChargeAroundChannel(channel: Vec3[], chargeField: ChargeField): void {
  const depletionRadius = 0.05; // Normalized units
  const depletionFactor = 0.2;  // Retain 20% of charge

  for (const point of channel) {
    // Reduce charge density within radius
    chargeField.applyDepletion(point, depletionRadius, depletionFactor);
  }
}
```

### Charge Recovery

```typescript
function accumulateCharge(region: ChargeRegion, dt: number): void {
  const maxCharge = region.baseCharge;
  const recoveryTime = 60; // seconds

  // Exponential recovery toward maximum
  region.currentCharge += (maxCharge - region.currentCharge) * (dt / recoveryTime);
}
```

### Wind Shear Visualization

```typescript
// Apply horizontal offset to upper charge regions
function applyWindShear(regions: ChargeRegion[], shearStrength: number): void {
  regions.forEach(region => {
    const offset = (region.center.y - 0.5) * shearStrength; // Higher = more offset
    region.center.x += offset;
  });
}
```

---

## Flash Rate Control

```typescript
const flashIntervals = {
  singleCell: 5000,    // ms between flashes
  multicell: 1000,
  supercell: 300,
};

function computeFlashProbability(chargeField: ChargeField, dt: number): number {
  const maxField = chargeField.getMaxFieldStrength();
  const threshold = chargeField.breakdownThreshold;

  if (maxField < threshold * 0.9) return 0;

  return (maxField - threshold * 0.9) / (threshold * 0.1) * (dt / baseInterval);
}
```

---

## References

- Williams, E.R. (1989). The tripole structure of thunderstorms. JGR.
- Rakov, V.A. & Uman, M.A. (2003). Lightning: Physics and Effects.
- Dwyer, J.R. & Uman, M.A. (2014). The physics of lightning. Physics Reports.
- Pereyra, R. et al. (2008). Charge separation in thunderstorm conditions. JGR.
- NWS: Understanding Lightning: Thunderstorm Electrification
