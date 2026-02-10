# Lightning Simulation Engine Implementation Plan

A detailed technical plan for replacing the current naive stepped leader implementation with a physics-based Dielectric Breakdown Model (DBM) simulation engine.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Simulation Module Architecture](#2-simulation-module-architecture)
3. [Core Algorithm: Field-Biased DBM](#3-core-algorithm-field-biased-dbm)
4. [Data Structures](#4-data-structures)
5. [Detail Level Scaling](#5-detail-level-scaling)
6. [Animation Timeline](#6-animation-timeline)
7. [Integration Points](#7-integration-points)
8. [File Organization](#8-file-organization)
9. [Implementation Phases](#9-implementation-phases)

---

## 1. Executive Summary

### Current Problems

The existing implementation has several fundamental issues:

1. **Mixed concerns**: `LightningBoltEffect` combines physics simulation with rendering, stepping the leader 2 segments per frame during the SEARCHING phase.

2. **Non-deterministic**: `AtmosphericField` uses `Math.random()` for its permutation table, making simulations non-reproducible.

3. **Broken PRNG**: `SteppedLeader` uses a weak LCG (`seed * 9301 + 49297 % 233280`) with poor statistical properties.

4. **Naive branching**: Branches are randomly injected based on a threshold check, not emergent from field competition.

5. **Unbounded memory**: `AtmosphericField` caches field values in a Map that grows indefinitely.

6. **No pre-computation**: The bolt geometry is computed frame-by-frame, preventing smooth animation playback.

### Solution Overview

Replace with a clean simulation engine that:

- Runs to completion before any rendering begins
- Produces a complete `BoltGeometry` data structure
- Uses proper DBM growth probability: `P(i) = |phi_i|^eta / sum_j(|phi_j|^eta)`
- Supports deterministic replay via seeded PRNG
- Scales between globe (coarse) and showcase (fine) detail levels
- Separates animation timing from simulation logic

---

## 2. Simulation Module Architecture

### 2.1 Design Principles

```
┌─────────────────────────────────────────────────────────────┐
│                    Pure Simulation Module                    │
│                                                             │
│  - Zero Three.js dependencies                               │
│  - Zero rendering logic                                     │
│  - Works in normalized coordinate space [0,1]               │
│  - Synchronous execution (or Web Worker for high detail)    │
│  - Deterministic given seed                                 │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                       BoltGeometry
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Animation Layer                           │
│                                                             │
│  - Maps simulation steps to wall-clock time                 │
│  - Controls reveal progression                              │
│  - Manages phases (leader, connection, stroke, fade)        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Rendering Layer                           │
│                                                             │
│  - Three.js geometry generation                             │
│  - Coordinate transform (normalized → world)                │
│  - Material and visual effects                              │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Module Interface

```typescript
// simulation/BoltSimulator.ts

export interface SimulationInput {
  start: Vec3;           // Normalized [0,1] space
  end: Vec3;             // Normalized [0,1] space
  seed: number;          // For deterministic replay
  config: SimulationConfig;
}

export interface SimulationOutput {
  geometry: BoltGeometry;
  stats: SimulationStats;
}

export function simulateBolt(input: SimulationInput): SimulationOutput;
```

### 2.3 Coordinate Space

The simulation operates in **normalized space** where:
- Origin: midpoint between start and end
- Y-axis: aligned with start→end direction
- Scale: total distance = 1.0

This decouples simulation from globe radius, altitude units, etc. The coordinate transform happens at the rendering layer.

```typescript
// Normalized space:
//   start = (0, 0.5, 0)
//   end   = (0, -0.5, 0)
//   Total path length = 1.0
```

---

## 3. Core Algorithm: Field-Biased DBM

### 3.1 Distance-Based Field Approximation

Full Laplace solve is too expensive. Instead, use superposition of contributions:

```typescript
function computeFieldAtPoint(
  point: Vec3,
  channelPoints: Vec3[],
  groundY: number,
  config: FieldConfig
): number {
  // Background field (uniform, pointing toward ground)
  let field = config.backgroundField;

  // Enhancement from proximity to existing channel
  // Each channel point acts as a conductor, enhancing local field
  for (const cp of channelPoints) {
    const dist = distance(point, cp);
    if (dist < config.epsilon) continue;

    // Field enhancement inversely proportional to distance
    // This approximates the effect of the conductor without solving Laplace
    field += config.channelInfluence / (dist + config.epsilon);
  }

  // Ground proximity effect (image charge)
  const groundDist = point.y - groundY;
  if (groundDist > 0) {
    field += config.groundInfluence / (groundDist + config.epsilon);
  }

  // Atmospheric noise (pre-computed or sampled)
  const noise = sampleNoise3D(point, config.noiseScale, config.noiseSeed);
  field *= (1 + noise * config.noiseAmplitude);

  return field;
}
```

### 3.2 Growth Step Algorithm

```typescript
function growthStep(
  state: SimulationState,
  config: SimulationConfig
): GrowthResult {
  const candidates: Candidate[] = [];

  // For each active head, enumerate candidate directions
  for (const head of state.activeHeads) {
    const directions = generateCandidateDirections(
      head.position,
      head.direction,
      config.candidateCount,      // 8-16 candidates
      config.coneHalfAngle,       // 30-45 degrees
      state.rng
    );

    for (const dir of directions) {
      const nextPos = add(head.position, scale(dir, config.stepLength));

      // Skip if out of bounds or too close to existing channel
      if (!isValidPosition(nextPos, state, config)) continue;

      const fieldValue = computeFieldAtPoint(
        nextPos,
        state.channelPoints,
        state.groundY,
        config.fieldConfig
      );

      candidates.push({
        headId: head.id,
        position: nextPos,
        direction: dir,
        fieldValue,
        depth: head.depth
      });
    }
  }

  if (candidates.length === 0) {
    return { connected: false, terminated: true };
  }

  // Compute DBM probabilities
  const probabilities = computeDBMProbabilities(candidates, config.eta);

  // Select winner(s) based on branching logic
  return selectGrowthPoints(candidates, probabilities, state, config);
}
```

### 3.3 DBM Probability Computation

```typescript
function computeDBMProbabilities(
  candidates: Candidate[],
  eta: number
): number[] {
  // P(i) = |phi_i|^eta / sum_j(|phi_j|^eta)

  const powers = candidates.map(c => Math.pow(Math.abs(c.fieldValue), eta));
  const sum = powers.reduce((a, b) => a + b, 0);

  if (sum === 0) {
    // Uniform distribution if all fields are zero
    return candidates.map(() => 1 / candidates.length);
  }

  return powers.map(p => p / sum);
}
```

### 3.4 Emergent Branching

Branching emerges when multiple candidates from the same head have similar high probabilities:

```typescript
function selectGrowthPoints(
  candidates: Candidate[],
  probabilities: number[],
  state: SimulationState,
  config: SimulationConfig
): GrowthResult {
  const selected: SelectedCandidate[] = [];
  const newHeads: GrowthHead[] = [];

  // Group candidates by head
  const byHead = groupBy(candidates, c => c.headId);

  for (const [headId, headCandidates] of byHead) {
    const head = state.activeHeads.find(h => h.id === headId)!;
    const headProbs = headCandidates.map(c =>
      probabilities[candidates.indexOf(c)]
    );

    // Sort by probability descending
    const sorted = headCandidates
      .map((c, i) => ({ candidate: c, prob: headProbs[i] }))
      .sort((a, b) => b.prob - a.prob);

    // Primary selection: highest probability
    const primary = sorted[0];
    selected.push({ ...primary.candidate, isPrimary: true });

    // Branch check: if second-highest is close to highest, may branch
    if (sorted.length > 1 && head.depth < config.maxBranchDepth) {
      const branchThreshold = config.branchProbabilityRatio; // e.g., 0.7
      const depthDecay = Math.exp(-head.depth / config.branchDepthDecay);

      for (let i = 1; i < sorted.length && i < config.maxBranchesPerStep; i++) {
        const ratio = sorted[i].prob / primary.prob;

        if (ratio > branchThreshold) {
          // Candidate field is competitive, evaluate for branching
          const branchProb = config.baseBranchProb * ratio * depthDecay;

          if (state.rng.next() < branchProb) {
            selected.push({
              ...sorted[i].candidate,
              isPrimary: false,
              newDepth: head.depth + 1
            });
          }
        }
      }
    }
  }

  // Create new heads from selected points
  for (const sel of selected) {
    const newHead: GrowthHead = {
      id: state.nextHeadId++,
      position: sel.position,
      direction: sel.direction,
      depth: sel.isPrimary ? sel.depth : sel.newDepth!,
      parentHeadId: sel.headId,
      stepIndex: state.currentStep
    };
    newHeads.push(newHead);
  }

  // Check for ground connection
  const connected = selected.some(s =>
    distance(s.position, { x: 0, y: state.groundY, z: 0 }) < config.connectionThreshold
  );

  return {
    connected,
    terminated: false,
    newHeads,
    segments: createSegments(selected, state)
  };
}
```

### 3.5 Seeded PRNG: mulberry32

Replace the broken LCG with mulberry32, a high-quality 32-bit PRNG:

```typescript
// simulation/prng.ts

export function createRNG(seed: number): () => number {
  let state = seed >>> 0;

  return function mulberry32(): number {
    state = (state + 0x6D2B79F5) >>> 0;
    let t = state;
    t = Math.imul(t ^ (t >>> 15), t | 1);
    t ^= t + Math.imul(t ^ (t >>> 7), t | 61);
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}

export interface SeededRNG {
  next(): number;           // [0, 1)
  nextInt(max: number): number;
  nextGaussian(): number;   // Box-Muller
  fork(): SeededRNG;        // Create child RNG with derived seed
}

export function createSeededRNG(seed: number): SeededRNG {
  const base = createRNG(seed);
  let hasSpare = false;
  let spare = 0;

  return {
    next: base,

    nextInt(max: number): number {
      return Math.floor(base() * max);
    },

    nextGaussian(): number {
      if (hasSpare) {
        hasSpare = false;
        return spare;
      }

      let u, v, s;
      do {
        u = base() * 2 - 1;
        v = base() * 2 - 1;
        s = u * u + v * v;
      } while (s >= 1 || s === 0);

      const mul = Math.sqrt(-2 * Math.log(s) / s);
      spare = v * mul;
      hasSpare = true;
      return u * mul;
    },

    fork(): SeededRNG {
      // Derive a new seed from current state
      const childSeed = Math.floor(base() * 0xFFFFFFFF);
      return createSeededRNG(childSeed);
    }
  };
}
```

---

## 4. Data Structures

### 4.1 BoltGeometry

The complete output of simulation:

```typescript
// simulation/types.ts

export interface Vec3 {
  x: number;
  y: number;
  z: number;
}

export interface BoltSegment {
  id: number;
  start: Vec3;
  end: Vec3;
  depth: number;           // 0 = main channel, 1+ = branches
  parentSegmentId: number | null;
  stepIndex: number;       // When this segment was created (for animation)
  intensity: number;       // Field strength at creation (affects brightness)
  isMainChannel: boolean;  // Part of the path that reaches ground
}

export interface BoltGeometry {
  segments: BoltSegment[];

  // Tree structure for traversal
  segmentTree: SegmentNode;

  // Main channel (the path that connects to ground)
  mainChannelIds: number[];

  // Animation metadata
  totalSteps: number;
  connectionStep: number;  // Step when ground was reached

  // Bounds for coordinate mapping
  bounds: {
    min: Vec3;
    max: Vec3;
  };
}

export interface SegmentNode {
  segmentId: number;
  children: SegmentNode[];
}
```

### 4.2 SimulationConfig

All tunable parameters:

```typescript
export interface SimulationConfig {
  // Detail level preset
  detailLevel: DetailLevel;

  // DBM parameters
  eta: number;                    // Branching exponent (1.5-3.0, default 2.0)
  stepLength: number;             // Normalized units (0.005-0.02)
  maxSteps: number;               // Safety limit

  // Candidate generation
  candidateCount: number;         // Directions per head (8-24)
  coneHalfAngle: number;          // Radians (0.5-0.8, ~30-45 degrees)

  // Branching
  maxBranchDepth: number;         // Max generations (2-4)
  baseBranchProb: number;         // Base probability (0.02-0.1)
  branchProbabilityRatio: number; // Threshold for competition (0.5-0.8)
  branchDepthDecay: number;       // Lambda for exp decay (1.5-2.5)
  maxBranchesPerStep: number;     // Limit simultaneous branches (1-3)

  // Field computation
  fieldConfig: FieldConfig;

  // Connection
  connectionThreshold: number;    // Distance to ground for connection

  // Performance
  maxSegments: number;            // Hard limit on output size
}

export interface FieldConfig {
  backgroundField: number;        // Base downward field
  channelInfluence: number;       // Field enhancement near channel
  groundInfluence: number;        // Field enhancement near ground
  epsilon: number;                // Regularization for distance
  noiseScale: number;             // Spatial frequency of noise
  noiseAmplitude: number;         // Strength of atmospheric variation
  noiseSeed: number;              // For deterministic noise
}

export enum DetailLevel {
  GLOBE = 'globe',
  SHOWCASE = 'showcase'
}
```

### 4.3 Detail Level Presets

```typescript
export const DETAIL_PRESETS: Record<DetailLevel, Partial<SimulationConfig>> = {
  [DetailLevel.GLOBE]: {
    stepLength: 0.02,
    maxSteps: 80,
    candidateCount: 8,
    maxBranchDepth: 2,
    baseBranchProb: 0.03,
    maxBranchesPerStep: 1,
    maxSegments: 100,
    fieldConfig: {
      // Simplified field, less computation
      backgroundField: 1.0,
      channelInfluence: 0.3,
      groundInfluence: 0.5,
      epsilon: 0.01,
      noiseScale: 0.1,
      noiseAmplitude: 0.2,
      noiseSeed: 0  // Will be set from main seed
    }
  },

  [DetailLevel.SHOWCASE]: {
    stepLength: 0.008,
    maxSteps: 200,
    candidateCount: 16,
    maxBranchDepth: 4,
    baseBranchProb: 0.05,
    maxBranchesPerStep: 2,
    maxSegments: 800,
    fieldConfig: {
      backgroundField: 1.0,
      channelInfluence: 0.5,
      groundInfluence: 0.7,
      epsilon: 0.005,
      noiseScale: 0.15,
      noiseAmplitude: 0.25,
      noiseSeed: 0
    }
  }
};
```

### 4.4 Flat Segment Array for Rendering

The tree structure is convenient for traversal, but rendering needs a flat array:

```typescript
export function flattenGeometry(geometry: BoltGeometry): RenderableSegment[] {
  const result: RenderableSegment[] = [];

  for (const segment of geometry.segments) {
    result.push({
      start: segment.start,
      end: segment.end,
      depth: segment.depth,
      intensity: segment.intensity,
      stepIndex: segment.stepIndex,
      isMainChannel: segment.isMainChannel
    });
  }

  // Sort by depth for proper draw order (main channel last = on top)
  result.sort((a, b) => b.depth - a.depth);

  return result;
}
```

---

## 5. Detail Level Scaling

### 5.1 Globe vs Showcase Comparison

| Aspect | Globe | Showcase |
|--------|-------|----------|
| **Time budget** | <1ms | <16ms |
| **Segment count** | 50-100 | 300-800 |
| **Branch depth** | 2 | 4 |
| **Step length** | 0.02 (50 steps) | 0.008 (125 steps) |
| **Candidates/step** | 8 | 16 |
| **Field computation** | Distance-only | Full superposition |
| **Noise sampling** | Low-res | High-res |

### 5.2 Scaling Strategy

The same core algorithm serves both levels; only parameters change:

```typescript
export function createConfig(
  level: DetailLevel,
  overrides?: Partial<SimulationConfig>
): SimulationConfig {
  const preset = DETAIL_PRESETS[level];
  const base: SimulationConfig = {
    detailLevel: level,
    eta: 2.0,
    stepLength: preset.stepLength!,
    maxSteps: preset.maxSteps!,
    candidateCount: preset.candidateCount!,
    coneHalfAngle: Math.PI / 6, // 30 degrees
    maxBranchDepth: preset.maxBranchDepth!,
    baseBranchProb: preset.baseBranchProb!,
    branchProbabilityRatio: 0.6,
    branchDepthDecay: 2.0,
    maxBranchesPerStep: preset.maxBranchesPerStep!,
    fieldConfig: { ...preset.fieldConfig! },
    connectionThreshold: 0.02,
    maxSegments: preset.maxSegments!
  };

  return { ...base, ...overrides };
}
```

### 5.3 Performance Budgets

```typescript
// Globe: Must complete in <1ms
// At 8 candidates per step, 80 steps max:
//   - 640 field evaluations max
//   - Each evaluation: ~5 distance calculations (channel length / step)
//   - Total: ~3200 distance ops = trivial

// Showcase: Can take up to 16ms (one frame)
// At 16 candidates per step, 200 steps max:
//   - 3200 field evaluations max
//   - Each evaluation: ~25 distance calculations
//   - Total: ~80000 distance ops
//   - Still fast, but could use spatial acceleration if needed
```

### 5.4 Spatial Acceleration (Showcase Only)

For showcase with many channel points, use a simple grid:

```typescript
class SpatialGrid {
  private cells: Map<string, Vec3[]> = new Map();
  private cellSize: number;

  constructor(cellSize: number = 0.05) {
    this.cellSize = cellSize;
  }

  add(point: Vec3): void {
    const key = this.cellKey(point);
    if (!this.cells.has(key)) {
      this.cells.set(key, []);
    }
    this.cells.get(key)!.push(point);
  }

  getNearby(point: Vec3, radius: number): Vec3[] {
    const result: Vec3[] = [];
    const cellRadius = Math.ceil(radius / this.cellSize);
    const baseCell = this.cellCoords(point);

    for (let dx = -cellRadius; dx <= cellRadius; dx++) {
      for (let dy = -cellRadius; dy <= cellRadius; dy++) {
        for (let dz = -cellRadius; dz <= cellRadius; dz++) {
          const key = `${baseCell.x + dx},${baseCell.y + dy},${baseCell.z + dz}`;
          const cell = this.cells.get(key);
          if (cell) {
            for (const p of cell) {
              if (distance(point, p) <= radius) {
                result.push(p);
              }
            }
          }
        }
      }
    }

    return result;
  }

  private cellKey(p: Vec3): string {
    const c = this.cellCoords(p);
    return `${c.x},${c.y},${c.z}`;
  }

  private cellCoords(p: Vec3): { x: number; y: number; z: number } {
    return {
      x: Math.floor(p.x / this.cellSize),
      y: Math.floor(p.y / this.cellSize),
      z: Math.floor(p.z / this.cellSize)
    };
  }
}
```

---

## 6. Animation Timeline

### 6.1 BoltTimeline Structure

The simulation produces geometry; the timeline controls how it's revealed:

```typescript
// animation/BoltTimeline.ts

export interface BoltTimeline {
  // Total durations (milliseconds)
  leaderDuration: number;      // Time to reveal all leader segments
  connectionPause: number;     // Pause at connection
  returnStrokeDuration: number; // Time for return stroke to travel
  strokeHoldDuration: number;  // Time at peak brightness
  fadeDuration: number;        // Time to fade out

  // Subsequent strokes (the flickering)
  subsequentStrokes: number;   // Number of re-illuminations
  interstrokeInterval: number; // Time between strokes

  // Computed from geometry
  totalSteps: number;
  connectionStep: number;
  mainChannelLength: number;
}

export interface AnimationState {
  phase: AnimationPhase;
  phaseProgress: number;       // 0-1 within current phase
  elapsedTime: number;

  // What to render
  visibleSegments: Set<number>;
  segmentBrightness: Map<number, number>;
  returnStrokePosition: number; // 0-1 along main channel
  strokeCount: number;          // Which subsequent stroke we're on
}

export enum AnimationPhase {
  LEADER_STEPPING = 'leader_stepping',
  CONNECTION_PAUSE = 'connection_pause',
  RETURN_STROKE = 'return_stroke',
  STROKE_HOLD = 'stroke_hold',
  FADING = 'fading',
  INTERSTROKE = 'interstroke',
  COMPLETE = 'complete'
}
```

### 6.2 Animation Controller

```typescript
export class BoltAnimator {
  private geometry: BoltGeometry;
  private timeline: BoltTimeline;
  private startTime: number = 0;

  constructor(geometry: BoltGeometry, timeline: BoltTimeline) {
    this.geometry = geometry;
    this.timeline = timeline;
  }

  start(currentTime: number): void {
    this.startTime = currentTime;
  }

  update(currentTime: number): AnimationState {
    const elapsed = currentTime - this.startTime;
    return this.computeState(elapsed);
  }

  private computeState(elapsed: number): AnimationState {
    // Phase timing
    const leaderEnd = this.timeline.leaderDuration;
    const pauseEnd = leaderEnd + this.timeline.connectionPause;
    const strokeEnd = pauseEnd + this.timeline.returnStrokeDuration;
    const holdEnd = strokeEnd + this.timeline.strokeHoldDuration;
    const fadeEnd = holdEnd + this.timeline.fadeDuration;

    // Single stroke cycle duration
    const strokeCycle = this.timeline.returnStrokeDuration +
                        this.timeline.strokeHoldDuration +
                        this.timeline.fadeDuration;
    const fullCycle = strokeCycle + this.timeline.interstrokeInterval;

    if (elapsed < leaderEnd) {
      return this.leaderState(elapsed / leaderEnd);
    }

    if (elapsed < pauseEnd) {
      return this.connectionPauseState();
    }

    // Handle subsequent strokes
    const postPauseElapsed = elapsed - pauseEnd;
    const strokeIndex = Math.floor(postPauseElapsed / fullCycle);

    if (strokeIndex >= this.timeline.subsequentStrokes) {
      return this.completeState();
    }

    const withinCycle = postPauseElapsed % fullCycle;

    if (withinCycle < this.timeline.returnStrokeDuration) {
      return this.returnStrokeState(
        withinCycle / this.timeline.returnStrokeDuration,
        strokeIndex
      );
    }

    if (withinCycle < this.timeline.returnStrokeDuration + this.timeline.strokeHoldDuration) {
      return this.strokeHoldState(strokeIndex);
    }

    if (withinCycle < strokeCycle) {
      const fadeProgress = (withinCycle - this.timeline.returnStrokeDuration -
                           this.timeline.strokeHoldDuration) / this.timeline.fadeDuration;
      return this.fadingState(fadeProgress, strokeIndex);
    }

    return this.interstrokeState(strokeIndex);
  }

  private leaderState(progress: number): AnimationState {
    // Reveal segments progressively based on their stepIndex
    const targetStep = Math.floor(progress * this.geometry.totalSteps);
    const visible = new Set<number>();
    const brightness = new Map<number, number>();

    for (const seg of this.geometry.segments) {
      if (seg.stepIndex <= targetStep) {
        visible.add(seg.id);

        // Recent segments are brighter
        const age = targetStep - seg.stepIndex;
        const b = Math.max(0.3, 1 - age * 0.02);
        brightness.set(seg.id, b * seg.intensity);
      }
    }

    return {
      phase: AnimationPhase.LEADER_STEPPING,
      phaseProgress: progress,
      elapsedTime: progress * this.timeline.leaderDuration,
      visibleSegments: visible,
      segmentBrightness: brightness,
      returnStrokePosition: 0,
      strokeCount: 0
    };
  }

  private returnStrokeState(progress: number, strokeIndex: number): AnimationState {
    // Return stroke travels from ground up the main channel
    // Segments brighten as the stroke passes them
    const visible = new Set(this.geometry.segments.map(s => s.id));
    const brightness = new Map<number, number>();

    // Main channel lights up from bottom
    const mainChannel = this.geometry.mainChannelIds;
    const litCount = Math.floor(progress * mainChannel.length);

    for (let i = 0; i < mainChannel.length; i++) {
      const segId = mainChannel[mainChannel.length - 1 - i]; // Reverse order (ground up)
      const seg = this.geometry.segments.find(s => s.id === segId)!;

      if (i < litCount) {
        // Already passed - full brightness with slight decay
        const decay = 1 - (litCount - i) * 0.01;
        brightness.set(segId, Math.max(0.8, decay));
      } else if (i === litCount) {
        // At wavefront - peak brightness
        brightness.set(segId, 1.0);
      } else {
        // Not yet reached - dim leader remnant
        brightness.set(segId, 0.2);
      }
    }

    // Branches illuminate briefly as their parent segment lights up
    for (const seg of this.geometry.segments) {
      if (!seg.isMainChannel) {
        const parentLit = brightness.get(seg.parentSegmentId!) ?? 0;
        brightness.set(seg.id, parentLit * 0.3 * Math.exp(-seg.depth * 0.5));
      }
    }

    return {
      phase: AnimationPhase.RETURN_STROKE,
      phaseProgress: progress,
      elapsedTime: 0, // Would need full tracking
      visibleSegments: visible,
      segmentBrightness: brightness,
      returnStrokePosition: progress,
      strokeCount: strokeIndex
    };
  }

  // ... other state methods follow similar pattern
}
```

### 6.3 Timeline Presets

```typescript
export const TIMELINE_PRESETS = {
  [DetailLevel.GLOBE]: {
    leaderDuration: 200,       // Fast for background effect
    connectionPause: 30,
    returnStrokeDuration: 50,
    strokeHoldDuration: 80,
    fadeDuration: 200,
    subsequentStrokes: 2,
    interstrokeInterval: 60
  },

  [DetailLevel.SHOWCASE]: {
    leaderDuration: 800,       // Visible stepping
    connectionPause: 50,
    returnStrokeDuration: 80,  // Visible upward travel
    strokeHoldDuration: 150,
    fadeDuration: 400,
    subsequentStrokes: 4,
    interstrokeInterval: 80
  }
};
```

---

## 7. Integration Points

### 7.1 LightningBoltEffect Changes

The effect becomes a thin wrapper that:
1. Triggers simulation once
2. Owns an animator
3. Delegates rendering

```typescript
// effects/LightningBoltEffect/LightningBoltEffect.ts (revised)

export class LightningBoltEffect implements Effect {
  private geometry: BoltGeometry | null = null;
  private animator: BoltAnimator | null = null;
  private renderer: BoltRenderer;
  private coordinateTransform: LightningCoordinateTransform;

  private isCompleted: boolean = false;
  private isInitialized: boolean = false;

  constructor(
    private scene: THREE.Scene,
    private globeEl: any,
    private config: LightningBoltEffectConfig
  ) {
    this.coordinateTransform = new LightningCoordinateTransform(globeEl);
    this.renderer = new BoltRenderer(scene);
  }

  private initialize(): void {
    if (this.isInitialized) return;
    this.isInitialized = true;

    // Convert world coordinates to normalized simulation space
    const start = this.coordinateTransform.toWorldCoordinates(
      this.config.lat, this.config.lng, this.config.startAltitude
    );
    const end = this.coordinateTransform.getGroundPoint(
      this.config.lat, this.config.lng
    );

    const normalizedStart = { x: 0, y: 0.5, z: 0 };
    const normalizedEnd = { x: 0, y: -0.5, z: 0 };

    // Run simulation to completion
    const simConfig = createConfig(
      this.config.detailLevel ?? DetailLevel.GLOBE,
      { /* any overrides */ }
    );

    const result = simulateBolt({
      start: normalizedStart,
      end: normalizedEnd,
      seed: this.config.seed ?? Date.now(),
      config: simConfig
    });

    this.geometry = result.geometry;

    // Create timeline and animator
    const timeline = createTimeline(this.geometry, this.config.detailLevel);
    this.animator = new BoltAnimator(this.geometry, timeline);

    // Initialize renderer with geometry (creates buffers)
    this.renderer.setGeometry(
      this.geometry,
      this.coordinateTransform,
      start,
      end
    );
  }

  update(currentTime: number): void {
    if (!this.isInitialized) {
      this.initialize();
      this.animator!.start(currentTime);
    }

    const state = this.animator!.update(currentTime);

    if (state.phase === AnimationPhase.COMPLETE) {
      this.markComplete();
      return;
    }

    this.renderer.render(state);
  }

  // ... rest of lifecycle methods
}
```

### 7.2 Shared Simulation Module

Both `LightningLayer` (globe) and `LightningController` (showcase) use the same simulation:

```typescript
// In LightningLayer (globe view)
const config = createConfig(DetailLevel.GLOBE);
const result = simulateBolt({ start, end, seed, config });

// In LightningController (showcase view)
const config = createConfig(DetailLevel.SHOWCASE, {
  eta: userSettings.eta,           // Allow user tuning
  maxBranchDepth: userSettings.branchDepth
});
const result = simulateBolt({ start, end, seed, config });
```

### 7.3 Coordinate Transform Location

The transform stays in the effect/layer, NOT in simulation:

```typescript
// Simulation works in normalized [-0.5, 0.5] space
// Effect/Layer transforms to world coordinates

export class BoltRenderer {
  private worldTransform: {
    origin: Vec3;
    scale: number;
    rotation: THREE.Quaternion;
  } | null = null;

  setGeometry(
    geometry: BoltGeometry,
    coordTransform: LightningCoordinateTransform,
    worldStart: Vec3,
    worldEnd: Vec3
  ): void {
    // Compute transform from normalized to world
    const worldLength = distance(worldStart, worldEnd);
    const worldDir = normalize(subtract(worldEnd, worldStart));

    this.worldTransform = {
      origin: midpoint(worldStart, worldEnd),
      scale: worldLength,
      rotation: quaternionFromDirection(worldDir)
    };

    // Build geometry buffers with world-space positions
    this.buildBuffers(geometry);
  }

  private toWorldSpace(normalized: Vec3): Vec3 {
    const t = this.worldTransform!;
    const scaled = scale(normalized, t.scale);
    const rotated = applyQuaternion(scaled, t.rotation);
    return add(rotated, t.origin);
  }
}
```

---

## 8. File Organization

### 8.1 Proposed Structure

```
client/src/
  effects/
    LightningBoltEffect/
      simulation/                    # NEW: Pure TypeScript simulation
        index.ts                     # Public exports
        BoltSimulator.ts             # Main simulation entry point
        GrowthStep.ts                # Single step logic
        FieldComputation.ts          # Field approximation
        BranchSelection.ts           # DBM probability & branching
        types.ts                     # BoltGeometry, SimulationConfig, etc.
        config.ts                    # Detail level presets
        prng.ts                      # Seeded PRNG (mulberry32)
        spatial.ts                   # SpatialGrid for showcase
        noise.ts                     # Seeded Perlin noise

      animation/                     # NEW: Timeline and state management
        index.ts
        BoltTimeline.ts              # Timeline structure and creation
        BoltAnimator.ts              # Animation state machine
        types.ts                     # AnimationState, AnimationPhase

      physics/                       # DEPRECATED - to be removed
        SteppedLeader.ts             # DELETE after migration
        AtmosphericField.ts          # DELETE after migration
        ReturnStroke.ts              # DELETE after migration
        index.ts                     # UPDATE to re-export from simulation/

      rendering/                     # MODIFY: Simpler, state-driven
        index.ts
        BoltRenderer.ts              # NEW: Unified renderer consuming AnimationState
        LeaderRenderer.ts            # DEPRECATED - merge into BoltRenderer
        StrokeRenderer.ts            # DEPRECATED - merge into BoltRenderer
        FlashEffect.ts               # KEEP: Point light effect
        ScreenFlashEffect.ts         # KEEP: Screen overlay flash
        materials.ts                 # NEW: Shared material definitions

      LightningBoltEffect.ts         # MODIFY: Use simulation + animation
      LightningTypes.ts              # KEEP: Coordinate transform stays here
      index.ts
```

### 8.2 Files to Delete

After migration is complete:
- `physics/SteppedLeader.ts`
- `physics/AtmosphericField.ts`
- `physics/ReturnStroke.ts`
- `rendering/LeaderRenderer.ts`
- `rendering/StrokeRenderer.ts`

### 8.3 Files to Create

New files:
- `simulation/index.ts`
- `simulation/BoltSimulator.ts`
- `simulation/GrowthStep.ts`
- `simulation/FieldComputation.ts`
- `simulation/BranchSelection.ts`
- `simulation/types.ts`
- `simulation/config.ts`
- `simulation/prng.ts`
- `simulation/spatial.ts`
- `simulation/noise.ts`
- `animation/index.ts`
- `animation/BoltTimeline.ts`
- `animation/BoltAnimator.ts`
- `animation/types.ts`
- `rendering/BoltRenderer.ts`
- `rendering/materials.ts`

### 8.4 Files to Modify

- `LightningBoltEffect.ts` - Complete rewrite of update logic
- `LightningTypes.ts` - Add `DetailLevel` enum
- `physics/index.ts` - Re-export from simulation for backward compat
- `rendering/index.ts` - Export new `BoltRenderer`
- `config/definitions/lightning.ts` - Add simulation parameters

---

## 9. Implementation Phases

### Phase 1: Core Simulation (Priority: Critical)

**Goal**: Working DBM simulation that produces `BoltGeometry`.

**Files**:
- `simulation/types.ts`
- `simulation/prng.ts`
- `simulation/config.ts`
- `simulation/noise.ts`
- `simulation/FieldComputation.ts`
- `simulation/GrowthStep.ts`
- `simulation/BranchSelection.ts`
- `simulation/BoltSimulator.ts`
- `simulation/index.ts`

**Validation**:
- Unit tests for PRNG distribution
- Unit tests for field computation
- Integration test: simulate with known seed, verify determinism
- Visual test: log geometry, verify reasonable segment count and structure

**Estimated effort**: 2-3 days

### Phase 2: Animation Timeline (Priority: High)

**Goal**: Working animator that reveals geometry over time.

**Files**:
- `animation/types.ts`
- `animation/BoltTimeline.ts`
- `animation/BoltAnimator.ts`
- `animation/index.ts`

**Validation**:
- Unit tests for phase transitions
- Unit tests for segment visibility at key times
- Integration test: animate geometry, verify all segments visible at end

**Estimated effort**: 1-2 days

### Phase 3: Rendering Integration (Priority: High)

**Goal**: New renderer consuming animation state.

**Files**:
- `rendering/BoltRenderer.ts`
- `rendering/materials.ts`
- Modify `LightningBoltEffect.ts`

**Validation**:
- Visual test in showcase view
- Performance test: verify <16ms for showcase, <1ms for globe
- Compare visual quality to reference

**Estimated effort**: 2 days

### Phase 4: Layer Integration (Priority: Medium)

**Goal**: Both globe and showcase views working.

**Files**:
- Modify `LightningTypes.ts`
- Modify `LightningLayer.ts`
- Create or modify showcase controller

**Validation**:
- End-to-end test on globe
- End-to-end test in showcase
- Performance profiling

**Estimated effort**: 1 day

### Phase 5: Cleanup (Priority: Low)

**Goal**: Remove deprecated code, finalize API.

**Tasks**:
- Delete old physics files
- Delete old renderer files
- Update exports
- Documentation

**Estimated effort**: 0.5 days

---

## Appendix A: Key Formulas Reference

### DBM Growth Probability
```
P(i) = |phi_i|^eta / sum_j(|phi_j|^eta)
```

### Branch Probability with Depth Decay
```
P_branch = P_0 * (E_local / E_ref)^beta * exp(-depth / lambda)
```

### Distance-Based Field Approximation
```
E(r) = E_0 + sum_i(k / |r - r_i|) + k_ground / (y - y_ground)
```

### Step Direction Sampling
```
P(theta) ~ exp(-(theta - theta_0)^2 / 2*sigma^2) * max(0, E(theta) - E_th)^eta
```

---

## Appendix B: Parameter Tuning Guide

| Parameter | Effect of Increasing | Suggested Range |
|-----------|---------------------|-----------------|
| `eta` | Less branching, straighter paths | 1.5 - 3.0 |
| `stepLength` | Coarser geometry, faster sim | 0.005 - 0.025 |
| `candidateCount` | Smoother paths, slower sim | 6 - 24 |
| `baseBranchProb` | More branches | 0.01 - 0.1 |
| `branchDepthDecay` | Branches die faster | 1.0 - 3.0 |
| `noiseAmplitude` | More tortuous path | 0.1 - 0.4 |
| `coneHalfAngle` | Wider deviation range | 0.3 - 0.8 rad |

---

*Document created: 2026-02-07*
*Last updated: 2026-02-07*
