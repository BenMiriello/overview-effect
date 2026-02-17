# DBM Parameter Tuning Guide

Hard-earned knowledge about how simulation parameters affect visual output.

## Field Parameters

### channelInfluence

Controls attraction/repulsion to existing channel segments.

| Value | Effect |
|-------|--------|
| Positive (0.5) | Self-attraction - bolt clusters near itself, creates "jellyfish" appearance |
| Zero | No self-influence - cleanest paths |
| Negative (-0.1) | Self-repulsion - spreads out, avoids existing path |

**Learned the hard way**: At 0.5 with epsilon 0.005, a candidate 0.05 units from channel gets field contribution of ~9, overwhelming all other influences. Probability of staying near existing channel: ~97.5%.

**Recommendation**: Set to 0 unless you specifically want clustering.

### groundInfluence

Attraction toward ground. Higher = more vertical paths.

**Gotcha**: This is distance-based, not direction-based. All candidates from the same point have the same ground distance, so it doesn't differentiate between them. The `directional bias` multiplier (in FieldComputation) is what actually discriminates.

### Directional Bias Multiplier

```javascript
field *= 1 + downwardness * BIAS;
```

| BIAS | Effect |
|------|--------|
| 0.15 | Too weak - sideways movement common |
| 0.6 | Strong - down is 2.56x more likely than sideways (with eta=2) |
| 1.0+ | Very strong - nearly always goes down |

**With eta=2**: probability ratio = (1+bias)^2 : 1

## Growth Parameters

### eta (DBM exponent)

Controls how "greedy" path selection is.

| Value | Effect |
|-------|--------|
| 1.0 | Linear - more random, many detours |
| 2.0 | Quadratic - balanced (recommended) |
| 3.0+ | Highly greedy - always picks best candidate |

### coneHalfAngle

How far candidates can deviate from current direction.

| Value | Effect |
|-------|--------|
| PI/4 (45deg) | Wide - can go sideways, creates chaos at top |
| PI/6 (30deg) | Moderate - some deviation, more directional |
| PI/8 (22.5deg) | Tight - very directional, less natural |

**Recommendation**: PI/6 for balanced appearance.

### mainChannelJitter & jitterDecayRate

```javascript
jitter = mainChannelJitter * pow(jitterDecayRate, step)
```

**Critical bug found**: `jitterDecayRate: 1.0` means NO DECAY (1^anything = 1).

| jitterDecayRate | Effect |
|-----------------|--------|
| 1.0 | Constant jitter forever (bug!) |
| 0.97 | Decays to ~5% by step 100 |
| 0.95 | Decays to ~0.6% by step 100 |

**Recommendation**: 0.97 with base jitter 1.5

## Branching Parameters

### maxBranchDepth & baseBranchProb

These enable real-time branching during simulation.

**Key insight**: Setting `maxBranchDepth: 0` disables real-time branching entirely, relying on post-process. This creates "decoration" appearance, not genuine seeking.

For authentic seeking:
- `maxBranchDepth: 2-3`
- `baseBranchProb: 0.05-0.08`
- Must ALSO enable branch continuation (see GrowthStep.ts line ~233)

### branchSurvivalDecay (new parameter)

Controls how long branches live before dying.

```javascript
survivalProb = pow(branchSurvivalDecay, depth)
```

| Value | Depth 1 | Depth 2 | Depth 3 |
|-------|---------|---------|---------|
| 0.9 | 90%/step | 81%/step | 73%/step |
| 0.85 | 85%/step | 72%/step | 61%/step |
| 0.8 | 80%/step | 64%/step | 51%/step |

Creates natural exponential branch length distribution.

## Animation Parameters

### Leader brightness

```javascript
const b = Math.max(MIN, 1 - age * DECAY) * intensity;
```

| MIN | Effect |
|-----|--------|
| 0.3 | Too visible - "thick snake" appearance |
| 0.05 | Good - faint trail, visible seeking |
| 0.02 | Very faint - only tip visible |

**Recommendation**: MIN=0.02-0.05, with bright tip (front 3-5 segments at 0.8-1.0)

## Globe vs Showcase

| Parameter | Globe | Showcase | Why |
|-----------|-------|----------|-----|
| stepLength | 0.02 | 0.008 | Less detail for small display |
| maxSteps | 80 | 200 | Fewer steps = faster |
| candidateCount | 8 | 16 | Less computation |
| linewidth | 2 | 4 | Scale appropriate |
| leaderDuration | 200ms | 800ms | Less scrutiny time |
