# Lightning Rendering & Performance Plan

## Executive Summary

The current rendering implementation has several critical performance issues:
- Geometry/material recreation every frame during animation phases
- `LineBasicMaterial.linewidth` is ignored in WebGL (bolts appear as 1px wireframes)
- Unbounded Map growth in AtmosphericField
- PointLight per flash (expensive shadow calculations)
- DOM manipulation for screen flash effects
- No object pooling or buffer reuse

This plan addresses each issue with specific, implementable solutions.

---

## 1. Geometry Strategy

### Current Problems
```typescript
// LeaderRenderer.render() - called every frame during SEARCHING phase
this.clear();  // Disposes all geometries
const geometry = new THREE.BufferGeometry().setFromPoints(points);  // Creates new geometry
const material = this.material.clone();  // Clones material
```

### Solution: Pre-allocated BufferGeometry with Reveal Animation

**Architecture:**
1. Simulation engine produces complete `BoltGeometry` BEFORE rendering
2. Single BufferGeometry allocated with maximum capacity
3. Animation controlled via shader uniforms, not geometry recreation

**Implementation:**

```typescript
interface BoltGeometry {
  // Flat arrays for direct GPU upload
  positions: Float32Array;      // 6 floats per segment (start xyz, end xyz)
  intensities: Float32Array;    // 1 float per segment
  depths: Uint8Array;           // 1 byte per segment (0-3 branch depth)
  segmentCount: number;

  // Metadata for animation
  segmentOrder: Uint16Array;    // Order segments were created (for reveal)
  mainChannelMask: Uint8Array;  // 1 = main channel, 0 = branch
}
```

**BufferGeometry Setup (once per bolt):**

```typescript
class LightningGeometry {
  private geometry: THREE.BufferGeometry;
  private maxSegments: number;

  constructor(maxSegments: number = 256) {
    this.maxSegments = maxSegments;
    this.geometry = new THREE.BufferGeometry();

    // Pre-allocate all buffers
    const positions = new Float32Array(maxSegments * 6);  // 2 points * 3 coords
    const intensities = new Float32Array(maxSegments);
    const depths = new Float32Array(maxSegments);
    const segmentOrder = new Float32Array(maxSegments);

    this.geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    this.geometry.setAttribute('intensity', new THREE.BufferAttribute(intensities, 1));
    this.geometry.setAttribute('depth', new THREE.BufferAttribute(depths, 1));
    this.geometry.setAttribute('order', new THREE.BufferAttribute(segmentOrder, 1));

    // Mark as dynamic for efficient updates
    this.geometry.attributes.position.setUsage(THREE.DynamicDrawUsage);
  }

  updateFromBolt(bolt: BoltGeometry): void {
    const posAttr = this.geometry.attributes.position as THREE.BufferAttribute;
    posAttr.array.set(bolt.positions);
    posAttr.needsUpdate = true;

    // Update draw range instead of recreating geometry
    this.geometry.setDrawRange(0, bolt.segmentCount * 2);
  }
}
```

**Reveal Animation via Uniform:**

The vertex shader discards segments beyond the current reveal progress:

```glsl
uniform float uRevealProgress;  // 0.0 to 1.0
uniform float uTotalSegments;
attribute float order;

void main() {
  // Segments with order > revealProgress * totalSegments are invisible
  float segmentProgress = order / uTotalSegments;
  if (segmentProgress > uRevealProgress) {
    // Move vertex off-screen (cheaper than discard in fragment)
    gl_Position = vec4(9999.0, 9999.0, 9999.0, 1.0);
    return;
  }

  // Normal positioning
  gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}
```

**Animation Loop:**
```typescript
useFrame((_, delta) => {
  if (phase === 'SEARCHING') {
    revealProgress.current = Math.min(1.0, revealProgress.current + delta * revealSpeed);
    material.uniforms.uRevealProgress.value = revealProgress.current;
    // NO geometry recreation, just uniform update
  }
});
```

---

## 2. Line Rendering with Actual Width

### Current Problem

WebGL ignores `LineBasicMaterial.linewidth` on most platforms. All lines render as 1 pixel regardless of setting.

### Solution: drei's `<Line>` Component (uses THREE.Line2)

**Why Line2:**
- GPU-extruded lines with actual width in screen pixels
- Built into drei, no additional dependencies
- Supports per-vertex colors and widths
- Efficient for our segment count (~200 segments max)

**Not MeshLine:** While powerful, MeshLine is more suited for smooth continuous curves. Our lightning has many disconnected segments (branches), making LineSegments2 a better fit.

**Implementation for Leader Phase:**

```tsx
import { Line } from '@react-three/drei';

interface LightningLineProps {
  points: THREE.Vector3[];
  widths: number[];
  colors: THREE.Color[];
  opacity: number;
}

const LightningLine: React.FC<LightningLineProps> = ({ points, widths, colors, opacity }) => {
  return (
    <Line
      points={points}
      lineWidth={2}
      vertexColors={colors}
      transparent
      opacity={opacity}
      // Line2 supports per-vertex width via linewidth attribute
    />
  );
};
```

**For LineSegments (disconnected segments):**

drei's Line expects a continuous path. For our branching structure with disconnected segments, we need LineSegments2:

```typescript
import { LineSegments2 } from 'three/examples/jsm/lines/LineSegments2.js';
import { LineSegmentsGeometry } from 'three/examples/jsm/lines/LineSegmentsGeometry.js';
import { LineMaterial } from 'three/examples/jsm/lines/LineMaterial.js';

class LightningLineRenderer {
  private line: LineSegments2;
  private geometry: LineSegmentsGeometry;
  private material: LineMaterial;

  constructor() {
    this.geometry = new LineSegmentsGeometry();

    this.material = new LineMaterial({
      color: 0xaaaaff,
      linewidth: 3,        // Screen pixels
      transparent: true,
      opacity: 0.8,
      vertexColors: true,
      resolution: new THREE.Vector2(window.innerWidth, window.innerHeight),
      worldUnits: false    // Use screen pixels, not world units
    });

    this.line = new LineSegments2(this.geometry, this.material);
  }

  updateFromBolt(bolt: BoltGeometry): void {
    // LineSegmentsGeometry.setPositions expects pairs of points
    this.geometry.setPositions(bolt.positions);

    // Per-segment colors based on depth
    const colors = new Float32Array(bolt.segmentCount * 6);  // RGB per vertex
    for (let i = 0; i < bolt.segmentCount; i++) {
      const depth = bolt.depths[i];
      const brightness = 0.9 - depth * 0.15;
      // Start vertex
      colors[i * 6 + 0] = brightness * 0.7;
      colors[i * 6 + 1] = brightness * 0.7;
      colors[i * 6 + 2] = brightness * 1.0;
      // End vertex
      colors[i * 6 + 3] = brightness * 0.7;
      colors[i * 6 + 4] = brightness * 0.7;
      colors[i * 6 + 5] = brightness * 1.0;
    }
    this.geometry.setColors(colors);
  }

  updateResolution(width: number, height: number): void {
    this.material.resolution.set(width, height);
  }
}
```

**Width Variation (thicker main channel, thinner branches):**

LineSegments2 doesn't support per-segment linewidth. Options:
1. **Multiple LineSegments2 objects** grouped by width tier (main=4px, depth1=3px, depth2=2px, depth3=1px)
2. **Custom shader** that reads width from vertex attribute

Recommendation: Use option 1 for simplicity. With max 4 depth levels, that's 4 draw calls per bolt - acceptable.

```typescript
class TieredLightningRenderer {
  private tiers: Map<number, LineSegments2> = new Map();
  private widthByDepth = [4, 3, 2, 1];  // Main channel = 4px

  updateFromBolt(bolt: BoltGeometry): void {
    // Group segments by depth
    const segmentsByDepth: Map<number, number[]> = new Map();

    for (let i = 0; i < bolt.segmentCount; i++) {
      const depth = bolt.depths[i];
      if (!segmentsByDepth.has(depth)) {
        segmentsByDepth.set(depth, []);
      }
      segmentsByDepth.get(depth)!.push(i);
    }

    // Update each tier's geometry
    for (const [depth, indices] of segmentsByDepth) {
      const tier = this.getOrCreateTier(depth);
      const positions = new Float32Array(indices.length * 6);

      for (let j = 0; j < indices.length; j++) {
        const srcOffset = indices[j] * 6;
        const dstOffset = j * 6;
        positions.set(bolt.positions.subarray(srcOffset, srcOffset + 6), dstOffset);
      }

      tier.geometry.setPositions(positions);
    }
  }
}
```

---

## 3. Glow and Bloom - Done Right

### Previous Problem

Bloom was applied globally, affecting the entire scene including the globe. This caused:
- Performance issues (full-screen post-processing is expensive)
- Visual artifacts on globe textures

### Solution: Selective Bloom with Layers

**Three.js Layers System:**

Objects can be assigned to layers (0-31). The camera renders all layers by default, but we can configure post-processing to only affect specific layers.

```typescript
// Layer assignments
const LAYER_DEFAULT = 0;
const LAYER_BLOOM = 1;

// Lightning bolts go on bloom layer
lightningMesh.layers.enable(LAYER_BLOOM);

// Globe stays on default layer only
globe.layers.set(LAYER_DEFAULT);
```

**Selective Bloom Implementation (Showcase):**

Using @react-three/postprocessing with selective bloom:

```tsx
import { EffectComposer, Bloom, SelectiveBloom } from '@react-three/postprocessing';
import { Selection, Select } from '@react-three/postprocessing';

const ShowcaseEffects: React.FC<{ lightningRef: React.RefObject<THREE.Group> }> = ({ lightningRef }) => {
  return (
    <EffectComposer>
      <SelectiveBloom
        selection={lightningRef}
        intensity={1.5}
        luminanceThreshold={0.2}
        luminanceSmoothing={0.9}
        mipmapBlur
        radius={0.8}
      />
    </EffectComposer>
  );
};

// Or using Selection API
const ShowcaseScene: React.FC = () => {
  return (
    <Selection>
      <EffectComposer>
        <Bloom
          intensity={1.2}
          luminanceThreshold={0.1}
          luminanceSmoothing={0.9}
          mipmapBlur
        />
      </EffectComposer>

      {/* Non-bloomed elements */}
      <GroundPlane />

      {/* Bloomed lightning */}
      <Select enabled>
        <LightningBolt />
      </Select>
    </Selection>
  );
};
```

**Performance Settings for 60fps:**

| Setting | Showcase Value | Notes |
|---------|---------------|-------|
| intensity | 1.0-1.5 | Higher = brighter glow |
| luminanceThreshold | 0.1-0.3 | Lower = more glow coverage |
| luminanceSmoothing | 0.9 | Smooth falloff |
| mipmapBlur | true | Much faster than Gaussian |
| radius | 0.8 | Glow spread |
| levels | 5 (default) | Mipmap levels, reduce to 3-4 if needed |

**Globe View: Fake Glow (No Post-Processing)**

For the globe with 10 concurrent bolts, post-processing is too expensive. Use the classic "duplicate line" trick:

```typescript
class GlobeLightningRenderer {
  private mainLine: LineSegments2;
  private glowLine: LineSegments2;

  constructor() {
    // Main bolt line
    this.mainLine = new LineSegments2(
      new LineSegmentsGeometry(),
      new LineMaterial({
        color: 0xffffff,
        linewidth: 2,
        transparent: true,
        opacity: 1.0
      })
    );

    // Glow line: wider, semi-transparent, additive blending
    this.glowLine = new LineSegments2(
      new LineSegmentsGeometry(),
      new LineMaterial({
        color: 0x8888ff,
        linewidth: 6,
        transparent: true,
        opacity: 0.3,
        blending: THREE.AdditiveBlending,
        depthWrite: false
      })
    );

    // Glow renders behind main
    this.glowLine.renderOrder = 999;
    this.mainLine.renderOrder = 1000;
  }

  updateFromBolt(bolt: BoltGeometry): void {
    // Both lines share the same positions
    this.mainLine.geometry.setPositions(bolt.positions);
    this.glowLine.geometry.setPositions(bolt.positions);
  }
}
```

**Additive Blending Notes:**
- Set `depthWrite: false` to prevent glow from occluding other elements
- Glow color should be slightly blue-shifted (0x8888ff) for lightning aesthetic
- Multiple overlapping glows will stack (which looks good for lightning)

---

## 4. Flash Effects - Simplified

### Current Problems

```typescript
// FlashEffect.ts - PointLight is expensive
this.light = new THREE.PointLight(config.color, config.intensity * 100, 50, 2);

// ScreenFlashEffect.ts - DOM manipulation
this.overlay = document.createElement('div');
document.body.appendChild(this.overlay);
```

### Solution: Shader-Based Flash

**Showcase Flash: Emissive Pulse + Ambient Bump**

```typescript
class LightningFlashController {
  private ambientLight: THREE.AmbientLight;
  private baseAmbientIntensity: number = 0.15;

  constructor(scene: THREE.Scene) {
    this.ambientLight = scene.getObjectByName('ambient') as THREE.AmbientLight;
  }

  triggerFlash(intensity: number, duration: number): void {
    const startTime = performance.now();

    const animate = () => {
      const elapsed = (performance.now() - startTime) / 1000;
      const progress = elapsed / duration;

      if (progress >= 1) {
        this.ambientLight.intensity = this.baseAmbientIntensity;
        return;
      }

      // Sharp attack, exponential decay
      const flashCurve = Math.exp(-progress * 5);
      this.ambientLight.intensity = this.baseAmbientIntensity + intensity * flashCurve;

      requestAnimationFrame(animate);
    };

    animate();
  }
}
```

**Bolt Emissive Pulse:**

The bolt material should have an emissive component that pulses during strike:

```glsl
// In lightning line material
uniform float uFlashIntensity;
uniform vec3 uBaseColor;
uniform vec3 uEmissiveColor;

void main() {
  vec3 color = uBaseColor;

  // Add emissive glow during flash
  color += uEmissiveColor * uFlashIntensity;

  gl_FragColor = vec4(color, opacity);
}
```

```typescript
// Animation
useFrame(() => {
  if (phase === 'STRIKING') {
    const flashIntensity = Math.exp(-phaseTime * 4);
    material.uniforms.uFlashIntensity.value = flashIntensity;
  }
});
```

**Screen Flash (Showcase Only): Full-Screen Quad in Three.js**

Replace DOM overlay with a Three.js plane in front of the camera:

```tsx
const ScreenFlash: React.FC<{ intensity: number }> = ({ intensity }) => {
  const meshRef = useRef<THREE.Mesh>(null);

  return (
    <mesh ref={meshRef} position={[0, 0, -1]} renderOrder={9999}>
      <planeGeometry args={[10, 10]} />
      <meshBasicMaterial
        color="white"
        transparent
        opacity={intensity * 0.3}
        depthTest={false}
        depthWrite={false}
      />
    </mesh>
  );
};
```

Better: Use a post-processing effect for screen flash:

```tsx
import { EffectComposer, Vignette } from '@react-three/postprocessing';
import { ToneMappingEffect, BlendFunction } from 'postprocessing';

// Custom brightness effect for flash
const FlashEffect = forwardRef(({ intensity }, ref) => {
  const effect = useMemo(() => {
    return new BrightnessContrastEffect({
      brightness: intensity * 0.5,
      contrast: 0
    });
  }, []);

  useEffect(() => {
    effect.brightness = intensity * 0.5;
  }, [intensity]);

  return <primitive ref={ref} object={effect} />;
});
```

**Ground Illumination:**

Already implemented via shader uniform. Just ensure timing syncs:

```typescript
// In LightningController
const triggerStrike = () => {
  const startTime = performance.now() / 1000;

  // Ground plane listens for this event
  window.dispatchEvent(new CustomEvent('lightning-strike', {
    detail: { startTime, intensity: 1.0 }
  }));
};
```

**Globe View:**

Skip all flash effects. The bolt's additive glow provides sufficient visual feedback. If any flash is needed, briefly increase the bolt's glow line opacity:

```typescript
// During STRIKING phase on globe
glowMaterial.opacity = 0.3 + flashIntensity * 0.4;  // 0.3 to 0.7
```

---

## 5. Material Strategy

### Current Problem

```typescript
// Every frame during render:
const material = this.material.clone();  // New material allocation
material.opacity = 0.8 - depth * 0.2;     // Modified then discarded
```

### Solution: Reusable Materials with Uniform Animation

**Material Pool:**

Create a fixed set of materials at initialization:

```typescript
class LightningMaterialPool {
  // LineMaterial for Line2/LineSegments2
  readonly mainChannel: LineMaterial;
  readonly branch1: LineMaterial;
  readonly branch2: LineMaterial;
  readonly branch3: LineMaterial;
  readonly glow: LineMaterial;

  // Shared uniforms (optional: use custom ShaderMaterial for more control)
  private uniforms = {
    uRevealProgress: { value: 0.0 },
    uFlashIntensity: { value: 0.0 },
    uOpacity: { value: 1.0 },
    uTime: { value: 0.0 }
  };

  constructor() {
    const baseConfig = {
      transparent: true,
      vertexColors: true,
      worldUnits: false
    };

    this.mainChannel = new LineMaterial({
      ...baseConfig,
      color: 0xffffff,
      linewidth: 4,
      opacity: 1.0
    });

    this.branch1 = new LineMaterial({
      ...baseConfig,
      color: 0xccccff,
      linewidth: 3,
      opacity: 0.8
    });

    this.branch2 = new LineMaterial({
      ...baseConfig,
      color: 0xaaaaee,
      linewidth: 2,
      opacity: 0.6
    });

    this.branch3 = new LineMaterial({
      ...baseConfig,
      color: 0x8888dd,
      linewidth: 1.5,
      opacity: 0.4
    });

    this.glow = new LineMaterial({
      ...baseConfig,
      color: 0x6688ff,
      linewidth: 8,
      opacity: 0.25,
      blending: THREE.AdditiveBlending,
      depthWrite: false
    });
  }

  getMaterialForDepth(depth: number): LineMaterial {
    switch (depth) {
      case 0: return this.mainChannel;
      case 1: return this.branch1;
      case 2: return this.branch2;
      default: return this.branch3;
    }
  }

  updateUniforms(revealProgress: number, flashIntensity: number, opacity: number): void {
    // Update all materials if using custom uniforms
    this.uniforms.uRevealProgress.value = revealProgress;
    this.uniforms.uFlashIntensity.value = flashIntensity;
    this.uniforms.uOpacity.value = opacity;
  }

  updateResolution(width: number, height: number): void {
    // LineMaterial needs screen resolution for width calculation
    const resolution = new THREE.Vector2(width, height);
    this.mainChannel.resolution = resolution;
    this.branch1.resolution = resolution;
    this.branch2.resolution = resolution;
    this.branch3.resolution = resolution;
    this.glow.resolution = resolution;
  }

  dispose(): void {
    this.mainChannel.dispose();
    this.branch1.dispose();
    this.branch2.dispose();
    this.branch3.dispose();
    this.glow.dispose();
  }
}
```

**Custom ShaderMaterial (for advanced reveal animation):**

If LineMaterial's built-in features aren't sufficient, use a custom shader:

```typescript
const lightningShader = {
  uniforms: {
    uRevealProgress: { value: 0.0 },
    uFlashIntensity: { value: 0.0 },
    uOpacity: { value: 1.0 },
    uColor: { value: new THREE.Color(0xaaaaff) },
    uEmissive: { value: new THREE.Color(0x4444ff) }
  },

  vertexShader: `
    attribute float segmentOrder;
    attribute float intensity;

    uniform float uRevealProgress;
    uniform float uTotalSegments;

    varying float vIntensity;
    varying float vVisible;

    void main() {
      // Reveal animation
      float threshold = uRevealProgress * uTotalSegments;
      vVisible = step(segmentOrder, threshold);

      vIntensity = intensity;

      vec3 pos = position;
      if (vVisible < 0.5) {
        pos = vec3(9999.0);  // Off-screen
      }

      gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }
  `,

  fragmentShader: `
    uniform float uFlashIntensity;
    uniform float uOpacity;
    uniform vec3 uColor;
    uniform vec3 uEmissive;

    varying float vIntensity;
    varying float vVisible;

    void main() {
      if (vVisible < 0.5) discard;

      vec3 color = uColor * vIntensity;
      color += uEmissive * uFlashIntensity;

      gl_FragColor = vec4(color, uOpacity);
    }
  `
};
```

---

## 6. Memory and Lifecycle Management

### Current Problems

```typescript
// AtmosphericField - unbounded Map growth
private field: Map<string, ElectricField> = new Map();

// Never cleared during simulation
this.field.set(key, { ... });
```

### Solution: Object Pooling and Fixed Allocations

**Geometry Pool:**

```typescript
class GeometryPool {
  private available: LightningGeometry[] = [];
  private inUse: Set<LightningGeometry> = new Set();
  private maxPoolSize: number;

  constructor(maxPoolSize: number = 15) {
    this.maxPoolSize = maxPoolSize;

    // Pre-allocate pool
    for (let i = 0; i < maxPoolSize; i++) {
      this.available.push(new LightningGeometry(256));
    }
  }

  acquire(): LightningGeometry | null {
    const geometry = this.available.pop();
    if (geometry) {
      this.inUse.add(geometry);
      return geometry;
    }

    // Pool exhausted - either wait or create overflow (not recommended)
    console.warn('Geometry pool exhausted');
    return null;
  }

  release(geometry: LightningGeometry): void {
    if (this.inUse.has(geometry)) {
      this.inUse.delete(geometry);
      geometry.reset();  // Clear buffers
      this.available.push(geometry);
    }
  }

  dispose(): void {
    for (const geom of this.available) {
      geom.dispose();
    }
    for (const geom of this.inUse) {
      geom.dispose();
    }
    this.available = [];
    this.inUse.clear();
  }
}
```

**AtmosphericField Fix:**

Replace unbounded Map with fixed-size cache or LRU:

```typescript
class AtmosphericField {
  // Option 1: Fixed-size circular buffer cache
  private cacheKeys: string[] = new Array(1024);
  private cacheValues: ElectricField[] = new Array(1024);
  private cacheIndex: number = 0;
  private cacheMap: Map<string, number> = new Map();  // key -> index

  getField(x: number, y: number, z: number): ElectricField {
    const key = this.hash(x, y, z);

    const cachedIndex = this.cacheMap.get(key);
    if (cachedIndex !== undefined) {
      return this.cacheValues[cachedIndex];
    }

    // Calculate new field
    const field = this.calculateField(x, y, z);

    // Store in circular buffer (overwrites oldest entry)
    const oldKey = this.cacheKeys[this.cacheIndex];
    if (oldKey) {
      this.cacheMap.delete(oldKey);
    }

    this.cacheKeys[this.cacheIndex] = key;
    this.cacheValues[this.cacheIndex] = field;
    this.cacheMap.set(key, this.cacheIndex);

    this.cacheIndex = (this.cacheIndex + 1) % 1024;

    return field;
  }

  // Option 2: Reset between bolts
  reset(): void {
    this.cacheMap.clear();
    this.cacheIndex = 0;
  }
}
```

**Proper Disposal Sequence:**

```typescript
class LightningBoltRenderer {
  dispose(): void {
    // 1. Remove from scene first
    if (this.group.parent) {
      this.group.parent.remove(this.group);
    }

    // 2. Dispose geometries
    this.group.traverse((child) => {
      if (child instanceof THREE.Mesh || child instanceof LineSegments2) {
        child.geometry.dispose();
      }
    });

    // 3. Return materials to pool (don't dispose if pooled)
    // Or dispose if not pooled:
    // materialPool.release(this.material);

    // 4. Return geometry to pool
    if (this.pooledGeometry) {
      geometryPool.release(this.pooledGeometry);
    }

    // 5. Clear references
    this.group.clear();
  }
}
```

---

## 7. Performance Budgets

### Frame Time Breakdown

**60fps = 16.67ms per frame**

| Component | Globe Budget | Showcase Budget |
|-----------|-------------|-----------------|
| JavaScript/React | 2ms | 2ms |
| Physics/Simulation | 1ms | 1ms |
| Geometry Updates | 1ms | 1ms |
| Draw Calls (lines) | 2ms | 1ms |
| Post-Processing | 0ms | 4ms |
| React-Globe.gl | 8ms | N/A |
| Buffer/Slack | 2.67ms | 7.67ms |
| **Total** | 16.67ms | 16.67ms |

### Globe View: 10 Concurrent Bolts

**Draw call budget:** 2ms for 10 bolts = 0.2ms per bolt

Each bolt uses:
- 4 LineSegments2 objects (one per depth tier)
- 1 glow LineSegments2 object
- Total: 5 draw calls per bolt, 50 draw calls for 10 bolts

**Optimization if over budget:**
1. Reduce to 2 tiers (main + branches combined)
2. Skip glow on distant bolts
3. Lower segment count per bolt

```typescript
class AdaptiveLightningRenderer {
  private qualityLevel: 'high' | 'medium' | 'low' = 'high';

  adjustQuality(frameTime: number): void {
    if (frameTime > 20) {
      this.qualityLevel = 'low';
    } else if (frameTime > 16) {
      this.qualityLevel = 'medium';
    } else {
      this.qualityLevel = 'high';
    }
  }

  getConfig() {
    switch (this.qualityLevel) {
      case 'high':
        return { tiers: 4, glow: true, maxSegments: 256 };
      case 'medium':
        return { tiers: 2, glow: true, maxSegments: 150 };
      case 'low':
        return { tiers: 1, glow: false, maxSegments: 80 };
    }
  }
}
```

### Showcase View: Full Effects

With only 1 bolt and no globe rendering:
- Full 4-tier rendering with glow
- Bloom post-processing enabled
- Screen flash effect

**Bloom Performance Tips:**
- `mipmapBlur: true` is essential (10x faster than Gaussian)
- Reduce `levels` from 5 to 3 if needed
- Use `Selection` component to exclude non-glowing objects

---

## 8. Integration with React-Three-Fiber

### Declarative vs Imperative

**Declarative (preferred for static elements):**
```tsx
<mesh position={[0, 0, 0]}>
  <boxGeometry />
  <meshStandardMaterial />
</mesh>
```

**Imperative (required for animation):**
```tsx
const LightningBolt: React.FC = () => {
  const lineRef = useRef<LineSegments2>(null);
  const materialRef = useRef<LineMaterial>(null);

  // Imperative updates in useFrame - no React re-renders
  useFrame(() => {
    if (materialRef.current) {
      materialRef.current.opacity = calculateOpacity();
    }
  });

  return (
    <primitive object={lineSegments2} ref={lineRef} />
  );
};
```

### Avoiding React Re-renders

**Bad: State-driven animation**
```tsx
// Triggers re-render 60 times per second
const [opacity, setOpacity] = useState(1);
useFrame(() => setOpacity(prev => prev * 0.99));
```

**Good: Ref-driven animation**
```tsx
const opacityRef = useRef(1);
const materialRef = useRef<Material>(null);

useFrame(() => {
  opacityRef.current *= 0.99;
  if (materialRef.current) {
    materialRef.current.opacity = opacityRef.current;
  }
});
```

### Component Structure

```
ShowcasePage
├── Canvas
│   ├── Scene
│   │   ├── ambientLight
│   │   ├── GroundPlane (declarative mesh with shader material)
│   │   ├── LightningController (imperative, manages bolt lifecycle)
│   │   │   └── LightningBolt (imperative, animates via refs)
│   │   │       ├── LineSegments2 (main channel)
│   │   │       ├── LineSegments2 (branches tier 1)
│   │   │       ├── LineSegments2 (branches tier 2)
│   │   │       ├── LineSegments2 (branches tier 3)
│   │   │       └── LineSegments2 (glow)
│   │   └── OrbitControls
│   └── PostProcessing
│       ├── EffectComposer
│       └── SelectiveBloom (targets LightningBolt)
└── Controls UI (React, outside Canvas)
```

### EffectComposer Sharing

One EffectComposer per Canvas. All effects chain together:

```tsx
<EffectComposer>
  {/* Order matters - effects apply in sequence */}
  <Bloom intensity={1.2} ... />
  <Vignette opacity={0.3} />  {/* Optional atmospheric effect */}
</EffectComposer>
```

### Event Communication

For cross-component coordination (e.g., lightning strike -> ground flash), avoid React state:

**Option 1: Custom events (current approach)**
```typescript
window.dispatchEvent(new CustomEvent('lightning-strike', { detail }));
```

**Option 2: Zustand store (better for complex state)**
```typescript
const useLightningStore = create((set) => ({
  activeStrikes: [],
  addStrike: (strike) => set((state) => ({
    activeStrikes: [...state.activeStrikes, strike]
  })),
  removeStrike: (id) => set((state) => ({
    activeStrikes: state.activeStrikes.filter(s => s.id !== id)
  }))
}));
```

**Option 3: Context (for simple shared refs)**
```typescript
const LightningContext = createContext<{
  flashIntensityRef: React.MutableRefObject<number>;
}>(null);
```

---

## 9. Implementation Phases

### Phase 1: Core Geometry Refactor
1. Create `BoltGeometry` interface and pre-compute full bolt before rendering
2. Implement `LightningGeometry` with pre-allocated BufferGeometry
3. Replace per-frame geometry creation with buffer updates

### Phase 2: Line Rendering
1. Switch from LineBasicMaterial to LineSegments2/LineMaterial
2. Implement tiered rendering (4 depth levels)
3. Add fake glow (wider additive line) for globe view

### Phase 3: Materials & Animation
1. Create material pool with reusable LineMaterials
2. Implement reveal animation via uniform (not geometry recreation)
3. Replace PointLight flash with ambient bump + emissive pulse

### Phase 4: Post-Processing (Showcase Only)
1. Add EffectComposer with SelectiveBloom
2. Configure bloom settings for 60fps
3. Add optional screen flash via post-processing

### Phase 5: Memory & Polish
1. Implement geometry pool for globe view
2. Fix AtmosphericField Map growth
3. Add adaptive quality system
4. Profile and optimize

---

## 10. Files to Create/Modify

**New Files:**
- `client/src/effects/LightningBoltEffect/rendering/LightningGeometry.ts`
- `client/src/effects/LightningBoltEffect/rendering/LightningMaterialPool.ts`
- `client/src/effects/LightningBoltEffect/rendering/LightningLineRenderer.ts`
- `client/src/effects/LightningBoltEffect/rendering/GeometryPool.ts`
- `client/src/pages/ShowcasePage/PostProcessing.tsx`

**Modified Files:**
- `client/src/effects/LightningBoltEffect/LightningBoltEffect.ts` - use new renderers
- `client/src/effects/LightningBoltEffect/rendering/LeaderRenderer.ts` - rewrite with LineSegments2
- `client/src/effects/LightningBoltEffect/rendering/StrokeRenderer.ts` - rewrite with LineSegments2
- `client/src/effects/LightningBoltEffect/rendering/FlashEffect.ts` - simplify, remove PointLight
- `client/src/effects/LightningBoltEffect/physics/AtmosphericField.ts` - add cache limit
- `client/src/pages/ShowcasePage/Scene.tsx` - add EffectComposer

**Delete Files:**
- None (refactor in place)
