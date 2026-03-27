# Rain Physics Reference

Research for adding a rain layer to the lightning simulation.

---

## 1. Raindrop Physics

### Size Distribution: Marshall-Palmer Model

The Marshall-Palmer (1948) exponential distribution is the standard model for stratiform rain:

```
N(D) = N₀ · exp(-Λ · D)
```

Where:
- `N(D)` = number of drops per unit volume per unit diameter interval [m⁻³ mm⁻¹]
- `N₀` = 8000 m⁻³ mm⁻¹ (intercept parameter, constant in M-P)
- `D` = drop diameter [mm]
- `Λ = 4.1 · R^(-0.21)` [mm⁻¹], R = rain rate [mm/h]

For thunderstorms (convective), use a modified intercept: N₀ ≈ 20,000–40,000 m⁻³ mm⁻¹ (higher concentration of smaller drops).

**Practical size range:** 0.5 mm (drizzle) to 5–6 mm (large drops). Drops above ~6 mm are unstable and break apart.

### Terminal Velocity by Drop Size (Beard 1976, sea level)

| Diameter (mm) | Terminal Velocity (m/s) |
|---------------|------------------------|
| 0.5           | 2.1                    |
| 1.0           | 4.0                    |
| 2.0           | 6.5                    |
| 3.0           | 7.9                    |
| 4.0           | 8.8                    |
| 5.0           | 9.1                    |

**Empirical fit (Atlas et al., good to ~5%):**
```
v_t(D) = 9.65 - 10.3 · exp(-0.6 · D)   [m/s, D in mm]
```

Or simplified polynomial often used in simulations:
```
v_t(D) ≈ 3.78 · D^0.67   [m/s, D in mm]
```

The plateau near 9 m/s for large drops is due to drop flattening—large drops become oblate spheroids with increasing drag.

### Number Density

Integrating the Marshall-Palmer distribution gives total concentration:
```
N_total = N₀ / Λ   [drops/m³]
```

Typical values:
- **Light rain** (1 mm/h): ~100–200 drops/m³
- **Moderate rain** (5 mm/h): ~300–500 drops/m³
- **Heavy thunderstorm** (50 mm/h): ~800–1500 drops/m³

### Visibility Reduction (Beer-Lambert approximation)

The optical extinction from rain is roughly:
```
β_ext ≈ 1.25 × 10⁻⁴ · R^0.65   [m⁻¹], R in mm/h
```

At 50 mm/h (heavy storm), this gives ~1.4 km visibility. At 5 mm/h, ~8 km. For the simulation, use opacity that scales with rain intensity following this relationship.

---

## 2. Wind Interaction with Rain

### Trajectory in Horizontal Wind

At terminal velocity, horizontal wind directly adds to drop motion. A drop in wind speed `u` [m/s] and terminal velocity `v_t` [m/s] travels at angle θ from vertical:

```
θ = arctan(u / v_t)
```

At wind speed u = 10 m/s:
- Small drop (0.5 mm, v_t ≈ 2.1 m/s): θ ≈ 78° (nearly horizontal)
- Medium drop (2.0 mm, v_t ≈ 6.5 m/s): θ ≈ 57°
- Large drop (5.0 mm, v_t ≈ 9.1 m/s): θ ≈ 48°

**Visibly angled rain threshold:** u > ~3 m/s (Beaufort 3, gentle breeze). Below this, rain appears nearly vertical. Above ~8 m/s (Beaufort 4–5), angle becomes dramatic.

### Acceleration/Response Time

Drops don't instantly match wind speed—they have a response time τ (velocity relaxation time):
```
τ ≈ (ρ_water · D²) / (18 · μ_air · C_D · Re / 24)
```

Simplified: τ ≈ 0.1–0.5 s for rain-sized drops. For visual rendering, the steady-state approximation (drops move with wind) is accurate enough since τ << typical gust duration.

### Differential Response by Drop Size

Small drops (< 1 mm) act almost like tracers—they follow wind fluctuations closely. Large drops (> 3 mm) have more inertia and lag behind wind gusts. This creates a visual spread in streak angles during gusts that looks very natural.

**Implementation:** Assign a wind-response factor per drop class:
```
u_effective = u_wind × (1 - exp(-D / D_ref))
```
where D_ref ≈ 2–3 mm gives a natural rolloff.

### Turbulence

In a thunderstorm, turbulence adds random horizontal velocity perturbations. Model with a Kolmogorov-spectrum noise field:
- Add `±1–3 m/s` random horizontal components per particle
- Low-frequency spatial variation (large eddies): update every 0.5–2 s
- This prevents the "too-uniform" look of constant-angle rain

---

## 3. Rain in Thunderstorms

### Structure and Distribution

A mature single-cell thunderstorm has:
- **Updraft core**: precipitation-free or with small suspended drops; located near the forward flank
- **Precipitation core (forward flank)**: heaviest rain, offset downshear from updraft
- **Anvil/trailing stratiform region**: lighter rain, large horizontal extent
- **Gust front**: leading edge where downdraft diverges; high surface wind, moderate rain

Rain is heaviest where the updraft collapses (mature to dissipating stage), directly below and downshear of the main cell.

### Relationship to Lightning

Lightning is most intense during the mature stage, which coincides with maximum precipitation flux. As a rule of thumb:
- Maximum CG (cloud-to-ground) flash rate correlates with peak rainfall at ground level
- In supercell storms, the "lightning hole" (low CG rate) occurs over the main updraft—rain is lighter there too
- Most intense rain ≠ most intense lightning (lag of minutes between peak updraft and peak surface rain)

### Rain's Effect on Electric Field

Rain is electrically significant in two ways:
1. **Space charge transport**: Large drops carry charge down from the negative charge region, depositing positive space charge near ground. This partially shields the surface electric field.
2. **Point discharge (corona)**: Near-ground objects under high fields produce upward corona ions that also partially neutralize the field.

The net effect: heavy rain slightly reduces near-surface electric field magnitude but does not prevent lightning. For visual purposes, heavy rain correlates with the period just before and after peak lightning activity.

### Precipitation Downdraft

The downdraft is driven by:
1. Drag from falling precipitation (primary)
2. Evaporational cooling (enhances it)

Downdraft descent speed: 5–15 m/s in a strong storm. At the surface, the outflow spreads outward at ground level as the "cold pool." This creates the gust front with surface winds of 10–25 m/s that dramatically angles all rain.

---

## 4. Visualization Techniques

### Particle System (Line/Streak Geometry)

Represent each raindrop as a line segment oriented along its velocity vector. Streak length is:
```
streak_length = v_resultant × frame_time × scale_factor
```

A scale_factor of 2–5× elongates streaks for readability. Typical visual streak: 0.05–0.3 world-units.

**Three.js approach:** Use `THREE.LineSegments` with `THREE.InstancedBufferGeometry` for maximum throughput. Each instance is one streak: two vertices (bottom of drop, top of drop).

Or use `THREE.Points` with custom vertex shader that extrudes each point into a billboard streak—reduces vertex buffer size by 2× but requires geometry shader or double-pass.

### Instanced Mesh Approach (Recommended for Three.js)

```glsl
// Vertex shader snippet
vec3 dropPos = instancePosition + vec3(
  windX * time * windResponse,
  -terminalVelocity * mod(time + instancePhase, cycleTime),
  windZ * time * windResponse
);
vec4 projected = projectionMatrix * modelViewMatrix * vec4(dropPos, 1.0);
```

Store per-instance: `position`, `phase`, `size`, `windResponse` as InstancedBufferAttributes.

### Shader-Based Screen-Space Rain

Pros:
- Extremely cheap (single fullscreen quad pass)
- Good for background/far rain

Cons:
- Breaks entirely when camera looks up or down
- No parallax or depth interaction
- Looks like a 2D overlay when camera moves

**Verdict:** Use for background/ambient rain at distance; use particle system for foreground rain within ~100 world-units of camera.

### Layered Approach (Recommended)

1. **Near layer** (0–50 units): ~2000–5000 instanced streaks, full physics, per-particle wind response
2. **Mid layer** (50–200 units): ~5000–10000 points, simplified physics, one average wind speed
3. **Far layer** (200+ units): Screen-space texture overlay, scrolling UV, no per-particle computation

The near layer is the only one visible enough to show wind angle variation.

### Ground Wetness / Puddles

- Apply a wet surface shader pass: increase specular intensity and decrease diffuse roughness on horizontal surfaces when rain is active
- Ripple texture on puddles: animated normal map with ring ripples, tiled, scrolling, blending 2–3 layers at different scales for variance
- Rain splash sprites: billboard quads at impact points, using a flipbook texture of splash animation

---

## 5. Performance Considerations

### Particle Counts

| Scenario | Near Particles | Mid Particles | Far | Total Draw Call Cost |
|----------|---------------|---------------|-----|---------------------|
| Light rain | 1,000 | 3,000 | 1 quad | Low |
| Heavy storm | 5,000 | 15,000 | 1 quad | Medium |
| Extreme (VFX) | 20,000 | 50,000 | 1 quad | High (GPU-bound) |

WebGL 2 with instancing: 50,000 instanced streaks runs at 60fps on mid-range GPU. 100,000 is feasible with simple shaders.

### Instancing Setup

```typescript
const geometry = new THREE.InstancedBufferGeometry();
// Base geometry: single line segment (2 verts)
geometry.setAttribute('position', new THREE.BufferAttribute(
  new Float32Array([0, 0, 0,  0, 1, 0]), 3
));
// Per-instance data
geometry.setAttribute('instanceOffset', new THREE.InstancedBufferAttribute(
  offsets, 3  // x, y, z world position
));
geometry.setAttribute('instanceVelocity', new THREE.InstancedBufferAttribute(
  velocities, 3  // vx, vy, vz
));
geometry.setAttribute('instancePhase', new THREE.InstancedBufferAttribute(
  phases, 1  // random 0..1 for staggered timing
));
```

### Culling Strategies

1. **Camera-relative recycling**: Maintain a pool; recycle particles that exit the view frustum or fall below ground. Move the spawn volume with the camera.
2. **Depth culling**: Skip particles behind opaque geometry (significant win in urban/indoor scenarios—less relevant for outdoor storm).
3. **Adaptive count**: Reduce particle count based on frame time; target 60fps by adjusting near/mid layer counts.
4. **Frustum-based spawn box**: Only spawn within the camera frustum ± a margin. A 100×100×60 unit box around the camera is sufficient for near/mid layers.

### Update Strategy

Move all per-particle position updates into the vertex shader. The CPU only updates:
- Global wind vector (uniform, per frame)
- Global time (uniform, per frame)
- Camera position (uniform, per frame, for recycle logic)

Particle positions are computed entirely in the vertex shader from:
```glsl
vec3 pos = instanceOffset + instanceVelocity * mod(uTime + instancePhase * CYCLE, CYCLE);
// Wrap: when pos.y < groundY, shift pos.y up by CYCLE_HEIGHT
```

This means zero CPU-side particle position updates per frame.

---

## 6. Specific Recommendations for This Simulation

### Integration with Existing Layer System

Add a `showRain` toggle in the existing `SettingsPanel` layers section (alongside `showCharge`, `showMoisture`, etc.). Rain intensity should correlate with storm activity—scale rain particle count and speed based on the current simulation's electrical activity level.

### Wind Field

The simulation likely already models atmospheric motion to some degree. If not, add a 2D wind field (horizontal plane) that:
- Has a base direction (stormy conditions: 5–15 m/s)
- Has low-frequency noise perturbation (0.1–0.3 Hz, ±3 m/s)
- Intensifies near the gust front (leading edge of storm)

### Layered Rendering Order

```
1. Background / sky
2. Charge field layers (existing)
3. Rain far layer (screen-space)
4. Rain mid layer (instanced points)
5. Lightning bolts (existing)
6. Rain near layer (instanced streaks)  ← in front of bolts for realism
7. UI
```

Placing near rain in front of lightning bolts is accurate—rain obscures distant lightning and creates the characteristic streaky glow.

### Recommended Starting Parameters

```typescript
const RAIN_CONFIG = {
  nearCount: 3000,
  midCount: 8000,
  nearRange: 60,      // world units radius around camera
  midRange: 200,
  streakLength: 0.4,  // world units
  baseVelocity: 7.5,  // m/s (medium drop average)
  windResponseFactor: 0.6,  // fraction of wind speed applied (simulates size mix)
  cycleHeight: 80,    // world units before particle recycles
  color: new THREE.Color(0x8ab4d4),  // slightly blue-grey
  opacity: 0.35,
};
```

### Opacity and Intensity Scaling

Scale total opacity and particle count with rain intensity R [mm/h]:
```
count_scale = (R / R_reference)^0.4   // sublinear: doubling rain ≠ doubling particles
opacity_scale = min(1.0, R / 20.0)    // cap at heavy rain value
```

The sublinear count scaling reflects that higher rain rates come from larger drops (fewer but heavier), not just more drops.

---

## Key Equations Summary

```
N(D) = 8000 · exp(-4.1 · R^(-0.21) · D)      Marshall-Palmer DSD
v_t(D) = 9.65 - 10.3 · exp(-0.6D)             Beard terminal velocity [m/s]
θ = arctan(u / v_t)                             Rain angle from vertical
τ ≈ ρ_water · D² / (18 · μ_air)               Drop response time [s]
β = 1.25×10⁻⁴ · R^0.65                        Extinction coefficient [m⁻¹]
```
