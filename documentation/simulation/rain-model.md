# Rain Model

This document describes the physics and implementation approach for rain visualization in the lightning simulation.

---

## Raindrop Physics

### Size Distribution (Marshall-Palmer)

The Marshall-Palmer distribution (1948) is the standard model:

```
N(D) = N0 * exp(-Lambda * D)
```

Where:
- `N(D)` = number of drops per unit volume per unit diameter interval (m^-3 mm^-1)
- `D` = drop diameter (mm)
- `N0` = intercept parameter (~8000 m^-3 mm^-1 for stratiform rain)
- `Lambda = 4.1 * R^(-0.21)` where R = rain rate (mm/hr)

For thunderstorms (convective rain), N0 increases to ~20,000-40,000 m^-3 mm^-1.

### Drop Size Ranges

| Rain Type | Typical Drop Diameter |
|-----------|----------------------|
| Drizzle | 0.1-0.5 mm |
| Light rain | 0.5-2 mm |
| Moderate rain | 1-4 mm |
| Heavy rain | 2-6 mm |
| Maximum stable | ~6 mm (larger drops break up) |

### Terminal Velocity

Raindrops reach terminal velocity quickly. The Beard (1976) formula:

```
v_t(D) = 9.65 - 10.3 * exp(-0.6 * D)
```

Where:
- `v_t` = terminal velocity (m/s)
- `D` = diameter (mm)

| Diameter | Terminal Velocity |
|----------|-------------------|
| 0.5 mm | ~2.1 m/s |
| 1.0 mm | ~4.0 m/s |
| 2.0 mm | ~6.5 m/s |
| 4.0 mm | ~8.5 m/s |
| 5.0 mm | ~9.1 m/s |

Larger drops plateau near 9.1 m/s due to aerodynamic flattening.

### Number Density

| Rain Intensity | Drops per m^3 |
|----------------|---------------|
| Light | 100-200 |
| Moderate | 300-500 |
| Heavy (thunderstorm) | 800-1500 |

---

## Wind Interaction

### Rain Angle

Rain angle from vertical depends on horizontal wind and terminal velocity:

```
theta = arctan(u_wind / v_terminal)
```

| Wind Speed | Small Drop (0.5mm) | Large Drop (5mm) |
|------------|-------------------|------------------|
| 3 m/s | 55 deg | 18 deg |
| 10 m/s | 78 deg | 48 deg |
| 20 m/s | 84 deg | 66 deg |

Small drops are nearly horizontal in strong wind; large drops maintain more vertical trajectory.

### Differential Response

Different drop sizes respond differently to wind:
- Small drops: Follow wind closely, nearly horizontal
- Large drops: Resist wind, more vertical
- Creates natural spread of streak angles

Implementation: Assign wind-response factor per drop size class.

### When Rain Becomes Visibly Angled

Rain appears noticeably angled at wind speeds above ~3 m/s.

---

## Rain in Thunderstorms

### Spatial Distribution

Rain is offset **downshear** from the updraft core:
- Updraft lifts precipitation to upper levels
- Wind advects ice/rain horizontally
- Precipitation falls in a shaft downwind of the updraft

### Relationship to Lightning

- Peak CG lightning rate correlates with peak precipitation flux
- BUT: Peak surface rain **lags** peak lightning by minutes
- Rain doesn't suppress lightning; slight field reduction from space charge transport

### Precipitation Downdraft

- Driven by: Drag + evaporational cooling
- Speed: 5-15 m/s descent
- Creates surface cold pool
- Gust front winds: 10-25 m/s (dramatically angles near-surface rain)

---

## Visualization Techniques

### Three-Layer Rendering Approach

For performance with visual quality, use three distance layers:

#### Near Layer (0-50 units from camera)
- **Geometry**: 3,000-5,000 instanced `LineSegments`
- **Physics**: Full per-particle wind response
- **Orientation**: Aligned along velocity vector
- **Update**: GPU vertex shader handles motion

#### Mid Layer (50-200 units)
- **Geometry**: 8,000-15,000 instanced `Points`
- **Physics**: Simplified, single average wind speed
- **Appearance**: Smaller, less detailed
- **Update**: GPU vertex shader

#### Far Layer (200+ units)
- **Geometry**: Screen-space scrolling texture quad
- **Physics**: None (pure visual effect)
- **Cost**: Extremely cheap
- **Limitation**: Breaks when looking up/down

### GPU-Driven Updates

All position updates belong in the vertex shader:

```glsl
uniform float uTime;
uniform vec2 uWind;  // Horizontal wind (x, z)
uniform float uGravity;  // Scaled terminal velocity

void main() {
  vec3 pos = aPosition;

  // Apply wind + gravity
  pos.xz += uWind * uTime;
  pos.y -= uGravity * uTime;

  // Wrap vertically (infinite rain loop)
  pos.y = mod(pos.y + uCycleHeight, uCycleHeight * 2.0) - uCycleHeight;

  gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
}
```

CPU only updates three uniforms per frame:
- `uTime` (accumulated)
- `uWind` (from wind model)
- `uCameraPos` (for distance-based LOD)

### Streak Rendering

For near-layer line segments:

```glsl
// Vertex shader outputs velocity direction to fragment
varying vec3 vVelocity;

void main() {
  vec3 velocity = vec3(uWind.x, -uGravity, uWind.y);
  vVelocity = velocity;

  // Stretch vertex along velocity for motion blur effect
  vec3 stretchedPos = aPosition + normalize(velocity) * aStretch;
  // ...
}
```

---

## Performance Considerations

### Particle Counts

With WebGL 2 instancing:
- 50,000 instanced streaks: 60fps on mid-range GPU
- Recommended baseline: 3,000 near + 8,000 mid

### Scaling with Rain Intensity

Particle count scales sublinearly with rain rate:

```
count_scale = (R / R_ref)^0.4
```

Doubling rain rate does NOT double drop count; distribution shifts toward larger, fewer drops.

### Instancing Setup (Three.js)

```typescript
const geometry = new THREE.BufferGeometry();
geometry.setAttribute('position', new THREE.BufferAttribute(basePositions, 3));
geometry.setAttribute('aOffset', new THREE.InstancedBufferAttribute(offsets, 3));
geometry.setAttribute('aSize', new THREE.InstancedBufferAttribute(sizes, 1));

const material = new THREE.ShaderMaterial({
  vertexShader: rainVert,
  fragmentShader: rainFrag,
  uniforms: {
    uTime: { value: 0 },
    uWind: { value: new THREE.Vector2() },
    uGravity: { value: 6.0 },
  },
  transparent: true,
  depthWrite: false,
});

const mesh = new THREE.InstancedMesh(geometry, material, particleCount);
```

---

## Implementation Notes

### Drop Size Classes

Use 3-4 size classes instead of continuous distribution:

| Class | Diameter | Terminal V | Wind Response | Visual |
|-------|----------|-----------|---------------|--------|
| Small | 0.5-1 mm | 3 m/s | High (0.9) | Faint, angled |
| Medium | 1-3 mm | 5.5 m/s | Medium (0.6) | Visible streaks |
| Large | 3-5 mm | 8 m/s | Low (0.3) | Prominent, vertical |

### Wind Response Factor

```typescript
const windResponse = {
  small: 0.9,   // Nearly follows wind
  medium: 0.6,
  large: 0.3,  // Resists wind
};

const effectiveWind = baseWind * windResponse[sizeClass];
```

### Cycle Wrapping

To create infinite rain without constant particle respawning:

```glsl
// Wrap position within cycle bounds
pos.y = mod(pos.y - uCycleOffset, uCycleHeight) + uCycleBottom;
```

---

## Visual Quality Checklist

- [ ] Rain appears continuous, not isolated particles
- [ ] Streaks oriented along velocity (not vertical)
- [ ] Different drop sizes visible (varying streak lengths)
- [ ] Wind affects rain angle realistically
- [ ] Near rain more detailed than far rain
- [ ] No visible popping/respawning
- [ ] Performance remains smooth (60fps target)

---

## Future Enhancements

- Ground splash effects at impact
- Windshield/lens rain (screen-space overlay)
- Rain sound integration
- Puddle/wet surface effects

---

## References

- Marshall, J.S. & Palmer, W.M. (1948). The distribution of raindrops with size.
- Beard, K.V. (1976). Terminal velocity and shape of cloud and precipitation drops.
- Tatarchuk, N. (2006). Artist-Directable Real-Time Rain Rendering. SIGGRAPH.
- GPU-Accelerated Particles with WebGL 2
