# Plan Revisions, Errata, and Cross-Plan Coordination

This document resolves conflicts, fills gaps, and provides the unified coordination layer across all five planning documents. Read this BEFORE implementing any plan.

---

## 1. Git State (Resolved)

- **Branch**: `bolt-animation-overhaul-3` in `client/` repo
- **Status**: Stuck rebase has been cleaned up (orphaned `.git/rebase-merge/` removed)
- **Position**: 1 commit ahead of `main` (commit `f7c6252` - the physics/rendering split WIP)
- **Working tree**: Clean except `StrokeRenderer.ts` has unstaged changes
- **Action**: Continue working on this branch. The WIP commit's structure (physics/ + rendering/ separation) aligns with where we're going, but the code inside those files will be replaced.
- **Server repo**: Separate git repo at `server/`, on `main` branch, clean state.

---

## 2. Unified File Manifest

The simulation and rendering plans each propose file changes that overlap. Here is the **single source of truth** for the final file structure:

```
client/src/effects/LightningBoltEffect/
  index.ts                          MODIFY (update exports)
  LightningBoltEffect.ts            REWRITE (thin orchestrator)
  LightningTypes.ts                 MODIFY (add DetailLevel, update config interface)

  simulation/                       NEW DIRECTORY
    index.ts                        NEW
    BoltSimulator.ts                NEW (main entry: simulateBolt())
    GrowthStep.ts                   NEW (single step logic)
    FieldComputation.ts             NEW (distance-based field approx)
    BranchSelection.ts              NEW (DBM probability + branching)
    types.ts                        NEW (BoltGeometry, SimulationConfig, Vec3, etc.)
    config.ts                       NEW (detail presets, default params)
    prng.ts                         NEW (mulberry32 SeededRNG)
    noise.ts                        NEW (seeded simplex/value noise - see Section 3)
    spatial.ts                      NEW (SpatialGrid for showcase perf)

  animation/                        NEW DIRECTORY
    index.ts                        NEW
    BoltTimeline.ts                 NEW (timeline structure + creation)
    BoltAnimator.ts                 NEW (state machine - see Section 4)
    types.ts                        NEW (AnimationState, AnimationPhase)

  rendering/                        MODIFY DIRECTORY
    index.ts                        MODIFY (update exports)
    BoltRenderer.ts                 NEW (unified renderer consuming AnimationState)
    LightningMaterials.ts           NEW (material pool, see rendering plan Section 5)
    FlashEffect.ts                  MODIFY (remove PointLight, use ambient bump)
    LeaderRenderer.ts               DELETE (after BoltRenderer works)
    StrokeRenderer.ts               DELETE (after BoltRenderer works)

  physics/                          DEPRECATED DIRECTORY
    index.ts                        MODIFY (re-export from simulation/ for compat during transition)
    SteppedLeader.ts                DELETE (after simulation/ works)
    AtmosphericField.ts             DELETE (after simulation/ works)
    ReturnStroke.ts                 DELETE (after simulation/ works)
```

**Showcase page changes:**
```
client/src/pages/ShowcasePage/
  ShowcasePage.tsx                   REWRITE (new layout with MathPanel)
  ShowcasePage.css                   REWRITE (new layout CSS)
  Scene.tsx                          MODIFY (add EffectComposer, read from context)
  LightningController.tsx            REWRITE (use simulateBolt + BoltAnimator)
  GroundPlane.tsx                    MODIFY (sync timing via context ref, not events)
  CloudGrid.tsx                      KEEP
  MathPanel.tsx                      NEW
  MathPanel.css                      NEW

client/src/components/
  math/
    FormulaDisplay.tsx               NEW
    FormulaSection.tsx               NEW
    index.ts                         NEW
  controls/
    ParameterSlider.tsx              NEW
    index.ts                         NEW

client/src/context/
  LightningContext.tsx               NEW

client/src/config/
  formulas.ts                        NEW
  definitions/lightning.ts           MODIFY (add simulation params)
```

**Server changes:**
```
server/
  server.js                          MODIFY (use new client, add health endpoint)
  blitzortung_client.js              NEW (direct WebSocket, replaces lightning_data.js)
  strike_processor.js                NEW (raw -> processed transform)
  lightning_data.js                  DELETE (after blitzortung_client works)
  utils/index.js                     KEEP (decode function still needed)
  package.json                       MODIFY (remove puppeteer)
```

**Note**: Server stays JavaScript (not TypeScript) to avoid adding a build step to a simple Node relay server.

---

## 3. Missing: Seeded Noise Implementation (simulation/noise.ts)

The simulation plan references `sampleNoise3D` but never defines it. The `AtmosphericField` currently uses Perlin noise with an unseeded permutation table. Here's what `noise.ts` must provide:

```typescript
// simulation/noise.ts

/**
 * Seeded 3D value noise for atmospheric field variations.
 *
 * Uses the simulation's SeededRNG to generate a deterministic
 * permutation table, unlike the current AtmosphericField which
 * uses Math.random().
 *
 * Interface:
 *   createNoise3D(seed: number): (x: number, y: number, z: number) => number
 *
 * Returns values in [-1, 1] range.
 * Uses improved Perlin noise with seeded permutation via the same
 * mulberry32 PRNG from prng.ts.
 *
 * Performance: ~0.001ms per sample (negligible).
 */
```

Key requirement: The permutation table MUST be generated from the seed, not `Math.random()`. The current code's bug (line 75 of AtmosphericField.ts: `Math.floor(Math.random() * (i + 1))`) is what breaks determinism.

---

## 4. Missing: Complete BoltAnimator State Methods

The simulation plan's `BoltAnimator` has `// ... other state methods follow similar pattern` which is insufficient for implementation. The missing methods are:

- `connectionPauseState()`: All leader segments visible at low brightness (0.3), main channel slightly brighter (0.5). This is the brief moment before the return stroke.
- `strokeHoldState(strokeIndex)`: All segments visible, main channel at brightness decaying from 1.0 with `Math.exp(-holdTime * 2)`, branches at `0.2 * exp(-depth * 0.5)`.
- `fadingState(fadeProgress, strokeIndex)`: Multiply all segment brightnesses by `(1 - fadeProgress)`. Main channel fades slower (multiply by `(1 - fadeProgress * 0.7)`).
- `interstrokeState(strokeIndex)`: All segments at minimal brightness (0.05-0.1). The channel is "warm" and about to re-illuminate.
- `completeState()`: Empty visible set, all brightness 0. Signals the effect to terminate.

Each subsequent stroke (strokeIndex > 0) should use slightly lower peak brightness: `peakBrightness = 1.0 * Math.pow(0.8, strokeIndex)` since subsequent strokes are typically dimmer than the first.

---

## 5. Missing: Geometry Serialization Bridge

The simulation outputs `BoltSegment[]` with `{start: Vec3, end: Vec3}` objects.
The renderer needs flat `Float32Array` buffers.

The bridge function (should live in `rendering/BoltRenderer.ts`):

```typescript
function serializeGeometry(
  segments: BoltSegment[],
  worldTransform: WorldTransform
): {
  positions: Float32Array;     // 6 floats per segment (start.xyz, end.xyz)
  depths: Float32Array;        // 1 per segment
  intensities: Float32Array;   // 1 per segment
  stepIndices: Float32Array;   // 1 per segment
  mainChannel: Uint8Array;     // 1 per segment (boolean)
  segmentCount: number;
} {
  const n = segments.length;
  const positions = new Float32Array(n * 6);
  const depths = new Float32Array(n);
  const intensities = new Float32Array(n);
  const stepIndices = new Float32Array(n);
  const mainChannel = new Uint8Array(n);

  for (let i = 0; i < n; i++) {
    const seg = segments[i];
    const ws = toWorldSpace(seg.start, worldTransform);
    const we = toWorldSpace(seg.end, worldTransform);
    positions[i * 6 + 0] = ws.x;
    positions[i * 6 + 1] = ws.y;
    positions[i * 6 + 2] = ws.z;
    positions[i * 6 + 3] = we.x;
    positions[i * 6 + 4] = we.y;
    positions[i * 6 + 5] = we.z;
    depths[i] = seg.depth;
    intensities[i] = seg.intensity;
    stepIndices[i] = seg.stepIndex;
    mainChannel[i] = seg.isMainChannel ? 1 : 0;
  }

  return { positions, depths, intensities, stepIndices, mainChannel, segmentCount: n };
}
```

---

## 6. Cross-Plan Conflicts Resolved

### 6.1 LeaderRenderer + StrokeRenderer: Rewrite vs Delete?

**Resolution**: Neither rewrite nor keep. Create `BoltRenderer.ts` as the single unified renderer. Delete `LeaderRenderer.ts` and `StrokeRenderer.ts` once `BoltRenderer` is working. During transition, keep old files but don't modify them.

### 6.2 LineMaterial resolution update on window resize

The rendering plan mentions this but doesn't show the integration point. Add this to `BoltRenderer`:

```typescript
// In BoltRenderer constructor or init
const handleResize = () => {
  const width = window.innerWidth;
  const height = window.innerHeight;
  this.materials.updateResolution(width, height);
};
window.addEventListener('resize', handleResize);
// Store for cleanup in dispose()
```

Also in the R3F showcase, use the `useThree` hook's `size` to update:
```typescript
const { size } = useThree();
useEffect(() => {
  materials.updateResolution(size.width, size.height);
}, [size.width, size.height]);
```

### 6.3 Event-based vs Context-based communication (Showcase)

The showcase currently uses `window.dispatchEvent('lightning-strike')` for ground glow sync. The UI plan introduces `LightningContext`.

**Resolution**: Use BOTH, for different purposes:
- **LightningContext**: For parameter state (eta, speed, detail, etc.) and `triggerNewBolt()` callback. React-level state management.
- **Refs for animation state**: The ground plane reads a shared ref (via context) for flash intensity, not events. This avoids the mixed `Date.now()` / `performance.now()` timing issue.

```typescript
// In LightningContext
flashIntensityRef: React.MutableRefObject<number>;  // 0-1, updated by BoltAnimator
```

The `GroundPlane` reads `flashIntensityRef.current` every frame in `useFrame`, eliminating the custom event dispatch entirely.

### 6.4 Coordinate Transform Ownership

**Simulation**: Works in normalized space `[-0.5, 0.5]` on Y axis.
**Showcase**: Uses `mockGlobeEl.getCoords()` to map to scene space.
**Globe**: Uses `globeEl.getCoords()` for spherical mapping.

**Resolution**: The coordinate transform lives in `LightningBoltEffect.ts` (for globe) and `LightningController.tsx` (for showcase). The simulation never knows about world coordinates. The `BoltRenderer.setGeometry()` receives the world-space transform at initialization time and bakes the positions into the GPU buffers.

---

## 7. Blitzortung Protocol Verification

The data pipeline plan claims `wss://ws1.blitzortung.org:3000/` but this needs verification. The current Puppeteer approach works by intercepting whatever WebSocket the Blitzortung page connects to. Before implementing the direct WebSocket:

1. Start the current server with `--verbose` to log the actual WebSocket URL being intercepted
2. Check if the connection requires any cookies/headers set by the page load
3. Test a direct `wscat` connection: `wscat -c wss://ws1.blitzortung.org:3000/` then send `{"time":0}`
4. If direct connection fails, check the Blitzortung page's JavaScript for the connection URL and any required handshake

**If direct WebSocket doesn't work**: Fall back to keeping Puppeteer but optimizing it (reuse browser instance, reduce page load scope). This is the riskiest part of the plan.

---

## 8. Implementation Sequence (Revised)

The plans suggest parallel workstreams but dependencies constrain the order:

### Phase 1: Simulation Engine (Foundation - do first)
1. `simulation/prng.ts` - SeededRNG
2. `simulation/noise.ts` - Seeded Perlin noise
3. `simulation/types.ts` - All data structures
4. `simulation/config.ts` - Detail presets
5. `simulation/FieldComputation.ts` - Distance-based field
6. `simulation/BranchSelection.ts` - DBM probabilities
7. `simulation/GrowthStep.ts` - Single step
8. `simulation/spatial.ts` - SpatialGrid
9. `simulation/BoltSimulator.ts` - Main entry
10. `simulation/index.ts` - Exports

**Validation**: Write a quick test script that runs `simulateBolt()` and logs the output geometry. Verify segment count, branch structure, and determinism (same seed = same output).

### Phase 2: Animation Layer
1. `animation/types.ts`
2. `animation/BoltTimeline.ts`
3. `animation/BoltAnimator.ts` (all state methods - see Section 4)
4. `animation/index.ts`

### Phase 3: Rendering
1. `rendering/LightningMaterials.ts` - Material pool
2. `rendering/BoltRenderer.ts` - Unified renderer with serialization bridge
3. Modify `rendering/FlashEffect.ts` - Simplify
4. Modify `rendering/index.ts`

### Phase 4: Integration (Showcase)
1. Modify `LightningBoltEffect.ts` - Wire simulation + animation + renderer
2. Rewrite `LightningController.tsx` - Use new system
3. Test in showcase: does a bolt appear, animate correctly, and clean up?
4. Add glow line (fake bloom for now)

### Phase 5: Showcase UI
1. Install KaTeX: `npm install katex react-katex`
2. Create `context/LightningContext.tsx`
3. Create `config/formulas.ts`
4. Create `components/math/FormulaDisplay.tsx`
5. Create `components/controls/ParameterSlider.tsx`
6. Create `ShowcasePage/MathPanel.tsx` + CSS
7. Rewrite `ShowcasePage.tsx` with new layout
8. Wire parameters to simulation config

### Phase 6: Post-Processing (Showcase)
1. Install postprocessing: `npm install @react-three/postprocessing postprocessing`
2. Add `EffectComposer` + `Bloom` to Scene
3. Configure selective bloom
4. Test performance, adjust settings

### Phase 7: Globe Integration
1. Modify `LightningLayer.ts` to use new simulation (GLOBE detail)
2. Add fake glow (additive wider line) instead of bloom
3. Test with WebSocket data (or demo mode)

### Phase 8: Server Improvements
1. Test current Blitzortung connectivity (may still work as-is)
2. If working: extract enhanced fields, update strike model
3. Attempt direct WebSocket (verify protocol first)
4. Add demo mode to client as fallback
5. If direct WS works: remove Puppeteer, update package.json

---

## 9. KaTeX Formula Highlight Fix

The showcase UI plan's highlight regex is fragile:
```typescript
const regex = new RegExp(`\\\\${varName}(?![a-zA-Z])`, 'g');
```

This only matches `\eta` at word boundaries. Better approach: pre-process the LaTeX string by wrapping specific variable tokens in `\textcolor` commands before passing to KaTeX. Use explicit per-formula variable position mappings rather than regex:

```typescript
// In formulas.ts, each variable specifies its exact LaTeX token
variables: [
  {
    name: 'eta',
    latexToken: '\\eta',  // Exact string to find and wrap
    description: 'Branching exponent',
    linkedParameter: 'eta'
  },
]

// Highlight function
function highlightLatex(latex: string, activeTokens: string[]): string {
  let result = latex;
  for (const token of activeTokens) {
    // Replace exact token with colored version
    result = result.split(token).join(`\\textcolor{#4fd1c5}{${token}}`);
  }
  return result;
}
```

---

## 10. Performance Safety Rails

To prevent "breaking the computer" again with bloom:

1. **Frame time monitoring**: Add a frame time tracker in the showcase. If 3 consecutive frames exceed 30ms, automatically reduce bloom quality or disable it.

```typescript
// In Scene.tsx
const frameTimesRef = useRef<number[]>([]);
useFrame((_, delta) => {
  frameTimesRef.current.push(delta * 1000);
  if (frameTimesRef.current.length > 10) frameTimesRef.current.shift();

  const avg = frameTimesRef.current.reduce((a, b) => a + b) / frameTimesRef.current.length;
  if (avg > 30 && bloomEnabled) {
    setBloomEnabled(false);
    console.warn('Bloom disabled: frame time exceeded 30ms');
  }
});
```

2. **Bloom defaults**: Start with `mipmapBlur: true`, `intensity: 1.0`, `levels: 3` (not the default 5). Only increase if frame budget allows.

3. **No bloom on globe**: The rendering plan is clear on this. Use ONLY the fake glow (additive wider line). No post-processing on the globe page at all.

---

## 11. What's NOT In Scope (Explicit Boundaries)

To prevent scope creep during implementation:

- Cloud-to-cloud lightning (altitude-based bolt type) - future work
- Positive lightning visual differences - future work
- Database persistence of strikes - future work
- Region filtering UI - future work
- Mobile-specific layout beyond basic responsive - future work
- Web Worker for simulation - future work (sync is fast enough for now)
- Thunder audio - not planned
- Multiple simultaneous showcase bolts - single bolt only in showcase
