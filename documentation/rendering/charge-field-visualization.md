# Charge Field Visualization

This document describes techniques for rendering realistic charge field visualizations in the lightning simulation.

---

## Current Implementation Issues

The existing `ChargeFieldRenderer` renders each Voronoi cell as a separate sprite mesh with cosine falloff. With up to 16 independently-rendered circular blobs at low opacity, the result is disconnected fuzzy circles that have no visual coherence.

**The core problem**: Cells are treated as independent point-glow sources rather than a continuous field.

---

## Recommended Approach: Single-Plane Shader Rendering

### Architecture Change

Replace per-cell sprite meshes with **one plane per charge layer** that evaluates the entire field in the fragment shader.

| Layer | Geometry | Orientation |
|-------|----------|-------------|
| Ceiling charge | `PlaneGeometry` | Horizontal at ceiling Y |
| Ground charge | `PlaneGeometry` | Horizontal at ground Y |
| Atmospheric charge | `PlaneGeometry` | Vertical slab or volume |

### Why This Works

- Merged field computation creates **irregular blended boundaries** where cells overlap
- Cells "connect" rather than sitting as isolated blobs
- No per-cell geometry: O(1) draw calls instead of O(n)
- Fragment shader evaluates continuous field at every pixel

---

## Core Fragment Shader

```glsl
uniform vec2 cellCenters[16];      // XZ positions in world space
uniform float cellIntensities[16];
uniform float cellRadii[16];
uniform int cellCount;
uniform vec3 baseColor;
uniform float time;
uniform float threshold;           // For metaball merging
uniform float edgeWidth;

varying vec2 vWorldXZ;             // From vertex shader

// Simple 2D noise for boundary warping
float hash(vec2 p) {
  return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453);
}

float noise(vec2 p) {
  vec2 i = floor(p);
  vec2 f = fract(p);
  vec2 u = f * f * (3.0 - 2.0 * f);
  return mix(mix(hash(i), hash(i + vec2(1, 0)), u.x),
             mix(hash(i + vec2(0, 1)), hash(i + vec2(1, 1)), u.x), u.y);
}

float fbm(vec2 p) {
  float v = 0.0;
  float a = 0.5;
  vec2 shift = vec2(100.0);
  mat2 rot = mat2(cos(0.5), sin(0.5), -sin(0.5), cos(0.5));
  for (int i = 0; i < 4; i++) {
    v += a * noise(p);
    p = rot * p * 2.0 + shift;
    a *= 0.5;
  }
  return v;
}

void main() {
  float fieldValue = 0.0;

  for (int i = 0; i < 16; i++) {
    if (i >= cellCount) break;

    vec2 d = vWorldXZ - cellCenters[i];

    // Warp distance with noise for irregular boundaries
    float noiseWarp = fbm(vWorldXZ * 2.5 + vec2(float(i) * 17.3, float(i) * 31.7));
    float dist = length(d) * (1.0 - 0.25 * noiseWarp);

    float r = cellRadii[i];
    if (dist < r) {
      float t = dist / r;
      // Cosine falloff (same as before, but summed)
      float falloff = (cos(t * 3.14159) + 1.0) * 0.5;
      fieldValue += cellIntensities[i] * falloff;
    }
  }

  // Metaball merging: threshold creates defined edges
  float alpha = smoothstep(threshold - edgeWidth, threshold + edgeWidth, fieldValue);

  // Optional: animated turbulence
  float turbulence = fbm(vWorldXZ * 1.5 + time * 0.05) * 0.1;
  alpha *= (1.0 + turbulence);

  // Clamp to reasonable opacity
  alpha = clamp(alpha, 0.0, 0.35);

  gl_FragColor = vec4(baseColor, alpha);
}
```

---

## Key Techniques

### 1. Noise-Warped Boundaries

Add FBM noise to the distance calculation:

```glsl
float noiseWarp = fbm(worldXZ * 2.5 + cellSeed);
float dist = length(d) * (1.0 - warpAmplitude * noiseWarp);
```

- **Warp amplitude**: 20-30% of cell radius
- **Noise frequency**: 2-4x world scale
- **Cell seed**: Use `float(i) * 17.3` for per-cell variety

This breaks circular symmetry and creates organic, irregular shapes.

### 2. Metaball Merging

Apply smoothstep threshold to make overlapping regions merge:

```glsl
float alpha = smoothstep(threshold - edge, threshold + edge, summedField);
```

- **Threshold**: 0.2-0.4 (controls where "solid" region begins)
- **Edge width**: 0.05-0.15 (controls sharpness of boundary)

Result: Overlapping charge regions become one contiguous mass instead of overlapping circles.

### 3. Elliptical Cells (Wind Stretch)

Use anisotropic distance for wind-stretched shapes:

```glsl
uniform vec2 windDir;
uniform float windStretch;  // 1.5 to 2.0

vec2 d = worldXZ - cellCenter;
float windAligned = dot(d, windDir);
float windPerp = dot(d, vec2(-windDir.y, windDir.x));
float dist = sqrt(pow(windAligned / (r * windStretch), 2.0) + pow(windPerp / r, 2.0));
```

This makes charge pockets appear elongated in the wind direction.

### 4. Animated Turbulence

Add subtle motion for a "living" field:

```glsl
float turbulence = fbm(worldXZ * 1.5 + time * 0.05) * 0.1;
alpha *= (1.0 + turbulence);
```

---

## Implementation Steps

### Step 1: Replace Sprite Architecture

```typescript
// OLD: One mesh per cell
for (const cell of cells) {
  const mesh = new THREE.Mesh(planeGeom, cellMaterial);
  scene.add(mesh);
}

// NEW: One plane for entire layer
const ceilingPlane = new THREE.Mesh(
  new THREE.PlaneGeometry(planeSize, planeSize),
  chargeFieldMaterial
);
ceilingPlane.rotation.x = -Math.PI / 2;  // Horizontal
ceilingPlane.position.y = ceilingY;
scene.add(ceilingPlane);
```

### Step 2: Pass Cell Data as Uniforms

```typescript
const uniforms = {
  cellCenters: { value: new Float32Array(32) },    // 16 cells x 2 components
  cellIntensities: { value: new Float32Array(16) },
  cellRadii: { value: new Float32Array(16) },
  cellCount: { value: cells.length },
  baseColor: { value: new THREE.Color(0.7, 0.85, 1.0) },
  threshold: { value: 0.3 },
  edgeWidth: { value: 0.1 },
  time: { value: 0 },
};

// Update each frame:
function updateUniforms(cells: VoronoiCell[]): void {
  const centers = uniforms.cellCenters.value;
  const intensities = uniforms.cellIntensities.value;
  const radii = uniforms.cellRadii.value;

  for (let i = 0; i < cells.length && i < 16; i++) {
    centers[i * 2] = cells[i].center.x;
    centers[i * 2 + 1] = cells[i].center.z;
    intensities[i] = cells[i].intensity;
    radii[i] = cells[i].falloffRadius;
  }
  uniforms.cellCount.value = Math.min(cells.length, 16);
}
```

### Step 3: Vertex Shader

```glsl
varying vec2 vWorldXZ;

void main() {
  vec4 worldPos = modelMatrix * vec4(position, 1.0);
  vWorldXZ = worldPos.xz;
  gl_Position = projectionMatrix * viewMatrix * worldPos;
}
```

---

## Field Deformation During Lightning

### Leader Propagation Phase

As the stepped leader descends, charge should visually "pinch" toward the channel:

```typescript
// During leader phase, modify cell positions toward strike origin
function deformForLeader(cells: VoronoiCell[], strikeOrigin: Vec3, progress: number): void {
  for (const cell of cells) {
    const dist = distance2D(cell.center, strikeOrigin);
    if (dist < deformRadius) {
      const pullStrength = progress * maxPullStrength * (1 - dist / deformRadius);
      const direction = normalize(subtract(strikeOrigin, cell.center));
      cell.center.x += direction.x * pullStrength;
      cell.center.z += direction.z * pullStrength;
    }
  }
}
```

### Return Stroke Phase

Immediate collapse of struck cell:

```typescript
function onReturnStroke(cells: VoronoiCell[], strikePosition: Vec3): void {
  // Find nearest cell, set intensity to near-zero
  const nearestIdx = findNearestCell(cells, strikePosition);
  cells[nearestIdx].intensity *= 0.15;  // Retain 15%

  // Reduce nearby cells
  for (let i = 0; i < cells.length; i++) {
    if (i === nearestIdx) continue;
    const dist = distance2D(cells[i].center, strikePosition);
    if (dist < dissipationRadius) {
      cells[i].intensity *= 0.4;  // Retain 40%
    }
  }
}
```

---

## Alternative: Two-Pass Metaball Rendering

For maximum visual quality:

### Pass 1: Field Values to Texture

```glsl
// Render to offscreen FBO
// Fragment outputs raw field value
gl_FragColor = vec4(fieldValue, 0.0, 0.0, 1.0);
```

### Pass 2: Threshold and Color

```glsl
// Screen-space pass
float fieldSample = texture2D(fieldTexture, vUv).r;
float alpha = smoothstep(threshold - edge, threshold + edge, fieldSample);

// Color gradient based on field strength
vec3 color = mix(edgeColor, coreColor, smoothstep(threshold, threshold + 0.4, fieldSample));
gl_FragColor = vec4(color, alpha * maxOpacity);
```

This produces regions with **defined edges** that merge smoothly.

---

## Performance Considerations

### Cell Count
- 16 cells maximum per layer (uniform array limit in GLSL ES 2.0)
- 5-8 typical for charge layers
- More than sufficient for visual effect

### Shader Complexity
- Fragment shader loops through 16 cells
- FBM adds 4 noise octaves
- Still <100 ALU operations per pixel
- No performance concern on modern GPUs

### Draw Calls
- OLD: 16+ draw calls per layer
- NEW: 1 draw call per layer
- Significant improvement

---

## Visual Quality Checklist

- [ ] Fields appear as continuous regions, not isolated circles
- [ ] Boundaries are irregular and organic
- [ ] Overlapping cells merge into contiguous mass
- [ ] Wind stretches cells in wind direction
- [ ] Subtle animation gives "living" appearance
- [ ] No visible hard edges or discontinuities
- [ ] Field deforms toward lightning channel during strike

---

## References

- Metaballs and WebGL (Jamie Wong)
- Drawing 2D Metaballs with WebGL2 (Codrops)
- Visualising Electric Fields with WebGL (Benjamin Cheng)
- GPU Gems: Procedural Noise

---

## Implementation Decision: Hybrid Rendering (2026-02-19)

### Background

Multiple attempts to render volumetric charge fields (atmospheric, moisture, ionization) as 3D spheres failed. Each attempt tweaked shader parameters without addressing the fundamental rendering architecture problem.

### What Failed and Why

**Approach: Nested icosahedron shells with additive blending**

Each charge cell was rendered as 5 nested icosahedron meshes (radii 0.3, 0.5, 0.7, 0.85, 1.0) with:
- `side: THREE.DoubleSide`
- `blending: THREE.AdditiveBlending`
- `depthWrite: false`
- Derivative-based normals (dFdx/dFdy)
- Diffuse lighting calculations

**Result**: Flat 2D circles with concentric rings. No 3D shading visible.

**Root causes**:

1. **DoubleSide + normal flip = directional lighting cancels**
   - Back faces render with normals flipped to face camera
   - Back face lit from opposite direction as front face
   - With AdditiveBlending, front + back = uniform brightness
   - No directional variation survives

2. **5 shells all add together**
   - Each shell contributes to the same pixel
   - Additive blending sums them all
   - Result is radial gradient, not 3D surface

3. **dFdx/dFdy produces faceted normals**
   - Screen-space derivatives are constant per-triangle
   - With IcosahedronGeometry level 2 (80 faces), visible polygon edges
   - Would need level 4+ (1280+ faces) to hide facets

4. **depthWrite + transparent + NormalBlending = depth rejection artifacts**
   - If we tried NormalBlending with depthWrite, overlapping transparent objects get incorrectly culled

**Key insight**: No shader parameter tweaks can fix this. The rendering architecture itself (nested shells + additive blending) is incompatible with directional lighting.

### Chosen Solution: Hybrid Rendering

Different field types have different physical characteristics and should render differently:

| Field Type | Rendering Approach | Rationale |
|------------|-------------------|-----------|
| **Ceiling/Ground** | Single-plane metaball | Already implemented, works well |
| **Atmospheric charge** | Single-plane metaball | Should merge like ceiling/ground; physically described as "horizontally elongated pancakes" |
| **Moisture** | Raymarched sphere impostors | Discrete 3D water pockets; need true volumetric appearance |
| **Ionization** | Raymarched sphere impostors | Small, bright 3D points; same rationale as moisture |

### Why This Works

**Single-plane metaball** (for atmospheric):
- Cells naturally merge where they overlap
- Matches ceiling/ground aesthetic
- No per-cell geometry overhead
- Fragment shader computes field at every pixel

**Raymarched sphere impostors** (for moisture/ionization):
- Screen-aligned quad per cell, always faces camera
- Fragment shader does ray-sphere intersection analytically
- Perfect smooth normals: `normal = normalize(hitPoint - sphereCenter)`
- No tessellation artifacts
- GPU-friendly (one quad per cell, ~100 ALU ops per pixel)
- FrontSide rendering gives true directional lighting

### Raymarched Impostor Technique

Instead of rendering a tessellated icosahedron, render a camera-facing quad and analytically compute the sphere intersection in the fragment shader:

```glsl
// Vertex shader: billboard quad
vec3 camRight = vec3(viewMatrix[0][0], viewMatrix[1][0], viewMatrix[2][0]);
vec3 camUp = vec3(viewMatrix[0][1], viewMatrix[1][1], viewMatrix[2][1]);
vec3 worldPos = sphereCenter + camRight * position.x * radius + camUp * position.y * radius;

// Fragment shader: ray-sphere intersection
vec3 oc = rayOrigin - sphereCenter;
float b = dot(oc, rayDir);
float c = dot(oc, oc) - radius * radius;
float h = b * b - c;
if (h < 0.0) discard;  // Ray misses sphere

float t = -b - sqrt(h);  // Front intersection
vec3 hitPoint = rayOrigin + t * rayDir;
vec3 normal = normalize(hitPoint - sphereCenter);  // Perfect analytical normal

// Diffuse lighting
float diffuse = max(0.0, dot(normal, lightDir));
```

**Advantages over tessellated geometry**:
- Perfect smooth normals at any resolution
- No faceting artifacts
- Single quad per cell (4 vertices vs 80+ for icosahedron)
- Can apply noise displacement to the sphere analytically

### Performance Characteristics

| Approach | Draw Calls | Vertices per Cell | Fragment Complexity |
|----------|------------|-------------------|---------------------|
| Current (5 nested icosahedrons) | 5 per cell | 400+ | ~50 ALU |
| Raymarched impostor | 1 per cell | 4 | ~100 ALU |
| Single-plane metaball | 1 per layer | 4 | ~150 ALU |

Raymarched impostors have slightly higher per-pixel cost but far fewer vertices and draw calls. Net performance is better.

### What We Explicitly Avoided

1. **Higher tessellation** (IcosahedronGeometry level 4+): Smooths facets but doesn't fix the additive blending problem
2. **Order-independent transparency**: Complex, not needed if we use the right approach per field type
3. **Manual depth sorting**: Expensive, fragile
4. **Keeping nested shells with different blending**: Still can't show directional lighting

### Visual Goals

After implementation:
- Atmospheric charge merges like ceiling/ground (wavy organic contours, cells connect)
- Moisture spheres show true 3D shading (one side bright, one side dark)
- Ionization spheres same as moisture (smaller, brighter)
- No concentric rings from nested geometry
- No faceted/low-poly appearance
