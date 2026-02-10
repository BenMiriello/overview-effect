# Emergent vs Configured Behaviors

A key design principle: behaviors should emerge from physics where possible, rather than being explicitly configured. This document explains what emerges and what's configured.

---

## The Principle

> "Multi-strike should be emergent, not a setting."

The user explicitly stated that authentic lightning behavior should arise from the physics simulation, not from toggles and sliders. This creates more natural variety and makes the simulation educational rather than decorative.

---

## Emergent Behaviors

### Number of Leaders

**NOT configured.** Emerges from ceiling charge distribution.

- If ceiling charge has 3 strong peaks, 3 leaders spawn
- If peaks are weak, fewer leaders qualify (intensity threshold)
- Varying seed creates varying leader count

**Implementation:**
```typescript
const startingPoints = deriveStartingPoints(ceilingCharge);
// startingPoints.length determines leader count
```

### Which Path Becomes Main Channel

**NOT configured.** First to reach ground wins.

- All leaders compete equally
- Progress toward ground determines survival
- Connection segment determines main channel
- Branches are paths that didn't win

**Implementation:**
```typescript
if (isConnected && !alreadyConnected) {
  connectionSegmentId = thisSegment;
  // This path wins, traced back to become main channel
}
```

### Branch Length Distribution

**NOT configured.** Emerges from survival probability.

- Longer-surviving branches went further
- Exponential length distribution arises naturally
- Many short branches, few long ones

**Implementation:**
```typescript
survivalProb = base - lagPenalty + progressBonus;
// Heads that fall behind die probabilistically
```

### Termination Point

**NOT configured.** Attracted to ground charge peaks.

- Ground charge influences path near ground
- Leader "seeks" high-charge regions
- Different seeds = different termination points

**Implementation:**
```typescript
if (groundDist < GROUND_CHARGE_RANGE) {
  field += groundCharge.getValue(point) * proximityFactor * WEIGHT;
}
```

### Tortuosity (Path "Wiggliness")

**Partially emergent.** Arises from:
- Atmospheric noise (configured amplitude)
- Candidate selection probability (eta configured)
- Cone angle (configured)

The specific path shape is not configured, but the degree of tortuosity is tunable.

---

## Configured Behaviors

### Step Length

**Configured.** Sets simulation resolution.

- Globe: 0.02 units (coarser, fewer segments)
- Showcase: 0.008 units (finer, more segments)

Maps to real ~50m stepped leader steps.

### Branch Probability (Event-Based)

**Configured.** Sets how often branching events occur.

But: *which* heads branch and *how many* branches is random.

```typescript
branchProbAtStart: 0.03,
branchProbAtEnd: 0.06,
// Rate increases toward ground, but specific branches are random
```

### Maximum Branch Depth

**Configured.** Limits recursion.

- Depth 4 typical for showcase
- Prevents infinite sub-branching
- Natural attenuation also limits depth (intensity decay)

### Animation Timing

**Configured.** How fast animation plays.

- Leader duration, return stroke speed, interstroke interval
- These are artistic/educational choices
- Real times are too fast to observe (~20ms leader, ~100us stroke)

### Detail Level

**Configured.** Globe vs Showcase.

- Affects step length, segment count, branch depth
- Performance vs quality tradeoff

---

## The Gray Area

Some behaviors are emergent from configured parameters:

### Branching Pattern

- Event probability is configured
- But spatial distribution emerges from noise modulation
- "Burstiness" is emergent within configured constraints

### Competition Dynamics

- Survival formula parameters are configured
- But which specific paths survive is emergent
- Leader head count fluctuates naturally

### Charge Distribution

- Cell count range is configured
- But specific cell positions are random
- Field shape emerges from random seed

---

## Why This Matters

### Educational Value

When behavior emerges from physics:
- Viewers learn about real lightning mechanics
- "Why did it do that?" has a physics answer
- Not just "that's the random variation"

### Natural Variety

Configuration-based systems often feel repetitive:
- Same branch angles every time
- Predictable patterns
- "Procedural but not organic"

Emergent systems create genuine variety:
- Each bolt genuinely different
- Surprising but explainable behaviors
- Feels like observing nature

### Maintainability

Fewer configuration knobs:
- Less to tune and break
- Physics constraints prevent invalid states
- System is internally consistent

---

## User's Words

From the user's requirements:

> "they should all be seeking ground equally, some die, others go further"

> "ONLY once one finds ground it becomes the main one"

> "multiple starting points from charge distribution (automatic)"

> "multi-strike should be emergent, not a setting"

> "shouldn't be hitting caps - natural dynamics should create distribution"

These requirements directly shaped the emergent approach.

---

## Trade-offs

### Less Direct Control

Can't configure "exactly 3 branches" - you configure branch probability and let physics decide.

### Harder to Debug

When something looks wrong, need to understand the physics to fix it.

### Performance Variance

Emergent head counts can spike, requiring safety limits.

### Mitigation

Safety caps exist but shouldn't normally trigger:
```typescript
maxActiveHeads: 20,  // Safety, not design driver
maxSegments: 800,    // Performance bound
```

If these caps are hit frequently, the physics parameters need tuning.
