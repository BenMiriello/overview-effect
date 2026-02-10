# Parameter Tuning

Hard-won knowledge about how simulation parameters affect visual output. This document captures lessons learned from extensive testing.

---

## Field Parameters

### channelInfluence

Controls self-attraction/repulsion to existing channel segments.

| Value | Effect |
|-------|--------|
| Positive (0.5) | Self-attraction - bolt clusters near itself, "jellyfish" appearance |
| Zero | No self-influence - cleanest paths |
| Negative (-0.1) | Self-repulsion - spreads out, avoids existing path |

**Critical bug discovered:**
With channelInfluence = 0.5 and epsilon = 0.005:
- Candidate 0.05 units from channel → field contribution ~9
- This overwhelms all other influences
- Probability of staying near existing channel: ~97.5%

**Recommendation:** Set to 0 unless specifically wanting clustering.

### groundInfluence

Attraction toward ground. Higher = more vertical paths.

**Gotcha:** This is distance-based, not direction-based. All candidates from the same head have the same ground distance, so it doesn't differentiate between them. The directional bias multiplier is what discriminates.

### Directional Bias

```javascript
if (direction.y > 0) {
  field *= 0.2;  // Heavy penalty for upward
} else {
  const downwardness = -direction.y;
  field *= 1 + downwardness * BIAS;
}
```

| BIAS | Effect |
|------|--------|
| 0.15 | Too weak - sideways movement common |
| 0.6 | Strong - down is 2.56x more likely than sideways |
| 1.0+ | Very strong - nearly always goes down |

**Evolution:**
- Started at 0.6 → paths too straight
- Reduced to 0.15 → better tortuosity
- Added upward penalty (0.2 multiplier) → allows occasional upward but rare

**With eta=2:** Probability ratio = (1+bias)^2 : 1

---

## Growth Parameters

### eta (DBM exponent)

Controls how "greedy" path selection is.

| Value | Effect |
|-------|--------|
| 1.0 | Linear - more random, many detours |
| 2.0 | Quadratic - balanced (**recommended**) |
| 3.0+ | Highly greedy - always picks best candidate |

eta = 2.0 gives fractal dimension ~1.5, matching real lightning.

### coneHalfAngle

How far candidates can deviate from current direction.

| Value | Effect |
|-------|--------|
| PI/8 (22.5 deg) | Tight - very directional, less natural |
| PI/6 (30 deg) | Moderate - balanced (**recommended**) |
| PI/4 (45 deg) | Wide - can go sideways, chaotic at top |

### mainChannelJitter & jitterDecayRate

```javascript
jitter = mainChannelJitter * pow(jitterDecayRate, step)
```

**Critical bug found:**
- `jitterDecayRate: 1.0` means NO DECAY (1^anything = 1)
- Constant jitter forever!

| jitterDecayRate | Effect |
|-----------------|--------|
| 1.0 | Constant jitter (bug!) |
| 0.97 | Decays to ~5% by step 100 |
| 0.95 | Decays to ~0.6% by step 100 |

**Recommendation:** jitterDecayRate = 0.97 with base jitter = 1.5

---

## Branching Parameters

### maxBranchDepth & baseBranchProb

These enable real-time branching during simulation.

**Key insight:** Setting `maxBranchDepth: 0` disables real-time branching entirely. The simulation only relies on post-process branching, creating a "decoration" appearance.

For authentic seeking:
- maxBranchDepth: 2-3
- baseBranchProb: 0.03-0.06 (was higher, tuned down)

### Event-Based Branching

Changed from per-head probability to global events:

**Problem with per-head:**
- More active heads = more branch attempts
- Exponential explosion at start
- Branches clustered at top

**Solution: Event-based**
```javascript
// Once per step, roll for branch event
const eventProb = baseBranchProb * noiseFactor;
if (rng.next() < eventProb) {
  // Select random eligible heads to branch
}
```

Creates even distribution over time.

### branchProbAtStart vs branchProbAtEnd

```javascript
branchProbAtStart: 0.03,
branchProbAtEnd: 0.06,
```

Higher probability near ground because:
- Electric field intensifies near ground
- Real lightning branches more near ground
- Creates natural distribution

---

## Animation Parameters

### Leader Brightness

```javascript
const b = Math.max(MIN, 1 - age * DECAY) * intensity;
```

| MIN | Effect |
|-----|--------|
| 0.3 | Too visible - "thick snake" appearance |
| 0.05 | Good - faint trail, visible seeking |
| 0.02 | Very faint - only tip visible |

**Recommendation:** MIN = 0.02-0.05 with bright tip (front 3-5 segments at 0.8-1.0)

### Animation Speed

The speed parameter (1.0 default) scales animation timing:
- speed = 2.0 → twice as fast
- speed = 0.5 → half speed

Separate from physics, purely for viewing preference.

---

## Globe vs Showcase

| Parameter | Globe | Showcase | Why |
|-----------|-------|----------|-----|
| stepLength | 0.02 | 0.008 | Less detail for small display |
| maxSteps | 80 | 200 | Fewer steps = faster |
| candidateCount | 8 | 16 | Less computation |
| maxBranchDepth | 2 | 4 | Less complexity |
| maxSegments | 100 | 800 | Performance bound |

---

## Debugging Techniques

### Branch Distribution

Log when branches are created:
```javascript
console.log('[Simulation] Branch segments by step range:', stepBuckets);
// Example: { "0-9": 12, "10-19": 8, "20-29": 11, ... }
```

Even distribution = good. Front-loaded = problem.

### Active Head Count

Track heads over time:
```javascript
console.log(`[Step ${step}] Active heads: ${heads.length}`);
```

Should fluctuate naturally, not explode or collapse.

### Competition Dynamics

Log survival rates:
```javascript
console.log(`[Step] ${beforeCount} heads -> ${afterCount} survived`);
```

Some attrition expected, but not mass extinction.

---

## Common Failure Modes

### "Jellyfish" - Bolt clusters on itself
- Cause: channelInfluence too high
- Fix: Set to 0

### "Straight Arrow" - No tortuosity
- Cause: directional bias too strong OR coneHalfAngle too narrow
- Fix: Reduce bias to 0.15, use PI/6 cone

### "Explosion at Top" - All branches at start
- Cause: Per-head branching with many heads
- Fix: Use event-based branching

### "Constant Width" - Jitter doesn't decay
- Cause: jitterDecayRate = 1.0
- Fix: Use 0.97

### "Too Few Branches" - Sparse appearance
- Cause: maxBranchDepth = 0 or baseBranchProb too low
- Fix: Enable real-time branching, increase prob

### "Performance Issues" - Frame drops
- Cause: Too many segments or complex field computation
- Fix: Reduce maxSteps, use spatial grid, check segment count
