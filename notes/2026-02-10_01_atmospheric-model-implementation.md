# Atmospheric Model Implementation Plan

**Date**: 2026-02-10
**Status**: Planning
**Prerequisites**: Existing simulation engine in `simulation/`, physics references in `2026-02-07_01_lightning-physics-and-math.md`
**Builds on**: `authentic-lightning-implementation.md`

---

## Table of Contents

1. [Context & Goals](#1-context--goals)
2. [Scale Normalization](#2-scale-normalization)
3. [Atmospheric Model Design](#3-atmospheric-model-design)
4. [Implementation Stages](#4-implementation-stages)
5. [Code Organization & Refactoring](#5-code-organization--refactoring)
6. [Data Structures](#6-data-structures)
7. [Visualization System](#7-visualization-system)
8. [Testing & Validation](#8-testing--validation)
9. [Session Handoff Notes](#9-session-handoff-notes)

---

## 1. Context & Goals

### Current State (2026-02-10)

The simulation engine produces lightning bolts but with issues:
- Branches cluster at the start due to exponential growth
- Paths too straight (60% downward bias was reduced to 15%)
- No atmospheric variation guiding path selection
- Competition model partially implemented (frontrunner protection)
- `physics/` folder exists with `AtmosphericField.ts` but not integrated with `simulation/`

### Goals

1. **Physically-based atmospheric model** with charge distribution, moisture, ionization
2. **Voronoi-based field representation** with sinusoidal (smooth) cell blending
3. **Multi-leader competition** emerging from charge distribution
4. **Visualization toggles** for each atmospheric layer
5. **Code organized** for incremental development across sessions

### User Requirements (Verbatim)

- "fewer branches overall, more likely sub-branching when it happens"
- "more variety in direction, less downward forcing"
- "longer distance between branches but more likelihood of branching"
- "probability of tighter vs dispersed branching should itself be variable"
- "shouldn't be hitting caps - natural dynamics should create distribution"
- "they should all be seeking ground equally, some die, others go further"
- "ONLY once one finds ground it becomes the main one"
- "multiple starting points from charge distribution (automatic)"
- "multi-strike should be emergent, not a setting"

---

## 2. Scale Normalization

### Physical to Normalized Scale

Real lightning parameters (from `2026-02-07_01_lightning-physics-and-math.md`):
- Cloud base altitude: 5-8 km
- Stepped leader step length: 50m (range: 3-200m)
- Total leader length: 5-8 km

Current normalized space:
- Start: (0, 0.5, 0)
- End: (0, -0.5, 0)
- Total height: 1.0 unit
- Step length (SHOWCASE): 0.008 units

**Derived scale factor:**
```
1 unit = 6.25 km (6,250 meters)
0.008 units = 50 meters (one step)
```

### Constants File: `simulation/constants.ts`

```typescript
/**
 * Scale conversion between normalized simulation units and real-world meters.
 * The simulation operates in normalized space for numerical stability.
 *
 * Reference: Cloud-to-ground lightning typically spans 5-8 km.
 * Our normalized space spans 1.0 unit (from y=0.5 to y=-0.5).
 */
export const SCALE = {
  // 1 simulation unit = this many meters
  METERS_PER_UNIT: 6250,

  // 1 meter = this many simulation units
  UNITS_PER_METER: 1 / 6250,

  // Common conversions
  STEP_LENGTH_METERS: 50,        // Real stepped leader step
  STEP_LENGTH_UNITS: 0.008,      // Corresponding simulation step

  // Atmospheric feature scales (in simulation units)
  CHARGE_POCKET_RADIUS: {
    MIN: 0.08,   // ~500m in reality
    MAX: 0.24,   // ~1.5km in reality
  },

  MOISTURE_REGION_RADIUS: {
    MIN: 0.08,   // Similar to charge (both driven by convection)
    MAX: 0.32,   // Slightly larger features
  },

  IONIZATION_SEED_RADIUS: {
    MIN: 0.002,  // ~12m - cosmic ray track scale
    MAX: 0.008,  // ~50m
  },
} as const;

// Helper functions
export function metersToUnits(meters: number): number {
  return meters * SCALE.UNITS_PER_METER;
}

export function unitsToMeters(units: number): number {
  return units * SCALE.METERS_PER_UNIT;
}
```

---

## 3. Atmospheric Model Design

### 3.1 Voronoi Field System

Each atmospheric property is represented as a **Voronoi-based scalar field** with smooth blending between cells.

**Cell value distribution (sinusoidal falloff):**
```
value(point) = intensity × (cos(distance/radius × π) + 1) / 2

Where:
- distance = |point - cell.center|
- radius = cell.falloffRadius
- At center (distance=0): value = intensity
- At edge (distance=radius): value = 0
- Smooth S-curve transition, no hard boundaries
```

Multiple overlapping cells blend additively.

### 3.2 Field Layers

| Layer | Dimension | Scale | Physical Basis | Effect on Simulation |
|-------|-----------|-------|----------------|---------------------|
| **Ceiling Charge** | 2D (y=ceiling) | Large (0.08-0.24 units) | Cloud charge centers from ice collisions | Determines leader starting points |
| **Ground Charge** | 2D (y=ground) | Large (0.08-0.24 units) | Induced ground charge, conductivity | Attracts leaders, affects termination |
| **Atmospheric Charge** | 3D | Large (0.08-0.24 units) | Field concentration in air | Curves paths toward high-charge regions |
| **Moisture** | 3D | Medium-Large (0.08-0.32 units) | Humidity from convection | Lowers breakdown voltage (easier path) |
| **Pre-ionization** | 3D (sparse points) | Small (0.002-0.008 units) | Cosmic ray tracks | Creates local attraction points |

### 3.3 Starting Point Selection

Leaders originate from **ceiling charge peaks**, not arbitrary positions:

1. Generate ceiling charge field with N cells
2. Find local maxima (cell centers with highest intensity)
3. Spawn one leader per significant charge peak
4. Number of leaders emerges from charge distribution, not configuration

### 3.4 Path Selection Integration

Current `computeFieldAtPoint()` returns a scalar field value. Enhanced version:

```typescript
function computeFieldAtPoint(
  point: Vec3,
  ctx: FieldContext,
  atmo: AtmosphericModel,
  direction?: Vec3
): number {
  let field = ctx.config.backgroundField;

  // Existing: channel influence, ground proximity, noise
  // ...existing code...

  // NEW: Atmospheric charge attraction
  field += atmo.atmosphericCharge.getValue(point) * ATMO_CHARGE_WEIGHT;

  // NEW: Moisture reduces breakdown threshold (higher field = easier)
  const moisture = atmo.moisture.getValue(point);
  field *= (1 + moisture * MOISTURE_CONDUCTIVITY_BOOST);

  // NEW: Pre-ionization seeds create local attraction
  for (const seed of atmo.ionizationSeeds) {
    const dist = distance(point, seed.position);
    if (dist < seed.radius * 2) {
      field += seed.strength / (dist + 0.001);
    }
  }

  // Reduced directional bias (let atmospheric model guide direction)
  if (direction) {
    const downwardness = -direction.y;
    field *= 1 + downwardness * 0.1; // Was 0.6, then 0.15, now 0.1
  }

  return field;
}
```

### 3.5 Competition-Based Death

Leaders die based on **competition**, not timers:

```typescript
function filterByCompetition(
  heads: SimHead[],
  groundY: number,
  rng: SeededRNG
): SimHead[] {
  if (heads.length <= 1) return heads;

  // Find the leader (closest to ground)
  const sorted = [...heads].sort((a, b) => a.position.y - b.position.y);
  const leaderY = sorted[0].position.y;
  const totalProgress = 0.5 - groundY; // Full journey distance

  return heads.filter(head => {
    // How far behind the leader?
    const lag = head.position.y - leaderY;
    const lagRatio = lag / totalProgress;

    // How much progress has this head made?
    const progress = (0.5 - head.position.y) / totalProgress;

    // Base survival
    let survivalProb = 0.97;

    // Penalty for falling behind (up to -30%)
    survivalProb -= lagRatio * 0.3;

    // Bonus for progress (up to +10%)
    survivalProb += progress * 0.1;

    // Clamp to reasonable range
    survivalProb = Math.max(0.6, Math.min(0.99, survivalProb));

    return rng.next() < survivalProb;
  });
}
```

### 3.6 Event-Based Branching

Instead of per-head probability, use **global branching events**:

```typescript
function checkBranchingEvent(
  state: GrowthState,
  atmo: AtmosphericModel,
  config: SimulationConfig
): BranchingEvent | null {
  // Base probability per step
  let eventProb = config.branchEventProbability; // e.g., 0.15

  // Modulate by local charge (check at all head positions)
  let maxLocalCharge = 0;
  for (const head of state.activeHeads) {
    const charge = atmo.atmosphericCharge.getValue(head.position);
    maxLocalCharge = Math.max(maxLocalCharge, charge);
  }
  eventProb *= (1 + maxLocalCharge * 0.5);

  // Spatial noise for "burstiness"
  const noisePos = state.activeHeads[0]?.position ?? { x: 0, y: 0, z: 0 };
  const noise = state.fieldCtx.noise3D(noisePos.x * 2, noisePos.y * 2, noisePos.z * 2);
  eventProb *= (1 + noise * 0.4);

  if (state.rng.next() > eventProb) return null;

  // Event occurs - determine how many branches (1-3)
  const branchCount = 1 + Math.floor(state.rng.next() * 2.5);

  // Select which heads branch (random selection)
  const eligibleHeads = state.activeHeads.filter(h => h.generation < 3);
  const selectedHeads: SimHead[] = [];
  for (let i = 0; i < branchCount && eligibleHeads.length > 0; i++) {
    const idx = Math.floor(state.rng.next() * eligibleHeads.length);
    selectedHeads.push(eligibleHeads.splice(idx, 1)[0]);
  }

  return { heads: selectedHeads };
}
```

---

## 4. Implementation Stages

Each stage is designed to be:
- Completable in one Claude session
- Testable independently
- Building on previous stages

### Stage 1: Constants & Scale Normalization
**Files**: `simulation/constants.ts`
**Task**: Create scale constants, update existing code to use them
**Test**: No behavior change, just organization
**Handoff**: "Constants file created, existing code unchanged"

### Stage 2: Voronoi Field Infrastructure
**Files**: `simulation/VoronoiField.ts`
**Task**: Implement VoronoiCell, VoronoiField with sinusoidal blending
**Test**: Unit test - create field, query values, verify smooth falloff
**Handoff**: "VoronoiField implemented, not yet integrated"

### Stage 3: Ceiling Charge Model
**Files**: `simulation/AtmosphericModel.ts` (partial)
**Task**: Generate ceiling charge distribution, derive starting points
**Test**: Log starting points, verify they match charge peaks
**Handoff**: "Ceiling charge works, single starting point still used"

### Stage 4: Multi-Leader Spawning
**Files**: `BoltSimulator.ts`, `GrowthStep.ts`
**Task**: Spawn multiple leaders from ceiling charge peaks
**Test**: Visual - multiple bolts descend simultaneously
**Handoff**: "Multiple leaders spawn, no competition yet"

### Stage 5: Competition-Based Death
**Files**: `GrowthStep.ts`
**Task**: Replace timer-based death with competition model
**Test**: Visual - clear winner emerges, losers fade
**Handoff**: "Competition works, winner determined by physics"

### Stage 6: Ground Charge Model
**Files**: `AtmosphericModel.ts` (extend)
**Task**: Add ground charge distribution, affect termination
**Test**: Leaders attracted to ground charge peaks
**Handoff**: "Ground charge affects termination points"

### Stage 7: Visualize Ceiling & Ground Charge
**Files**: `rendering/AtmosphericRenderer.ts`, UI toggles
**Task**: Render 2D charge distributions as gradient planes
**Test**: Visual - can see charge distribution, toggles work
**Handoff**: "2D charge visualization complete"

### Stage 8: Atmospheric Charge (3D)
**Files**: `AtmosphericModel.ts` (extend)
**Task**: Add 3D charge field, integrate with path selection
**Test**: Paths curve toward high-charge regions
**Handoff**: "3D charge affects paths"

### Stage 9: Visualize Atmospheric Charge
**Files**: `rendering/AtmosphericRenderer.ts` (extend)
**Task**: Volumetric fog rendering for 3D charge
**Test**: Visual - can see 3D charge, toggle works
**Handoff**: "3D charge visualization complete"

### Stage 10: Moisture Layer
**Files**: `AtmosphericModel.ts` (extend)
**Task**: Add moisture field, affects ionization potential
**Test**: Paths prefer moist regions
**Handoff**: "Moisture affects paths"

### Stage 11: Visualize Moisture
**Files**: `rendering/AtmosphericRenderer.ts` (extend)
**Task**: Blue-tinted volumetric for moisture
**Test**: Visual - blue moisture visible, distinct from charge
**Handoff**: "Moisture visualization complete"

### Stage 12: Pre-ionization Seeds
**Files**: `AtmosphericModel.ts` (extend)
**Task**: Sparse ionization seeds, local attraction
**Test**: Occasional path "jumps" toward seeds
**Handoff**: "Ionization seeds affect paths"

### Stage 13: Visualize Ionization
**Files**: `rendering/AtmosphericRenderer.ts` (extend)
**Task**: Red points/glow for ionization seeds
**Test**: Visual - red seeds visible, paths attracted
**Handoff**: "Ionization visualization complete"

### Stage 14: Event-Based Branching
**Files**: `GrowthStep.ts`, `BranchSelection.ts`
**Task**: Replace per-head branching with global events
**Test**: Constant branch rate over time (check logs)
**Handoff**: "Event-based branching, even distribution"

### Stage 15: Upward Leaders
**Files**: `GrowthStep.ts` (extend)
**Task**: Ground spawns upward leaders when charge high
**Test**: Visual - upward leaders meet descending
**Handoff**: "Upward leaders implemented"

### Stage 16: Integration & Polish
**Files**: All
**Task**: Parameter tuning, performance optimization
**Test**: Overall visual quality, performance targets
**Handoff**: "Atmospheric model complete"

---

## 5. Code Organization & Refactoring

### Current Structure
```
LightningBoltEffect/
├── simulation/          # Bolt geometry generation
│   ├── BoltSimulator.ts
│   ├── GrowthStep.ts
│   ├── BranchSelection.ts
│   ├── FieldComputation.ts
│   ├── config.ts
│   ├── types.ts
│   ├── noise.ts
│   ├── prng.ts
│   ├── spatial.ts
│   └── index.ts
├── physics/             # Partially implemented, not integrated
│   ├── AtmosphericField.ts
│   ├── SteppedLeader.ts
│   ├── ReturnStroke.ts
│   └── index.ts
├── animation/           # Time-based state machine
├── rendering/           # Three.js rendering
└── LightningBoltEffect.ts  # Main entry point
```

### Proposed Structure (After Refactoring)
```
LightningBoltEffect/
├── simulation/
│   ├── core/                    # Core simulation loop
│   │   ├── BoltSimulator.ts
│   │   ├── GrowthStep.ts
│   │   └── types.ts
│   ├── atmosphere/              # Atmospheric model (NEW)
│   │   ├── AtmosphericModel.ts
│   │   ├── VoronoiField.ts
│   │   ├── CeilingCharge.ts
│   │   ├── GroundCharge.ts
│   │   ├── Moisture.ts
│   │   ├── IonizationSeeds.ts
│   │   └── index.ts
│   ├── selection/               # Path & branch selection
│   │   ├── FieldComputation.ts
│   │   ├── BranchSelection.ts
│   │   ├── CompetitionModel.ts  # NEW
│   │   └── index.ts
│   ├── util/                    # Utilities
│   │   ├── constants.ts         # NEW - scale factors
│   │   ├── noise.ts
│   │   ├── prng.ts
│   │   ├── spatial.ts
│   │   └── index.ts
│   ├── config.ts
│   └── index.ts
├── animation/                   # Unchanged
├── rendering/
│   ├── bolt/                    # Bolt rendering
│   │   ├── BoltRenderer.ts
│   │   ├── LeaderRenderer.ts
│   │   ├── StrokeRenderer.ts
│   │   └── LightningMaterials.ts
│   ├── atmosphere/              # Atmospheric visualization (NEW)
│   │   ├── AtmosphericRenderer.ts
│   │   ├── ChargeFieldShader.ts
│   │   ├── MoistureShader.ts
│   │   └── index.ts
│   ├── effects/
│   │   └── FlashEffect.ts
│   └── index.ts
├── physics/                     # DELETE - merge into simulation/atmosphere
└── LightningBoltEffect.ts
```

### Refactoring Approach

**Do NOT refactor all at once.** Instead:
1. Create new structure alongside existing
2. Migrate one piece at a time
3. Delete old code only after new is working
4. Keep `index.ts` exports stable to avoid breaking changes

---

## 6. Data Structures

### VoronoiCell
```typescript
interface VoronoiCell {
  center: Vec3;
  intensity: number;        // Peak value at center (0-1)
  falloffRadius: number;    // Distance at which value → 0
}
```

### VoronoiField
```typescript
interface VoronoiField {
  cells: VoronoiCell[];

  // Get interpolated value at any point (sinusoidal blend)
  getValue(point: Vec3): number;

  // Get gradient direction (toward increasing value)
  getGradient(point: Vec3): Vec3;

  // Find local maxima (for starting point selection)
  getLocalMaxima(): Vec3[];
}
```

### AtmosphericModel
```typescript
interface AtmosphericModel {
  // 2D fields (planes)
  ceilingCharge: VoronoiField;  // y = ceilingY
  groundCharge: VoronoiField;   // y = groundY

  // 3D fields (volumetric)
  atmosphericCharge: VoronoiField;
  moisture: VoronoiField;

  // Sparse points
  ionizationSeeds: IonizationSeed[];

  // Derived data
  startingPoints: Vec3[];  // From ceiling charge peaks

  // Bounds
  ceilingY: number;
  groundY: number;
}

interface IonizationSeed {
  position: Vec3;
  strength: number;
  radius: number;
}
```

### AtmosphericConfig
```typescript
interface AtmosphericConfig {
  // Ceiling charge
  ceilingChargeCellCount: number;      // 3-8 typical
  ceilingChargeIntensityRange: [number, number];
  ceilingChargeRadiusRange: [number, number];

  // Ground charge
  groundChargeCellCount: number;
  groundChargeIntensityRange: [number, number];
  groundChargeRadiusRange: [number, number];

  // Atmospheric charge (3D)
  atmosphericChargeCellCount: number;  // 10-30 typical
  atmosphericChargeIntensityRange: [number, number];
  atmosphericChargeRadiusRange: [number, number];

  // Moisture (3D)
  moistureCellCount: number;
  moistureIntensityRange: [number, number];
  moistureRadiusRange: [number, number];

  // Ionization seeds
  ionizationSeedCount: number;         // 20-100 typical
  ionizationSeedStrengthRange: [number, number];
  ionizationSeedRadiusRange: [number, number];

  // Weights for simulation
  atmosphericChargeWeight: number;
  moistureConductivityBoost: number;
  ionizationAttractionStrength: number;
}
```

---

## 7. Visualization System

### Toggle State
```typescript
interface AtmosphericVisualization {
  showCeilingCharge: boolean;      // White gradient on ceiling plane
  showGroundCharge: boolean;       // White gradient on ground plane
  showAtmosphericCharge: boolean;  // White volumetric fog
  showMoisture: boolean;           // Blue volumetric fog
  showIonization: boolean;         // Red points/glow
}
```

### Rendering Approach

**2D Fields (Ceiling/Ground):**
- Gradient planes rendered as quads with custom shader
- Shader samples VoronoiField at each fragment
- Intensity mapped to opacity (0 = transparent, 1 = semi-opaque)
- White color for charge

**3D Fields (Atmospheric/Moisture):**
- Volumetric fog using raymarching or particle system
- Sinusoidal density distribution matches underlying field
- Semi-transparent (must see lightning through it)
- White for charge, desaturated blue for moisture

**Ionization Seeds:**
- Point sprites with glow
- Red tint, small radius
- Sparse distribution

### Shader: Sinusoidal Field Sampling
```glsl
// Fragment shader for 2D charge visualization
uniform vec3 cellCenters[MAX_CELLS];
uniform float cellIntensities[MAX_CELLS];
uniform float cellRadii[MAX_CELLS];
uniform int cellCount;

float getFieldValue(vec2 pos) {
  float value = 0.0;
  for (int i = 0; i < cellCount; i++) {
    float dist = distance(pos, cellCenters[i].xz);
    if (dist < cellRadii[i]) {
      float t = dist / cellRadii[i];
      float falloff = (cos(t * 3.14159) + 1.0) * 0.5;
      value += cellIntensities[i] * falloff;
    }
  }
  return clamp(value, 0.0, 1.0);
}

void main() {
  float intensity = getFieldValue(vPosition.xz);
  gl_FragColor = vec4(1.0, 1.0, 1.0, intensity * 0.3);
}
```

---

## 8. Testing & Validation

### Per-Stage Testing

Each stage has specific validation criteria (see Stage descriptions above).

### Integration Testing

After completing all stages:

1. **Distribution Test**: Log branch segments by step range
   - Expected: Relatively even distribution (no front-loading)

2. **Competition Test**: Run 10 simulations
   - Expected: Different winners based on charge distribution

3. **Visual Test**: Compare to reference lightning images
   - Check: Chaotic paths, varied branch lengths, natural clustering

4. **Performance Test**: Measure simulation time
   - Target: <50ms for SHOWCASE detail level

### Logging for Debugging

```typescript
// In BoltSimulator.ts
console.log('[Simulation] Starting points:', model.startingPoints.length);
console.log('[Simulation] Ceiling charge peaks:',
  model.ceilingCharge.getLocalMaxima().map(p => `(${p.x.toFixed(2)}, ${p.z.toFixed(2)})`));

// In GrowthStep.ts
console.log(`[Step ${step}] Active heads: ${heads.length}, Leader Y: ${leaderY.toFixed(3)}`);

// After simulation
const buckets = computeStepBuckets(segments);
console.log('[Simulation] Branch segments by step range:', buckets);
```

---

## 9. Session Handoff Notes

### Purpose

This section is updated at the end of each coding session to provide context for the next session.

### Template

```markdown
## Session: [DATE]

### Completed
- [List of completed stages/tasks]

### Current State
- [What's working]
- [What's partially done]
- [What's broken]

### Next Steps
- [Immediate next task]
- [Blockers if any]

### Key Decisions Made
- [Any design decisions]

### Files Modified
- [List of changed files]
```

### Current Handoff (2026-02-10)

**Completed:**
- Analysis and planning
- Existing code review
- Documentation created

**Current State:**
- Simulation engine works but has distribution issues
- Competition model partially implemented (frontrunner protection)
- Branching still front-loaded
- `physics/` folder exists but not integrated

**Next Steps:**
- Stage 1: Create constants.ts with scale normalization
- Stage 2: Implement VoronoiField

**Key Decisions:**
- Sinusoidal cell blending (not linear, not hard boundaries)
- Multi-strike emergent from physics (not a toggle)
- Constants file for scale (not comments everywhere)

**Files Modified This Session:**
- `GrowthStep.ts` - frontrunner protection, generation-based branching
- `config.ts` - adjusted parameters
- `FieldComputation.ts` - reduced downward bias
- `BranchSelection.ts` - removed verbose logging

---

## Appendix A: Physics References

See `2026-02-07_01_lightning-physics-and-math.md` for:
- Stepped leader measurements
- Charge distribution in thunderclouds
- Breakdown voltages
- Timing characteristics

Key values:
- Stepped leader step: 50m (3-200m range)
- Cloud base: 5-8 km altitude
- Main negative charge: 20-300 Coulombs
- Breakdown field: ~3 MV/m at sea level

---

## Appendix B: Existing Code Integration Points

### Where to integrate atmospheric model:

1. **BoltSimulator.ts:137** - `simulateBolt()` function
   - Create AtmosphericModel before simulation loop
   - Pass to GrowthStep

2. **GrowthStep.ts:116** - `growthStep()` function
   - Add atmospheric model parameter
   - Use for path selection and branching

3. **FieldComputation.ts:36** - `computeFieldAtPoint()` function
   - Add atmospheric field contributions
   - Weight by configured factors

4. **LightningBoltEffect.ts:53** - Main entry
   - Create AtmosphericModel
   - Pass to simulateBolt

### Existing code to potentially remove:

- `physics/AtmosphericField.ts` - Replace with new implementation
- `physics/SteppedLeader.ts` - Evaluate if needed
- `physics/ReturnStroke.ts` - Keep for animation phase
