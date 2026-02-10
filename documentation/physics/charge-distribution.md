# Atmospheric Charge Distribution

Lightning requires charge separation in the atmosphere. Understanding how charge distributes in thunderstorms and on the ground is essential for realistic simulation.

---

## Cloud Charge Structure

### The Tripole Model

Thunderstorms typically have a tripole charge structure:

```
     ===============================
            + + + + + + +           <- Upper positive (8-12 km)
     ===============================

     ===============================
            - - - - - - -           <- Main negative (5-8 km)
     ===============================

     ===============================
              + + + + +             <- Lower positive (3-5 km)
     ===============================
```

### Charge Magnitudes

| Region | Typical Charge | Altitude |
|--------|---------------|----------|
| Main negative center | 20-300 Coulombs | 5-8 km (-10 to -25 deg C) |
| Main positive center | Similar magnitude | Above negative |
| Lower positive center | Smaller | 3-5 km (variable) |

### Charge Separation Mechanism

The primary charging mechanism is the **non-inductive ice-graupel collision process**:

1. **Ice crystals** (small, carried upward by updrafts) acquire positive charge
2. **Graupel particles** (larger, falling due to gravity) acquire negative charge
3. Collision transfers charge between particles
4. Vertical separation creates the dipole/tripole structure

---

## Electric Fields

### Field Values at Different Locations

| Location | Electric Field |
|----------|---------------|
| Ground before discharge | 5-20 kV/m |
| Cloud base | 100-400 kV/m |
| Breakdown threshold (sea level) | ~3 MV/m |
| Breakdown threshold (5 km altitude) | ~1-2 MV/m |

The breakdown threshold decreases with altitude due to lower air density.

---

## Ground Charge: Electrostatic Induction

When negative charge accumulates at cloud base, it induces **positive charge** on the ground directly beneath.

### The Image Charge Principle

The conducting ground acts like a mirror for electric fields. Mathematically, the ground's effect is equivalent to a mirror charge of opposite sign located at equal distance below the surface.

### Physical Mechanism

1. Electrons in the ground are repelled by the negative cloud charge
2. This leaves the ground surface positively charged beneath the cloud
3. The induced charge distribution roughly mirrors the cloud charge distribution
4. Higher conductivity areas (metal structures, wet soil, water) accumulate more charge

### Our Model: Smeared Mirror with Jitter

For simulation, we model ground charge as correlated with but not identical to ceiling charge:

```
For each ceiling cell at (cx, cz):
  ground_x = cx + random_jitter (+-0.05 units, ~300m)
  ground_z = cz + random_jitter
  ground_intensity = ceiling_intensity * (0.7 to 1.0)
  ground_radius = ceiling_radius * (0.8 to 1.2)

Plus 1-2 independent cells for local conductivity features
```

**Why jitter?**
- Perfect mirror would create too-straight paths
- Real ground has conductivity variations
- Leaders should flow from ceiling peaks toward *corresponding* ground peaks, but with variety

**Why independent cells?**
- Metal structures, water bodies, and other high-conductivity features
- Creates occasional "attractors" not directly beneath ceiling charge

---

## Charge Pockets: Voronoi Cell Model

Real charge doesn't distribute uniformly. It concentrates in **pockets** driven by convection patterns.

### Characteristics

- **Size**: 500m to 1.5 km radius
- **Shape**: Roughly circular/elliptical
- **Intensity**: Varies by pocket, strongest near convective cores
- **Spacing**: Multiple pockets per storm cell

### Sinusoidal Falloff

Rather than hard boundaries, we model each pocket with smooth falloff:

```
value(distance) = intensity * (cos(distance/radius * pi) + 1) / 2
```

This creates:
- Maximum intensity at pocket center
- Smooth decrease toward edges
- Zero value beyond radius
- No discontinuities when pockets overlap

Multiple overlapping pockets blend additively.

---

## Starting Point Selection

Leaders don't start from arbitrary cloud positions. They originate from **charge peaks**.

### The Process

1. Generate ceiling charge field with N cells (typically 3-8)
2. Identify local maxima (cell centers ordered by intensity)
3. Filter by minimum intensity threshold
4. Spawn one potential leader per significant peak
5. Number of leaders emerges from charge distribution, not configuration

### Multi-Leader Competition

If multiple charge peaks exist:
- Multiple leaders descend simultaneously
- They compete based on progress toward ground
- First to reach ground "wins" and becomes the main channel
- Others become branches or terminate

This creates authentic seeking behavior where "which path wins" is genuinely undetermined until connection.

---

## Ground Charge Influence on Path

Ground charge only affects the lightning path when the leader gets close:

### Distance-Modulated Attraction

```typescript
const GROUND_CHARGE_RANGE = 0.08;  // ~500m in real terms
const proximityFactor = 1 - (groundDist / GROUND_CHARGE_RANGE);

if (groundDist < GROUND_CHARGE_RANGE) {
  field += groundCharge.getValue(point) * proximityFactor * WEIGHT;
}
```

This creates:
- No ground influence at high altitude
- Gradually increasing attraction as leader descends
- Final termination attracted toward charge peaks

---

## Scale Reference

| Physical Feature | Real Size | Simulation Units |
|-----------------|-----------|------------------|
| Charge pocket (min) | 500 m | 0.08 units |
| Charge pocket (max) | 1.5 km | 0.24 units |
| Ground influence range | 500 m | 0.08 units |
| Jitter for ground cells | 300 m | 0.05 units |

---

## Visualization

In our showcase mode, charge distribution can be visualized:

- **Ceiling charge**: Blue-white semi-transparent plane at cloud level
- **Ground charge**: Warm-tinted semi-transparent plane at ground level
- Both use shader-based rendering matching the Voronoi sinusoidal falloff
- Togglable for educational display

---

## References

- Rakov & Uman (2003) - Cloud electrification and charge structure
- Dwyer & Uman (2014) - Electric field measurements
- Williams (1989) - The tripole structure of thunderstorms
