# Pre-computation vs Real-time

A fundamental architectural decision: the simulation runs to completion before any rendering begins. This document explains why.

---

## The Problem with Real-time Physics

The original implementation stepped the physics during the render loop:

```javascript
// OLD APPROACH (problematic)
update(frame) {
  if (phase === SEARCHING) {
    advanceLeader();  // Physics step
    updateGeometry(); // Recreate buffers
  }
  render();
}
```

### Issues

1. **Mixed concerns**: Physics logic interleaved with rendering
2. **Non-deterministic**: Timing affects physics (frame rate variations)
3. **No replay**: Can't show the same bolt twice
4. **Geometry churn**: Buffer recreation every frame
5. **Hard to debug**: State scattered across frames

---

## The Solution: Separate Simulation

```
┌─────────────────────────────────────────────────────────────┐
│                    Pure Simulation Module                    │
│                                                             │
│  - Zero Three.js dependencies                               │
│  - Synchronous execution                                    │
│  - Deterministic given seed                                 │
│  - Returns complete BoltGeometry                            │
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
│  - Pure state machine, no physics                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                       AnimationState
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Rendering Layer                           │
│                                                             │
│  - Three.js geometry from BoltGeometry (once)               │
│  - Updates visibility/opacity from AnimationState           │
│  - No physics knowledge                                     │
└─────────────────────────────────────────────────────────────┘
```

---

## Benefits

### Determinism

Same seed = same bolt, always.

```javascript
const bolt1 = simulateBolt({ seed: 12345, ... });
const bolt2 = simulateBolt({ seed: 12345, ... });
// bolt1.geometry === bolt2.geometry (structurally)
```

Enables:
- Replay functionality
- Comparison testing
- Predictable debugging

### Separation of Concerns

Each layer has one job:
- **Simulation**: "What is the bolt's structure?"
- **Animation**: "Which parts are visible when?"
- **Rendering**: "How do we draw visible parts?"

### Performance Characteristics

Simulation: Single burst of CPU work
- GLOBE: <1ms
- SHOWCASE: <16ms

Animation: Trivial per-frame
- State lookup
- Simple arithmetic

Rendering: GPU-bound
- Geometry created once
- Visibility updates via uniforms/attributes

No garbage collection spikes from geometry recreation.

### Testability

```javascript
// Unit test simulation in isolation
const result = simulateBolt(testInput);
expect(result.geometry.segments.length).toBeGreaterThan(50);
expect(result.stats.connected).toBe(true);

// Unit test animation in isolation
const animator = new BoltAnimator(geometry, timeline);
animator.start(0);
const state = animator.update(500);
expect(state.phase).toBe(AnimationPhase.LEADER_STEPPING);
```

No Three.js mocking required for physics tests.

---

## The Trade-off

### Initial Load Time

Simulation must complete before first frame renders.

SHOWCASE config: up to 16ms before animation starts.

**Mitigation:**
- 16ms is imperceptible
- Could show loading indicator for slower configs
- Could use Web Worker if needed (not currently)

### Memory Usage

Entire geometry in memory at once.

SHOWCASE: ~800 segments x ~24 bytes/segment = ~20KB
Plus animation state maps: ~5KB

**Verdict:** Negligible for modern devices.

### No "Live" Interaction

Can't modify physics mid-animation.

**This is acceptable because:**
- Lightning happens fast (real: ~20ms)
- User isn't expected to interact mid-bolt
- Parameter changes trigger new simulation

---

## Implementation Details

### Simulation Entry Point

```typescript
function simulateBolt(input: SimulationInput): SimulationOutput {
  const { start, end, seed, config } = input;
  const rng = createSeededRNG(seed);

  // Create atmospheric model
  const atmosphere = createAtmosphericModel(rng.fork(), ...);

  // Run growth loop to completion
  while (!connected && step < maxSteps) {
    // ... growth step logic
  }

  // Post-process: main channel identification, depth assignment
  return {
    geometry: finalGeometry,
    stats: { ... },
    atmosphere: atmosphereData
  };
}
```

### Animation State Machine

```typescript
class BoltAnimator {
  private geometry: BoltGeometry;
  private timeline: BoltTimeline;
  private startTime: number;

  update(currentTime: number): AnimationState {
    const elapsed = currentTime - this.startTime;
    return this.computeState(elapsed);
  }

  private computeState(elapsed): AnimationState {
    // Pure function: elapsed → phase, visibility, brightness
  }
}
```

### Rendering Layer

```typescript
class BoltRenderer {
  setGeometry(geometry, worldStart, worldEnd) {
    // Create buffers ONCE
    this.buildBuffers(geometry);
  }

  render(state: AnimationState) {
    // Update visibility based on state
    // No geometry creation, just uniform updates
  }
}
```

---

## Alternative Considered: Streaming Simulation

Could have streamed geometry as computed:

```javascript
// ALTERNATIVE (not chosen)
async function* streamBolt(input) {
  while (!connected) {
    const newSegments = growthStep(...);
    yield newSegments;
  }
}

for await (const segments of streamBolt(input)) {
  addToGeometry(segments);
  render();
}
```

### Why Rejected

1. **Complexity**: Async generators, partial geometry updates
2. **Timing coupling**: Animation speed tied to simulation speed
3. **Main channel unknown**: Can't properly assign depth until complete
4. **Diminishing returns**: Simulation is fast enough synchronously

---

## Historical Context

The original implementation had:
- `SteppedLeader.ts` stepping 2 segments per frame
- `AtmosphericField.ts` caching values indefinitely (memory leak)
- `Math.random()` calls (non-deterministic)
- Geometry recreation every frame

The rewrite fixed all of these by embracing pre-computation.
