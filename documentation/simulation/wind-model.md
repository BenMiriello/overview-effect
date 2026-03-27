# Wind Model

This document describes the atmospheric wind model used in the lightning simulation, including wind profiles, shear, and turbulence.

---

## Physical Basis

Wind in the atmosphere varies with altitude due to surface friction, pressure gradients, and Coriolis effects. In thunderstorm environments, wind also interacts with updrafts, downdrafts, and convective circulations.

---

## Wind Speed Profiles

### Power Law (Hellmann Profile)

The standard model for wind speed variation with height:

```
v(z) = v_ref * (z / z_ref)^alpha
```

Where:
- `v(z)` = wind speed at height z
- `v_ref` = reference wind speed at reference height
- `z_ref` = reference height (typically 10m)
- `alpha` = power law exponent

### Exponent Values

| Terrain Type | Alpha |
|--------------|-------|
| Open water | 0.10 |
| Flat, open terrain | 0.14 |
| Scattered vegetation | 0.25 |
| Suburbs, forest | 0.30 |
| Urban, dense forest | 0.40 |

For visual drama in simulation, `alpha = 0.30` creates noticeable vertical shear.

### Logarithmic Profile (Alternative)

More accurate very close to the surface (z < 100m):

```
v(z) = (u* / k) * ln(z / z0)
```

Where:
- `u*` = friction velocity
- `k` = von Karman constant (0.41)
- `z0` = roughness length (0.01m grass to 1m suburbs)

Not necessary for this simulation given the spatial scale (cloud-to-ground).

---

## Wind Speeds in Thunderstorm Environments

| Altitude | Speed Range | Notes |
|----------|-------------|-------|
| Surface | 0-10 m/s | Calm to strong |
| 1 km | 8-25 m/s | Boundary layer top |
| 2 km (cloud base) | 10-30 m/s | |
| 5 km (mid-troposphere) | 15-50 m/s | |
| 10-12 km (cloud top) | 20-80+ m/s | Jet stream level |

---

## Wind Shear

### Definition

Wind shear is the change in wind speed or direction with altitude. Both are important:

- **Speed shear**: Wind faster aloft than at surface
- **Directional shear**: Wind direction rotates with height

### Typical Values

In thunderstorm environments:
- Speed increase: 3-5x from surface to cloud top
- Directional rotation: 20-60 degrees over 0-10 km column

### Effect on Charge Distribution

Wind shear tilts and stretches charge regions:
- Upper positive charge advects faster than lower negative
- Charge centers become horizontally offset rather than stacked
- This creates tilted lightning paths (not perfectly vertical)

---

## Updrafts and Downdrafts

### Updraft Core

- Speed: 10-70 m/s vertical
- Location: Main convective cell
- Effect: Carries positive charge (ice crystals) upward

### Precipitation Downdraft

- Speed: 5-20 m/s vertical (descending)
- Driven by: Precipitation drag + evaporational cooling
- Effect: Brings negative charge (graupel) downward

### Ground-Level Outflow

- Speed: 5-20 m/s radial
- Creates gust front preceding storm
- Can extend 10-30 km ahead of main precipitation

---

## Implementation

### Normalized Height

Convert world Y to normalized height (0 = ground, 1 = ceiling):

```typescript
const normalizedY = (y - groundY) / (ceilingY - groundY);
```

### Power Law Wind Speed

```typescript
private windSpeedAtHeight(normalizedY: number): number {
  const alpha = 0.3;  // Shear exponent
  const minFraction = 0.15;  // Ground wind is 15% of ceiling wind
  const heightFactor = minFraction + (1 - minFraction) * Math.pow(normalizedY, alpha);
  return this.config.baseWindSpeed * heightFactor;
}
```

### Directional Wind Shear

Interpolate direction between ground and ceiling:

```typescript
interface WindConfig {
  windDirection: { x: number; z: number };       // Surface direction
  upperWindDirection: { x: number; z: number };  // Cloud-level direction
}

private windVectorAtHeight(normalizedY: number): { x: number; z: number } {
  const t = normalizedY;
  const lo = this.config.windDirection;
  const hi = this.config.upperWindDirection;

  // Linear interpolation of direction components
  const dx = lo.x * (1 - t) + hi.x * t;
  const dz = lo.z * (1 - t) + hi.z * t;

  // Normalize
  const len = Math.sqrt(dx * dx + dz * dz);
  const speed = this.windSpeedAtHeight(normalizedY);

  return { x: (dx / len) * speed, z: (dz / len) * speed };
}
```

### Turbulence / Gusts

Add noise-driven variation for natural appearance:

```typescript
// Accumulate time for continuous variation
this.windTime += dt;

// Global gust variation (affects all cells uniformly)
const gustFreq = 0.08;  // ~12 second cycle
const gustAmp = 0.25;   // +/- 25% speed variation
const gustX = noise3D(this.windTime * gustFreq, 0, 0) * gustAmp;
const gustZ = noise3D(0, 0, this.windTime * gustFreq + 100) * gustAmp;

// Per-cell turbulence (local variation)
const turbStrength = 0.3;  // Fraction of drift per frame
const turbX = noise3D(cell.x * 3, cell.y * 3, this.windTime * 0.3 + i * 0.01) * turbStrength * dt;
const turbZ = noise3D(cell.x * 3 + 50, cell.y * 3, this.windTime * 0.3 + i * 0.01) * turbStrength * dt;
```

---

## Suggested Default Values

```typescript
const windConfig = {
  baseWindSpeed: 0.004,                      // Units per second
  windDirection: { x: 1, z: 0 },             // Surface direction (normalized)
  upperWindDirection: { x: 0.7, z: 0.3 },    // ~25 deg rotation at cloud top
  windAlpha: 0.3,                            // Power law exponent
  gustAmplitude: 0.25,                       // +/- 25% speed variation
  gustFrequency: 0.08,                       // ~12 second gust cycle
  turbulenceStrength: 0.3,                   // Per-cell turbulence
};
```

---

## Frame Rate Independence

All wind displacements use `* dt` (delta time in seconds):

```typescript
newPos.x += windVector.x * dt;
```

For noise-based variation, accumulate a continuous time coordinate:

```typescript
this.windTime += dt;
const noise = noise3D(this.windTime * frequency, ...);
```

Never use `dt` directly as a noise input (causes jitter at varying frame rates).

---

## What's NOT Worth Simulating

- Full updraft/downdraft velocity fields (requires CFD)
- Ekman spiral (boundary layer rotation)
- NIC charge separation physics (already abstracted)
- Actual convective cell dynamics

These are beyond real-time scope and not visually necessary.

---

## Visual Effects of Wind

When implemented correctly, wind should produce:

1. **Charge drift**: Cells move across the scene over time
2. **Vertical shear**: Upper cells move faster than lower
3. **Directional shear**: Charge regions tilt/stretch over time
4. **Natural variation**: No perfectly uniform motion

---

## References

- Power law wind profile (Hellmann exponent)
- Atmospheric boundary layer dynamics
- Thunderstorm wind structure (NWS, NSSL)
