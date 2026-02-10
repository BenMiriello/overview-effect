# Authentic Lightning Simulation: Implementation Plan

## Overview

This document outlines how to make the lightning simulation authentically represent real stepped leader physics, rather than the current approach of drawing a predetermined path with decorative branches.

**Core insight**: Real lightning doesn't have a "main channel" until ground connection. During the stepped leader phase, multiple paths explore simultaneously. One path reaches ground first; that becomes the main channel. Others become branches. The "seeking" appearance comes from watching this competition play out.

**Current state**: Single head grows, one candidate wins per step, branches added post-hoc as decoration.

**Target state**: Multiple heads grow in parallel, branches ARE the exploration, first to ground wins.

---

## The Real Physics

### Stepped Leader Mechanism

From [weather.gov](https://www.weather.gov/safety/lightning-science-initiation-stepped-leader) and research papers:

1. **Single origin**: Leader starts from one point in the cloud
2. **Steps**: Advances in ~50m segments, each taking ~1 microsecond
3. **Pauses**: 20-50 microseconds between steps
4. **Limited sensing**: Tip only "sees" charges within ~50m, making path stochastic

### Space Leaders & Branching

From [ScienceDirect research](https://www.sciencedirect.com/science/article/abs/pii/S0378779622002681):

1. **Space stems** form in the streamer zone ahead of the main channel tip
2. Some evolve into **space leaders** (ionized channels that can conduct)
3. **Multiple space leaders form in parallel** during a single step
4. Space leaders connecting to the **tip** extend the channel (continuation)
5. Space leaders connecting to **lateral surface** create branches
6. This parallel formation is why lightning branches and zigzags

**Key insight**: Branching isn't random decoration - it's the natural result of multiple space leaders forming and some connecting laterally rather than at the tip.

### Main Channel Determination

The "main channel" doesn't exist until ground connection:
- All branches are potential main channels
- The path that reaches ground first wins
- That path becomes the main channel for the return stroke
- Other paths remain as branches (they explored but didn't win)

### Return Stroke

- Occurs ONLY when connection to ground is made
- Travels UP from ground at ~1/3 speed of light (~100,000 km/s)
- **EXTREMELY bright** - this is the main visible flash
- Illuminates main channel + branches connected to it
- Takes ~100 microseconds

### Visual Characteristics

From [photography analysis](https://www.dblanchard.net/blog/2010/07/lightning-images-return-strokes-and-stepped-leaders/) and [timing studies](https://ermsta.com/posts/20200707):

- **Stepped leader**: Relatively dim, extensive branching visible, 10-20ms duration
- **Return stroke**: Single bright channel, ~100 microseconds, no visible branching structure
- **Brightness ratio**: Return stroke is orders of magnitude brighter than leader

---

## Current Architecture Analysis

### What We Have

```
BoltSimulator.simulateBolt()
  └─> GrowthStep.growthStep() [called in loop]
        ├─> generateCandidateDirections() - creates N candidates
        ├─> computeFieldAtPoint() - evaluates each candidate
        ├─> computeDBMProbabilities() - converts fields to probabilities
        ├─> selectForHead() - picks winner + potential branch indices
        └─> Creates segments, updates heads
  └─> addPostProcessBranches() [after main loop]
        └─> addBranchesRecursive() - decorative branches on main channel
```

### The Problem

1. **Single head paradigm**: Only one head continues at depth 0
2. **Instant branch death**: Branch heads (depth > 0) are not added to `newHeads` (line 233 in GrowthStep.ts)
3. **Post-process decoration**: Branches added AFTER path is complete, not during exploration
4. **Animation reveals predetermined path**: No actual "seeking" - just revealing what's already computed

### The Fix Location

In `GrowthStep.ts`, around line 233:
```javascript
} else if (head.depth === 0) {
  newHeads.push({...});  // Only main channel continues
}
```

Currently: Only depth 0 heads continue. Depth > 0 heads die after one segment.

Should be: Heads at any depth can continue, with probability of death increasing with depth.

---

## Detailed Implementation Plan

### Phase 1: Enable Authentic Branching

#### 1.1 Modify GrowthStep.ts - Let Branches Live

**File**: `src/effects/LightningBoltEffect/simulation/GrowthStep.ts`

**Current behavior** (lines 230-243):
```javascript
if (isConnected) {
  connected = true;
  connectionSegmentId = primarySegId;
} else if (head.depth === 0) {
  // Only main channel continues
  newHeads.push({...});
}
```

**New behavior**:
```javascript
if (isConnected) {
  connected = true;
  connectionSegmentId = primarySegId;
} else {
  // All heads can continue, with depth-based survival probability
  const survivalProb = Math.pow(config.branchSurvivalDecay, head.depth);
  if (head.depth === 0 || state.rng.next() < survivalProb) {
    newHeads.push({
      id: state.nextHeadId++,
      position: jitteredPosition,
      direction: primary.direction,
      depth: head.depth,
      parentSegmentId: primarySegId,
      stepIndex: state.currentStep,
    });
  }
}
```

**New config parameter**: `branchSurvivalDecay: number` (e.g., 0.85)
- Depth 0: 100% survival (main channel always continues)
- Depth 1: 85% survival per step
- Depth 2: 72% survival per step
- Depth 3: 61% survival per step

This creates exponentially decaying branch lengths naturally.

#### 1.2 Modify GrowthStep.ts - Branch Spawning Creates Continuing Heads

**Current behavior** (lines 246-268): Branch segments created but no heads added.

**New behavior**: When a branch spawns, add it as a continuing head:
```javascript
for (const branchIdx of selection.branchIndices) {
  const branchCandidate = headCandidates[branchIdx];
  // ... create segment ...

  // NEW: Add branch as continuing head (can keep growing)
  const branchSurvivalProb = Math.pow(config.branchSurvivalDecay, head.depth + 1);
  if (state.rng.next() < branchSurvivalProb) {
    newHeads.push({
      id: state.nextHeadId++,
      position: branchPos,
      direction: branchCandidate.direction,
      depth: head.depth + 1,
      parentSegmentId: branchSegId,
      stepIndex: state.currentStep,
    });
  }
}
```

#### 1.3 Update Config - Enable Real-Time Branching

**File**: `src/effects/LightningBoltEffect/simulation/config.ts`

```javascript
// DETAIL_PRESETS[DetailLevel.SHOWCASE]
maxBranchDepth: 3,           // Was 0, allow 3 levels of branches
baseBranchProb: 0.08,        // Was 0, probability of branch per candidate
branchSurvivalDecay: 0.85,   // NEW: survival probability decay per depth
maxBranchesPerStep: 2,       // Limit concurrent branch spawns

// Reduce or eliminate post-process branching (no longer needed)
postBranchProb: 0.02,        // Was 0.08, minimal touchup only
```

#### 1.4 Update Types - Add New Config Field

**File**: `src/effects/LightningBoltEffect/simulation/types.ts`

Add to `SimulationConfig`:
```javascript
branchSurvivalDecay: number;  // 0-1, probability multiplier per depth level
```

### Phase 2: Fix Main Channel Determination

#### 2.1 Any Head Can Win

**File**: `src/effects/LightningBoltEffect/simulation/GrowthStep.ts`

Currently, connection is checked for all heads but only depth 0 matters for continuation.

**New behavior**: Track which head connected first, regardless of depth:
```javascript
// In growthStep, after checking isConnected:
if (isConnected && !connected) {
  connected = true;
  connectionSegmentId = primarySegId;
  // Note: This might be a branch (depth > 0) that reached ground first
}
```

#### 2.2 Trace Main Channel From Winner

**File**: `src/effects/LightningBoltEffect/simulation/BoltSimulator.ts`

The existing `traceMainChannel` function already traces from the connection segment back to root. It will work correctly regardless of which head (depth 0 or branch) made the connection.

No changes needed here - the architecture already supports this.

### Phase 3: Animation Fixes

#### 3.1 Dim Leader Phase Dramatically

**File**: `src/effects/LightningBoltEffect/animation/BoltAnimator.ts`

**Current** (leaderState):
```javascript
const b = Math.max(0.3, 1 - age * 0.02) * seg.intensity;
```

**New**:
```javascript
// Much dimmer trail, bright tip only
const tipDistance = 5; // Only last 5 segments are "tip"
const isTip = age < tipDistance;
const tipBrightness = isTip ? (1 - age / tipDistance) * 0.8 + 0.2 : 0;
const trailBrightness = Math.max(0.02, 0.15 * Math.exp(-age * 0.1));
const b = (isTip ? tipBrightness : trailBrightness) * seg.intensity;
```

This creates:
- Bright moving tip (front ~5 segments)
- Rapidly fading trail behind
- Nearly invisible segments far behind

#### 3.2 Branches Animate With Main Path

**Current behavior**: All segments with `stepIndex <= targetStep` are visible.

This already works correctly for branches - they'll appear at their creation step along with the main path segments created at the same step.

No changes needed - branches will naturally appear as the animation progresses.

#### 3.3 Return Stroke Only Illuminates Winner Path

**Current** (returnStrokeState): All segments visible, main channel brightest.

**New behavior**: Only main channel + directly connected branches illuminate:
```javascript
// In returnStrokeState:
for (const seg of this.geometry.segments) {
  if (seg.isMainChannel) {
    // Full return stroke brightness
    const decay = 1 - (litCount - i) * 0.01;
    brightness.set(seg.id, Math.max(0.8, decay) * peak);
  } else if (this.isConnectedToMainChannel(seg)) {
    // Branches connected to main channel get partial illumination
    const parentBrightness = brightness.get(seg.parentSegmentId!) ?? 0;
    brightness.set(seg.id, parentBrightness * 0.4);
  } else {
    // Unconnected branches fade out during return stroke
    brightness.set(seg.id, 0.02);
  }
}
```

Need helper: `isConnectedToMainChannel(seg)` - traverse parentSegmentId chain to see if it reaches a main channel segment.

### Phase 4: Globe vs Showcase Tuning

#### Config Differences

| Parameter | Globe | Showcase | Rationale |
|-----------|-------|----------|-----------|
| `maxBranchDepth` | 2 | 3 | Less detail for small display |
| `baseBranchProb` | 0.05 | 0.08 | Fewer branches for cleaner look |
| `branchSurvivalDecay` | 0.8 | 0.85 | Shorter branches |
| `leaderDuration` | 200ms | 800ms | Faster for less scrutiny |
| `linewidth` (depth 0) | 2 | 4 | Thinner for scale |

These are already config-driven, just need value tuning.

### Phase 5: Remove Debug Logging

After implementation is verified, remove the console.log statements added for debugging:
- `LightningBoltEffect.ts` - simulation result logging
- `BoltAnimator.ts` - phase transition logging
- `LightningController.tsx` - speed logging

---

## Implementation Order

1. **Types**: Add `branchSurvivalDecay` to SimulationConfig
2. **Config**: Update presets with new branching values
3. **GrowthStep**: Enable branch continuation (Phase 1.1, 1.2)
4. **Test**: Verify branches now grow multiple segments
5. **Animation**: Fix leader brightness (Phase 3.1)
6. **Test**: Verify seeking appearance during leader phase
7. **Animation**: Fix return stroke (Phase 3.3)
8. **Test**: Verify only main path flashes
9. **Tuning**: Adjust config values for best appearance
10. **Cleanup**: Remove debug logging

---

## Verification Criteria

### Visual Checks

1. **Leader phase**: Multiple visible paths exploring, not just one snake
2. **Seeking appearance**: Looks like genuine exploration, not predetermined
3. **Branch variety**: Some short, some long, natural exponential distribution
4. **Tip brightness**: Clear bright front, dim trail
5. **Return stroke**: Only winning path flashes bright
6. **Flash timing**: Flash happens at ground connection, not before

### Console Checks (during development)

1. `depthCounts` should show significant counts at depth 1, 2, 3
2. Multiple heads should be active simultaneously (add logging if needed)
3. Main channel should sometimes be traced through depth > 0 segments

---

## Alternatives Considered

### Alternative A: Store and Render Failed Candidates

**Approach**: At each step, store ALL candidates evaluated (not just winner). During animation, briefly flash failed candidates as "probes" before showing winner.

**Pros**:
- Most authentic to physics (space leaders)
- No simulation changes needed

**Cons**:
- Significant memory increase (16 candidates x 200 steps = 3200 extra positions)
- Complex animation logic for brief flashes
- May look too busy/chaotic

**Why rejected**: More complex, higher memory, and real-time branching achieves similar visual effect with less overhead.

### Alternative B: Post-Process Branching Only (Current)

**Approach**: Keep single-head simulation, improve post-process branch quality.

**Pros**:
- Minimal code changes
- Guaranteed main path reaches ground

**Cons**:
- Never looks like seeking (fundamental limitation)
- Branches are obviously decorative
- Can't show "exploration" during leader phase

**Why rejected**: Can't achieve authentic seeking appearance.

### Alternative C: Multiple Independent Starting Heads

**Approach**: Start with 3-5 heads from cloud, let them race independently.

**Pros**:
- Guaranteed multiple exploring paths
- Simple conceptually

**Cons**:
- Not physically accurate (lightning has single origin)
- Would look like multiple separate bolts, not branching tree
- Branches wouldn't share common root

**Why rejected**: Not authentic to physics.

### Alternative D: Visual Tricks Only

**Approach**: Keep current simulation, just dramatically change brightness/animation.

**Pros**:
- No simulation changes
- Quick to implement

**Cons**:
- Predetermined path still visible if scrutinized
- Branches still obviously stubby decorations
- Doesn't address root cause

**Why rejected**: Masks problem rather than fixing it.

---

## Context & Reasoning

### Why Real-Time Branching Is The Right Approach

1. **Matches physics**: Space leaders forming in parallel IS what creates branches
2. **Uses existing code**: The branch spawning infrastructure exists, just disabled
3. **Natural exploration**: Multiple heads = genuine seeking appearance
4. **Emergent main channel**: First to ground wins, just like reality
5. **Exponential lengths**: Survival probability creates natural length distribution

### Why We Disabled It Before

Original implementation had issues:
1. Branch heads were killed instantly (no continuation)
2. This led to enabling post-process as compensation
3. Post-process created "decoration" appearance

The code supported branching but the continuation logic was missing.

### Animation Considerations

**Leader phase brightness** is critical:
- If all segments are equally visible, predetermined path is obvious
- If only tip is visible, exploration isn't shown
- Solution: Bright tip + dim but visible trail, with branches at same brightness level as main path segments at same age

**Return stroke exclusivity** matters:
- In reality, only main channel + connected branches illuminate
- Showing all branches at full brightness during return stroke looks wrong
- Solution: Trace connectivity, illuminate only connected segments

### Performance Considerations

More active heads = more computation per step:
- Worst case: 2^depth heads at maximum branching
- Mitigated by: `branchSurvivalDecay` killing branches probabilistically
- Mitigated by: `maxBranchesPerStep` limiting spawns per step
- Mitigated by: `maxBranchDepth` capping recursion

Expected active heads at any step: 3-8 (manageable)

---

## References

- [NWS Stepped Leader Science](https://www.weather.gov/safety/lightning-science-initiation-stepped-leader)
- [ScienceDirect: Better Understanding of Stepped Leaders](https://www.sciencedirect.com/science/article/abs/pii/S0378779622002681)
- [AGU: Numerical Simulation of Stepping and Branching](https://agupubs.onlinelibrary.wiley.com/doi/full/10.1029/2019JD031360)
- [Nature: Channel Branching and Zigzagging](https://www.nature.com/articles/s41598-017-03686-w)
- [Photography: Leaders vs Return Strokes](https://www.dblanchard.net/blog/2010/07/lightning-images-return-strokes-and-stepped-leaders/)
- [Rolling Shutter Timing Analysis](https://ermsta.com/posts/20200707)
- [UNC Lightning Rendering Paper](http://gamma.cs.unc.edu/LIGHTNING/)
- [SideFX DBM Implementation](https://www.sidefx.com/forum/topic/52813/)

See `./notes/tmp/bibliography.md` for full resource list with descriptions.
