# Charge Field Visualization: Physics & Design Theory

> **Purpose**: Define the physical basis, visual theory, and implementation approach for rendering charge, moisture, and ionization fields in the lightning simulation.

---

## Table of Contents

1. [The Five Field Types](#the-five-field-types)
2. [Physical Behavior: Before, During, After Strikes](#physical-behavior-before-during-after-strikes)
3. [Ionization: Formation and Dispersion](#ionization-formation-and-dispersion)
4. [Layer Interactions](#layer-interactions)
5. [Visual Representation Theory](#visual-representation-theory)
6. [Shape Definition: What Controls Structure](#shape-definition-what-controls-structure)
7. [Implementation Approach](#implementation-approach)
8. [Parameter Reference](#parameter-reference)
9. [Citations](#citations)

---

## The Five Field Types

Each field has distinct physical properties that dictate its shape, behavior, and visual representation:

| Field | Physical Basis | Geometry | Typical Scale | Persistence |
|-------|---------------|----------|---------------|-------------|
| **Ceiling Charge** | Main negative charge region at cloud base (~5-8 km) | 2D horizontal plane | 3-8 km radius pockets | Minutes (rebuilt by charging mechanism) |
| **Ground Charge** | Induced positive charge via electrostatic induction | 2D horizontal plane | Mirrors ceiling with jitter | Follows ceiling |
| **Atmospheric Charge** | Volumetric charge in mixed-phase zone | 3D spheroidal pockets | 1-5 km diameter | Seconds to minutes |
| **Moisture** | Supercooled water content driving charge separation | 3D distribution | 10-30 km extent, 1-3 km thick layers | Coupled to updraft |
| **Ionization** | Residual plasma from strike channels | 3D filamentary → dispersing | 1-4 cm core → meters | 30-100 ms functional, visible ~10 ms |

### Key Insight: Different Fields, Different Structures

- **Charge fields** (ceiling, ground, atmospheric): Broad, diffuse, slowly-evolving blobs that represent regions of charge concentration. These are **NOT** the lightning itself - they're the precondition.
- **Moisture**: Even broader, representing humidity distribution in the cloud. Affects breakdown threshold.
- **Ionization**: Starts as a **fine filament** matching the strike geometry, then disperses. This IS the aftermath of lightning.

---

## Physical Behavior: Before, During, After Strikes

### Pre-Strike Charge Behavior

**Does charge "gather" before a strike?**

No - not in a visible flow sense. Charge is deposited in place by the ice-graupel collision mechanism. What changes is **field intensity**, not charge position. The electric field concentrates locally due to:
- Hydrometeor polarization (ice crystals align with field)
- Turbulent mixing producing local fluctuations
- Continued charge deposition in active regions

**Timescale**: Minutes for initial buildup; 5-30 seconds for recovery between strikes [Pawar 2002].

**Visual implication**: Pre-strike animation should show **intensity increasing** in charge regions, not charge flowing toward a point. The only pre-strike visible phenomenon would be corona discharge from ground objects (St. Elmo's fire), which we don't currently simulate.

### During Strike

**Stepped Leader Phase (10-30 ms total)**:
- Leader carries ~5 C negative charge distributed along channel [Maggio 2009]
- Charge is deposited as channel propagates, not flowing along it
- Each step (50m, 1 microsecond) ionizes fresh air
- Speed: ~200,000 mph with pauses

**Return Stroke (100-200 microseconds)**:
- Propagates upward from ground at 1/3 speed of light
- Drains the ~5 C on the leader channel into Earth
- Peak current ~30,000 A
- Charge redistribution is nearly instantaneous on human timescales

**Visual implication**: The strike itself is too fast to show charge motion. The leader phase could show the channel forming step-by-step (we already do this). The return stroke is a single bright flash.

### Post-Strike Recovery

**Immediate (0-100 ms)**:
- Cylindrical volume ~100-500m radius around channel is depleted
- Channel remains ionized for subsequent strokes (dart leaders)
- Surrounding charge largely unaffected

**Short-term (5-30 seconds)**:
- The "charge hole" recovers via continued microphysical charging
- Recovery time constant ~5 seconds for exponential recovery [Pawar 2002]
- Multi-phase: fast initial recovery, intermediate plateau, final linear return

**Visual implication**: Show the depleted region as a "hole" in the charge field that slowly fills back in over 5-30 seconds. This is accurate - the mechanism is deposition, not flow, but the visual effect is the same.

---

## Ionization: Formation and Dispersion

### Initial Geometry

The lightning channel creates ionization with layered structure:

| Region | Diameter | Temperature | Conductivity |
|--------|----------|-------------|--------------|
| **Current-carrying core** | 1-4 cm | 20,000-30,000 K | >10,000 S/m |
| **Corona sheath** | 6-40 cm (up to 10m) | Lower | ~500 S/m |

The core begins at millimeter scale and undergoes supersonic shock expansion in microseconds, settling at a few centimeters [ScienceDirect 2018].

### Decay Timeline

| Time | Temperature | Visibility | Notes |
|------|-------------|------------|-------|
| 0 (peak) | 30,000 K | Intense white/blue | Return stroke |
| 1-10 ms | 10,000-20,000 K | Fading orange/red | Afterglow visible in high-speed video |
| 10-100 ms | Thousands K | Dim/invisible | Functionally conductive for dart leaders |
| >100 ms | ~1000 K | Dark | Warm gas column, no optical emission |

**Half-life of functional ionization**: 30-80 ms (for dart leader reuse). The continuing current between strokes maintains weak ionization via Joule heating [Springer Plasma Physics].

### Buoyancy and Dispersion

- **Rise**: The hot column is dramatically less dense and rises buoyantly at several to tens of m/s
- **Lateral spread**: Shock expansion in microseconds, then slow thermal diffusion over milliseconds
- **Wind drift**: At 10 m/s wind, 50 ms = 0.5 m lateral displacement

**Visual implication**: Ionization should:
1. Start as a **thin filament** matching the strike path geometry
2. **Fade rapidly** (visible afterglow ~10 ms)
3. **Rise and spread** - the filament becomes a diffuse column
4. **Drift with wind**
5. Persist as invisible warm air affecting subsequent strikes for ~100 ms

### Effect on Subsequent Strikes

- **<100 ms**: Dart leader reuses same channel (no branching, 5-250x faster than stepped leader)
- **>100 ms**: New stepped leader, typically new path to ground
- A typical flash has 3-4 strokes, interstroke interval 40-100 ms [weather.gov]

---

## Layer Interactions

### Moisture ↔ Charge

| Effect | Direction | Mechanism |
|--------|-----------|-----------|
| Moisture enables charging | Moisture → Charge | Supercooled water required for ice-graupel collisions |
| Vertical offset | Anti-correlated | Charge max at -15 to -25 C; moisture max lower |
| Breakdown threshold | Moisture lowers threshold (at high fields) | Above 55 kV/cm, moist air breaks down easier |
| Path selection | Leaders favor moist air | Lower resistance path |

**Spatial relationship**: Positive charge accumulates higher (drier), negative charge lower (moister). Offset ~1-3 km vertically [Saunders 2006].

### Moisture ↔ Ionization

- Electron attachment to H2O causes faster ionization decay
- Moist air has shorter ionization persistence
- Rain droplets can interact with ionized channels (coalescence enhancement)

### Charge ↔ Ionization

- Ionization provides conductive paths for subsequent strokes
- Residual ionization "seeds" can influence where the next breakdown occurs
- Post-strike charge depletion creates a hole that affects field geometry

### Wind Effects on All Fields

| Field | Wind Response |
|-------|--------------|
| Ceiling charge | Upper regions advected into anvil; main negative tied to updraft |
| Ground charge | Follows ceiling (induced) |
| Atmospheric charge | Tilted by wind shear; horizontally separated from ceiling |
| Moisture | Advected with airflow; upper regions spread into anvil |
| Ionization | Drifts with local wind; base can offset from channel top |

---

## Visual Representation Theory

### Hard Edges vs Soft Gradients

**Soft gradients are more physically accurate** for charge/moisture. These are diffuse concentrations, not objects with boundaries. Use `smoothstep` with a ~10% band:

```glsl
float alpha = smoothstep(threshold - 0.1, threshold + 0.1, totalField);
```

Hard edges (`step`) look like scientific contour maps - appropriate for educational/diagnostic modes, not naturalistic rendering.

### Metaball Merging: Why and How

**The problem**: Current implementation renders 16 independent sprites per layer. They don't visually merge - looks like random blobs.

**The solution**: Sum influence functions across all cells, apply threshold. Cells naturally merge where their contributions overlap.

**Threshold values**:
| Threshold | Effect |
|-----------|--------|
| 0.5 | Very early merge, bloated blobs |
| 0.8-1.0 | Natural merge, good for charge clouds |
| 1.2-1.5 | Tight merge, distinct lobes remain |

For atmospheric charge, **0.8-1.0** produces convincing "cumulus-like" merging [Jamie Wong, Metaballs and WebGL].

### Noise-Warped Boundaries

Break circular symmetry with FBM noise warping:

```glsl
vec2 warpedUV = uv + fbm(uv * 3.0) * 0.15;
float d = length(warpedUV - center);
```

**Parameters**:
| Parameter | Range | Effect |
|-----------|-------|--------|
| Octaves | 3-5 | 3 = smooth clouds, 5 = turbulent |
| Lacunarity | 1.8-2.2 | Frequency multiplier per octave |
| Gain | 0.4-0.6 | Amplitude reduction per octave |
| Warp strength | 0.08-0.2 | Boundary roughness (fraction of cell radius) |

**Static vs animated**: For pre-computed timeline playback, **static noise is preferable**. Animated noise creates visual noise that distracts from the lightning physics. If animating, use very slow drift (time multiplier 0.02-0.05).

### Wind Deformation

Cells should appear **elliptical**, elongated in wind direction:

```glsl
float stretchFactor = 1.0 + windSpeed * 1.5;  // 1.0-2.5x elongation
float along = dot(uv - center, windDir);
float perp = dot(uv - center, perpDir);
float dist = sqrt((along/stretchFactor)^2 + perp^2);
```

### 3D Fields: Billboards vs Volumetric

**Full volumetric ray marching**: Most accurate, 10-100x more expensive. Not justified for this use case.

**Billboarded sprites (current)**: Good for side/above view angles. Cost-effective.

**Single fullscreen plane per field type**: Best upgrade path. One fragment shader evaluates all 16 cells as metaball field. Enables merging with no per-cell geometry overhead.

**Recommendation**: Replace per-cell sprites with **one plane per field type**. Pass cell data as uniforms. Compute metaball merging in fragment shader.

---

## Shape Definition: What Controls Structure

### Charge Regions

**Physical controls**:
- Updraft intensity and position (charge concentrates near active updraft)
- Temperature (reversal at -15 C determines polarity zones)
- Wind shear (tilts and separates charge layers)

**Simulation parameters**:
- Cell count: 3-8 typical
- Cell radius: 0.08-0.24 units (500m - 1.5km)
- Cell intensity: 0.5-1.0
- Gaussian falloff to edges

**Shape character**: Horizontally elongated pancakes (wider than tall), 1-3 km thick, 10-30 km across for major layers. Pockets within are more equidimensional (1-5 km).

### Moisture Regions

**Physical controls**:
- Updraft carries moisture upward
- Temperature determines phase (supercooled liquid vs ice)
- Broader than charge regions

**Simulation parameters**:
- Cell radius: 0.08-0.32 units (500m - 2km)
- Lower intensity variation than charge
- Co-located with but vertically offset from charge

**Shape character**: Broad, diffuse, layered. Less defined boundaries than charge.

### Ionization

**Physical controls**:
- Strike geometry (starts as exact channel shape)
- Temperature decay (controls visibility)
- Buoyancy (causes rise)
- Wind (causes drift)
- Recombination (causes fade)

**Simulation parameters**:
- Initial: Match strike path exactly
- Radius: Start 0.002 units (~12m including corona), expand over time
- Intensity: Start 1.0, decay with ~30ms half-life
- Position: Rise at ~5-10 m/s, drift with wind

**Shape character**: Starts as fine branching filament, becomes diffuse column, fades.

---

## Implementation Approach

### Architecture Change

**Current**: Per-cell `PlaneGeometry` sprites with individual `ShaderMaterial`
- 16+ meshes per layer
- No inter-cell interaction
- No merging

**Proposed**: One `PlaneGeometry` per layer with metaball shader
- 1 mesh per layer (5 layers = 5 meshes)
- Cell data passed as uniforms
- Fragment shader computes merged field

### Shader Structure

```glsl
// Uniforms
uniform vec2 cellCenters[16];
uniform float cellIntensities[16];
uniform float cellRadii[16];
uniform int cellCount;
uniform vec3 baseColor;
uniform float threshold;
uniform float time;
uniform vec2 windDir;
uniform float windSpeed;

// FBM noise for boundary warping
float fbm(vec2 p) { /* 4-octave noise */ }

void main() {
    float totalField = 0.0;

    // Warp coordinates for organic boundaries
    vec2 warpedUV = vWorldXZ + fbm(vWorldXZ * 3.0) * 0.12;

    for (int i = 0; i < 16; i++) {
        if (i >= cellCount) break;

        vec2 d = warpedUV - cellCenters[i];

        // Wind deformation (elliptical stretch)
        float along = dot(d, windDir);
        float perp = dot(d, vec2(-windDir.y, windDir.x));
        float stretchFactor = 1.0 + windSpeed * 1.5;
        float dist = sqrt((along/stretchFactor)*(along/stretchFactor) + perp*perp);

        float r = cellRadii[i];
        if (dist < r) {
            float t = dist / r;
            float falloff = (cos(t * PI) + 1.0) * 0.5;
            totalField += cellIntensities[i] * falloff;
        }
    }

    // Metaball threshold with soft edge
    float alpha = smoothstep(threshold - 0.1, threshold + 0.1, totalField);
    alpha = clamp(alpha * u_opacity, 0.0, 0.35);

    gl_FragColor = vec4(baseColor, alpha);
}
```

### Ionization Special Handling

Ionization is different from other fields:
1. **Initial geometry**: Match strike path (line segments, not circles)
2. **Evolution**: Expand radius, fade intensity, rise, drift
3. **Rendering**: Could use tube geometry along path, or convert path points to cells with expanding radii

Simplest approach: Convert strike path points to small cells, animate their expansion and fade.

### Plane Sizing

The plan mentioned "2x worldScale" - this means:
- `worldScale` is the distance from ceiling to ground (~worldStart to worldEnd)
- Plane should extend beyond visible area to avoid edge clipping
- A plane covering 2x the simulation bounds in X and Z ensures full coverage
- For normalized simulation space (-0.5 to 0.5), a plane of size 2.0 covers the full extent with margin

---

## Parameter Reference

### Physical Constants

| Parameter | Value | Source |
|-----------|-------|--------|
| Charge pocket radius | 500m - 1.5km | Williams 1989 |
| Charge recovery time constant | ~5 seconds | Pawar 2002 |
| Ionization core diameter | 1-4 cm | Multiple |
| Ionization functional persistence | 30-100 ms | Multiple |
| Interstroke interval threshold | 100 ms | weather.gov |
| Breakdown threshold (sea level) | ~30 kV/cm | Standard |
| Breakdown threshold (10 km) | ~8-10 kV/cm | Paschen's law |

### Visualization Parameters

| Parameter | Recommended Value | Notes |
|-----------|------------------|-------|
| Metaball threshold | 0.8-1.0 | For natural merging |
| Edge smoothstep band | 0.1 | Soft edges |
| FBM octaves | 4 | Balance detail/cost |
| FBM warp strength | 0.12-0.15 | Organic but not chaotic |
| Wind stretch factor | 1.0 + windSpeed * 1.5 | Max 2.5x at full wind |
| Max opacity | 0.35 | Avoid obscuring lightning |

### Timescales for Animation

| Event | Duration | Visual Behavior |
|-------|----------|-----------------|
| Charge buildup | Minutes | Intensity increase (no motion) |
| Pre-strike | Seconds | Field intensifies |
| Strike | 10-30 ms (leader), 100 μs (return) | Channel appears |
| Ionization visible | ~10 ms | Bright afterglow |
| Ionization functional | 30-100 ms | Invisible but affects next stroke |
| Charge recovery | 5-30 seconds | Hole fills in |

---

## Citations

### Primary Sources

1. **Pawar, S. D., & Kamra, A. K. (2002)**. Recovery curves of the surface electric field after lightning discharges occurring between the positive charge pocket and negative charge centre in a thundercloud. *Geophysical Research Letters*. https://agupubs.onlinelibrary.wiley.com/doi/10.1029/2002GL015675

2. **Maggio, C. R., et al. (2009)**. Estimations of charge transferred and energy released by lightning flashes. *Journal of Geophysical Research*. https://agupubs.onlinelibrary.wiley.com/doi/full/10.1029/2008JD011506

3. **Cruz, P., et al. (2025)**. Correlation Between Speed of the Leader and Peak Current. *Geophysical Research Letters*. https://agupubs.onlinelibrary.wiley.com/doi/full/10.1029/2024GL111594

4. **Saunders, C. P. R. (2006)**. Laboratory studies of the effect of cloud conditions on graupel/crystal charge transfer in thunderstorm electrification. *Quarterly Journal of the Royal Meteorological Society*. https://rmets.onlinelibrary.wiley.com/doi/abs/10.1256/qj.05.218

5. **Kamra, A. K. (1997)**. Effect of relative humidity on the electrical conductivity of marine air. *Quarterly Journal of the Royal Meteorological Society*. https://rmets.onlinelibrary.wiley.com/doi/pdf/10.1002/qj.49712354108

6. **Williams, E. R. (1989)**. The tripole structure of thunderstorms. *Journal of Geophysical Research*.

7. **Rakov, V. A., & Uman, M. A. (2003)**. *Lightning: Physics and Effects*. Cambridge University Press.

### Technical Resources

8. **NOAA National Weather Service**. Lightning Science Series.
   - Stepped Leader Initiation: https://www.weather.gov/safety/lightning-science-initiation-stepped-leader
   - Dart Leaders: https://www.weather.gov/safety/lightning-science-dart-leaders
   - Electrification: https://www.weather.gov/safety/lightning-science-electrification

9. **Quilez, I.** Smooth Minimum. https://iquilezles.org/articles/smin/

10. **Quilez, I.** Fractal Brownian Motion. https://iquilezles.org/articles/fbm/

11. **The Book of Shaders, Chapter 13**. Fractal Brownian Motion. https://thebookofshaders.com/13/

12. **Wong, J. (2016)**. Metaballs and WebGL. https://jamie-wong.com/2016/07/06/metaballs-and-webgl/

13. **Codrops (2025)**. How to Create Interactive Metaballs with Three.js and GLSL. https://tympanus.net/codrops/2025/06/09/how-to-create-interactive-droplet-like-metaballs-with-three-js-and-glsl/

### Physics Data

14. **ScienceDirect (2022)**. Initial radius and discharge intensity of lightning return strokes. https://www.sciencedirect.com/science/article/abs/pii/S0169809522001478

15. **ScienceDirect (2018)**. Lightning channel expansion theory. https://www.sciencedirect.com/science/article/abs/pii/S1364682618306515

16. **AGU (2021)**. Vertical Temperature Profile of Natural Lightning Return Strokes. https://agupubs.onlinelibrary.wiley.com/doi/full/10.1029/2020JD034438

17. **Springer Plasma Physics (2025)**. Plasma in Conducting Channel of Lightning. https://link.springer.com/article/10.1134/S1063780X25603025

---

## Summary

### What We're Visualizing

1. **Charge fields**: Broad, diffuse regions where charge concentrates. Slowly evolving. Shape controlled by updraft, temperature, wind. Rendered as merged metaball blobs with organic noise-warped boundaries.

2. **Moisture**: Even broader distribution. Affects breakdown threshold. Anti-correlated vertically with charge. Rendered similarly to charge but with different parameters.

3. **Ionization**: Aftermath of strikes. Starts as fine filament matching strike path. Rapidly fades (~10ms visible), rises, spreads, drifts. Affects subsequent strokes for ~100ms.

### Key Design Principles

- **No visible charge flow**: Charge doesn't flow to the strike point. Intensity increases in place.
- **Post-strike hole**: The depleted region fills in over 5-30 seconds via deposition, not flow.
- **Ionization is different**: It's a filament that disperses, not a blob that appears.
- **Soft boundaries**: Use smoothstep, not hard edges.
- **Metaball merging**: Cells should blend where they overlap.
- **Wind deformation**: Elliptical stretch, not circular shapes.
- **Static noise**: Organic boundaries, but not animated (distracting).

### Next Steps

1. Rewrite `ChargeFieldRenderer.ts` to use single-plane metaball approach
2. Implement new fragment shader with FBM warping and metaball merging
3. Add wind deformation to shader
4. Implement ionization as expanding/fading cells along strike path
5. Add post-strike charge depletion and recovery animation

---

## Why Current Rendering Looks Like "Unintentional Mess"

### The Core Problem

The current `ChargeFieldRenderer` uses **radial cosine-falloff sprites** with additive blending. Each charge cell becomes a circular soft gradient disk. When multiple blobs overlap, the result is an **undifferentiated glow field with no internal structure** - visually equivalent to a blurred noise texture.

**Soft gradients fail when**:
1. Sources are **isotropic** (circular blobs have no preferred direction to read)
2. Multiple sources **overlap** (the sum is flatter than individual sources)
3. Opacity is **very low** (each blob individually fades below perceptual threshold)

The fundamental issue: **overlapping isotropic soft gradients produce flat, structureless fog**.

### What Makes Visualizations Look "Intentional"

Intentional storm energy has **structure at multiple scales**:
- **Large-scale**: Overall shape of the charge region (billowing forms, clearly bounded)
- **Medium-scale**: Distinct sub-regions with visible internal organization
- **Small-scale**: Texture/detail suggesting turbulence

**Messy** looks like: uniform glow fog with no center, no edge, no hierarchy.

---

## Visual Design Options: Beyond Soft Gradients

### Option 1: Concentric Threshold Rings

Instead of one smooth gradient per cell, render **3 concentric transparent rings** at `r*0.3`, `r*0.6`, `r*1.0` with decreasing opacity. The rings give the viewer a **readable structure**.

```glsl
// Multiple concentric rings per cell
float ringAlpha = 0.0;
float thresholds[3] = float[](0.3, 0.6, 1.0);
float opacities[3] = float[](0.4, 0.25, 0.1);

for (int t = 0; t < 3; t++) {
    float ringDist = abs(dist - cellRadii[i] * thresholds[t]);
    float ringWidth = 0.02;
    float ring = 1.0 - smoothstep(0.0, ringWidth, ringDist);
    ringAlpha += ring * opacities[t];
}
```

**Effect**: Charge regions appear as **nested shells** rather than blobs. The viewer's eye reads the rings as isocontours.

### Option 2: Edge-Emphasis (Ring-Bright, Interior-Dark)

Replace `(cos(dist * PI) + 1) * 0.5` with a **shell falloff** that's bright at the edge, dark inside:

```glsl
float shell = max(0.0, 1.0 - smoothstep(0.7, 1.0, t)) * (1.0 - smoothstep(0.0, 0.3, t));
```

**Effect**: Charge regions feel like **boundaries/shells** rather than filled volumes. Energy "concentrates at edges" - a visual language that reads as electric/energetic.

### Option 3: Contour Lines (Weather Radar Style)

Render the summed field at **discrete threshold levels** as anti-aliased lines:

```glsl
float contourLine(float value, float spacing) {
    float f = abs(fract(value / spacing) - 0.5);
    float df = fwidth(value / spacing) * 1.5;
    return smoothstep(0.0, df, f);
}

// Usage: 0 at isoline centers, 1 away from them
float mask = contourLine(totalField, 0.2);
vec3 color = mix(lineColor, baseColor, mask);
```

**Effect**: Scientific/topographic appearance. Hard discontinuities *are* the information. Best for educational/diagnostic modes.

### Option 4: Filled Bands + Outline Strokes

Composite filled regions with thin outline strokes at thresholds:

```glsl
// Layer 1: low-intensity fill (0.2-0.5 range)
float band1 = smoothstep(0.2, 0.22, field) * smoothstep(0.5, 0.48, field);
col = mix(col, vec4(0.2, 0.4, 0.8, 0.3), band1);

// Contour stroke at threshold
float line = 1.0 - smoothstep(0.0, fwidth(field) * 2.0, abs(field - 0.5));
col.rgb += vec3(1.0) * line * 0.8;
```

**Effect**: Combines soft fills with structured edges. Best of both worlds.

### Option 5: Procedural Hatching

Map field intensity to hatch density (cross-hatching lines):

```glsl
float hatchPattern(vec2 uv, float angle, float freq) {
    vec2 rotUv = vec2(
        uv.x * cos(angle) - uv.y * sin(angle),
        uv.x * sin(angle) + uv.y * cos(angle)
    );
    float lines = abs(fract(rotUv.y * freq) - 0.5);
    return smoothstep(0.0, fwidth(rotUv.y * freq), lines - 0.2);
}
```

**Effect**: Illustrative/scientific appearance. Dense hatching = high intensity. Non-photorealistic but highly readable.

### Option 6: Animated Flow Lines (LIC-lite)

Advect UV coordinates along field gradient over time:

```glsl
vec2 vel = getFieldGradient(vUv);
vec2 flowUv = vUv + vel * uTime * 0.05;
float stripe = fract(dot(flowUv, normalize(vel)) * 10.0 - uTime * 0.5);
float line = smoothstep(0.0, 0.1, stripe) * smoothstep(1.0, 0.9, stripe);
```

**Effect**: Shows field *direction*, not just magnitude. Animated flow makes it read as "live energy" rather than "static stain".

### Option 7: Particle-Based Field

Replace continuous surfaces with **particles advected by the field**:

- Spawn particles proportional to field intensity
- Move particles along field gradient each frame
- Density = visible intensity

**Effect**: Dynamic, energetic. Avoids the flat-fog problem entirely. Higher implementation cost (GPGPU).

---

## Recommended Approach: Hybrid

**Primary**: Single-plane metaball shader with **edge-emphasis** (bright rings at boundaries) + **multiple threshold bands** (filled at 3 intensity levels).

**Why this works**:
1. **Merging** via metaball math (cells connect)
2. **Structure** via multiple thresholds (viewer sees levels)
3. **Energy** via bright edges (reads as electric field)
4. **Organic** via FBM noise warp (not perfect circles)

```glsl
// Compute summed field as before, then:

// Multiple filled bands with discrete steps
vec3 col = vec3(0.0);
float bands[4] = float[](0.2, 0.4, 0.6, 0.8);
vec3 colors[4] = vec3[](
    vec3(0.1, 0.2, 0.4),   // dim blue
    vec3(0.2, 0.4, 0.7),   // medium blue
    vec3(0.4, 0.6, 0.9),   // bright blue
    vec3(0.7, 0.85, 1.0)   // near-white
);

for (int i = 0; i < 4; i++) {
    float inBand = smoothstep(bands[i] - 0.02, bands[i] + 0.02, totalField);
    col = mix(col, colors[i], inBand * 0.6);
}

// Edge emphasis: bright contour at each band boundary
for (int i = 0; i < 4; i++) {
    float line = 1.0 - smoothstep(0.0, fwidth(totalField) * 2.0, abs(totalField - bands[i]));
    col += vec3(0.8, 0.9, 1.0) * line * 0.5;
}
```

---

## Additional Phenomena Worth Modeling

### High Priority (fits existing systems, high visual payoff)

| Phenomenon | Description | Implementation |
|------------|-------------|----------------|
| **Stepped leader corona envelope** | Diffuse glow around leader tip as it propagates | Additive blur/halo around leader segments |
| **Continuing current glow** | Orange-red dim glow persisting 40-500ms after return stroke | Color/intensity decay curve on channel |
| **Corona discharge / St. Elmo's fire** | Blue-violet glow from ground points pre-strike | Additive shader on ground objects at high field |
| **Channel luminosity variation** | Branches dimmer than main channel | Brightness proportional to branch weight |
| **Precipitation curtains** | Visible rain sheets below cloud | Density gradient in existing rain system |

### Medium Priority (medium cost, contextual value)

| Phenomenon | Description | Implementation |
|------------|-------------|----------------|
| **Upward streamers** | Short tendrils reaching up from ground toward leader | Spawn upward filaments in final ms of leader phase |
| **Thunder wave** | Expanding acoustic wavefront | Post-process expanding ring at sound speed |
| **Updraft/downdraft streamlines** | Vertical flow visualization | Animated vertical particles |

### Skip (low value or high cost)

- Space stems / pilot leaders (nearly invisible, looks like artifacts)
- Electromagnetic pulse (fully invisible, pure abstraction)
- Heat shimmer (too subtle at scene scale)
- Bead lightning (rare, marginal visual impact)

### The Highest-Leverage Addition

**Stepped leader corona envelope**: Currently the leader phase looks like a dim version of the return stroke (same geometry, just dimmer). Adding a **diffuse glow envelope** around the propagating leader tip makes it visually distinct as a *process* (plasma expanding into air) rather than a dim flash.

---

## Hierarchy and Readability

### The Problem of Equal Visual Weight

Showing all five fields (ceiling, ground, atmospheric, moisture, ionization) at the same visual weight eliminates hierarchy. The viewer's eye has no anchor.

### Recommended Hierarchy

| Priority | Field | Visual Treatment |
|----------|-------|------------------|
| **Primary** | Ceiling charge | Bold, high contrast, clearly bounded |
| **Primary** | Ground charge | Same as ceiling (the dipole that drives lightning) |
| **Secondary** | Atmospheric charge corridors | Subtler, may overlap ceiling |
| **Tertiary** | Moisture | Even subtler, contextual only |
| **Special** | Ionization | Different visual language (filamentary, not blob) |

**Implementation**: Use different opacities and color saturation levels. Primary fields at 0.3-0.4 alpha, secondary at 0.15-0.2, tertiary at 0.1 or off by default.

---

## Additional Citations: Visual Design Research

### Scientific Visualization

18. **HellerWeather**. Data Visualization and the Rainbow Color Table. https://hellerweather.com/data-visualization-and-the-overused-rainbow-color-table/

19. **BAMS (2023)**. Effective Visualization of Radar Data for Color Vision Deficiency. https://journals.ametsoc.org/view/journals/bams/105/8/BAMS-D-23-0056.1.xml

20. **Wikipedia**. Line Integral Convolution. https://en.wikipedia.org/wiki/Line_integral_convolution

21. **LearnOpenGL**. Bloom Post-Processing. https://learnopengl.com/Advanced-Lighting/Bloom

22. **Benjamin Cheng**. Visualising Electric Fields with WebGL. https://bcheng.me/blog/visualising-electric-fields-with-webgl-kinda/

### Shader Techniques

23. **Observable (@stwind)**. GLSL Contour Lines. https://observablehq.com/@stwind/glsl-contour-lines

24. **ClickToRelease**. Cross-hatching GLSL Shader. https://www.clicktorelease.com/code/cross-hatching/

25. **GameIdea.org**. Fresnel Effect GLSL. https://gameidea.org/short-posts/fresnel-effect-glsl/

26. **Three.js Roadmap**. Rim Lighting Shader. https://threejsroadmap.com/blog/rim-lighting-shader

27. **InspirnNathan**. Glow Shader in Shadertoy. https://inspirnathan.com/posts/65-glow-shader-in-shadertoy/

28. **Shadertoy**. Edge Glow Tutorial. https://www.shadertoy.com/view/Mdf3zr

### Particle Systems

29. **Codrops (2024)**. Crafting a Dreamy Particle Effect with Three.js and GPGPU. https://tympanus.net/codrops/2024/12/19/crafting-a-dreamy-particle-effect-with-three-js-and-gpgpu/

30. **Three.js Journey**. GPGPU Flow Field Particles. https://threejs-journey.com/lessons/gpgpu-flow-field-particles-shaders

31. **anvaka/fieldplay**. Vector Field Explorer. https://github.com/anvaka/fieldplay

### Reference Implementations

32. **apbodnar/WebGL_LIC**. Line Integral Convolution in WebGL. https://github.com/apbodnar/WebGL_LIC

33. **philogb/LIC**. LIC in JS/Canvas/WebGL. https://github.com/philogb/LIC

34. **Will Usher**. Volume Rendering with WebGL. https://www.willusher.io/webgl/2019/01/13/volume-rendering-with-webgl/

### Lightning Phenomena

35. **NOAA JetStream**. The Lightning Process. https://www.noaa.gov/jetstream/lightning/how-lightning-is-created/jetstream-max-lightning-process-keeping-in-step

36. **AGU/GRL (2022)**. Close View of the Lightning Attachment Process. https://agupubs.onlinelibrary.wiley.com/doi/10.1029/2022GL101482

37. **Wikipedia**. St. Elmo's Fire. https://en.wikipedia.org/wiki/St._Elmo's_fire

38. **NOAA/NWS**. Continuing Current/Hot Lightning. https://www.weather.gov/safety/lightning-science-continuing-current

39. **Britannica**. Bead Lightning. https://www.britannica.com/science/bead-lightning

40. **Southwest Research Institute**. Seeing Thunder. https://www.swri.org/newsroom/technology-today/technology-today/seeing-thunder

---

## Summary: Making It Look Better

### The Fix

1. **Replace per-cell sprites with single metaball plane** - enables merging
2. **Add multiple threshold bands** - creates visible structure/levels
3. **Add edge-emphasis** - bright contours at band boundaries read as "energy"
4. **Keep FBM noise warp** - organic boundaries, not perfect circles
5. **Establish hierarchy** - primary fields bold, secondary subtle, tertiary off by default
6. **Consider stepped leader corona** - makes pre-strike phase visually distinct

### What NOT to Do

- Don't just increase opacity (makes it foggy, not structured)
- Don't animate the noise (distracting, looks like rendering artifact)
- Don't show all 5 layers at equal weight (no hierarchy)
- Don't use pure soft gradients (produces flat undifferentiated glow)

### The Core Insight

**Structure comes from discontinuities**. Weather radar works because it has hard edges between color bands. Scientific field visualizations work because they have contour lines. Soft gradients only work when there's a clear directional gradient to read. Isotropic overlapping soft blobs = structureless fog.

The solution: **add discontinuities** via threshold bands, contour lines, or edge emphasis - while keeping the organic shape via noise warping and metaball merging.

---

## Implementation Decision (2026-02-19)

### What We Tried and Why It Failed

Attempted to render volumetric fields (atmospheric, moisture, ionization) as 3D spheres using nested icosahedron shells with additive blending and derivative-based normals. After 10+ iterations of shader tweaks, the result was still flat 2D circles with concentric rings.

**Root cause**: The rendering architecture itself (DoubleSide + AdditiveBlending + nested shells) is fundamentally incompatible with directional lighting:
- DoubleSide renders both faces; shader flips back-face normals to face camera
- Both faces now have identical lighting; they add together via additive blending
- Result: uniform brightness, no directional shading survives

No amount of shader parameter adjustment fixes this. The structure must change.

### Final Decision: Hybrid Rendering

| Field Type | Approach | Rationale |
|------------|----------|-----------|
| Ceiling/Ground | Single-plane metaball | Already works |
| Atmospheric | Single-plane metaball | Should merge; physically "pancakes" not spheres |
| Moisture | Raymarched sphere impostors | Discrete 3D pockets; need true volumetric look |
| Ionization | Raymarched sphere impostors | Small 3D points; same as moisture |

**Raymarched impostors**: Camera-facing quads with fragment shader ray-sphere intersection. Produces perfect smooth normals analytically (`normal = normalize(hitPoint - center)`), no tessellation artifacts, GPU-friendly.

**See**: `documentation/rendering/charge-field-visualization.md` for full technical details.
