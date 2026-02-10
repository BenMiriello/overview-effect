# Branching Mechanics

Lightning branches extensively during the stepped leader phase. Understanding why and how branching occurs is crucial for authentic simulation.

---

## Core Insight

> Branching is NOT random decoration added after the fact. It emerges from the physics of leader propagation.

Branches are failed attempts at finding ground. Multiple paths explore simultaneously; one eventually wins. The "branches" are the paths that didn't win but were real candidates during propagation.

---

## The Streamer Zone

Ahead of the stepped leader tip, a **streamer zone** extends 10-100 meters. This zone consists of:

- **Corona streamers**: Cold plasma filaments (~300-1000 K)
- **Propagation speed**: ~10^6 m/s (faster than the leader itself)
- **Function**: Provides preliminary ionization and conductivity enhancement

The streamer zone is what "feels out" available paths.

---

## Space Leaders and Branching

### The Mechanism

From atmospheric physics research (ScienceDirect):

1. **Space stems** form in the streamer zone ahead of the main channel tip
2. Some evolve into **space leaders** - ionized channels that can conduct
3. **Multiple space leaders form in parallel** during a single step
4. Space leaders connecting to the **tip** extend the main channel
5. Space leaders connecting to the **lateral surface** create branches

### Key Insight

The parallel formation of space leaders is WHY lightning both:
- **Branches** (multiple space leaders, some become branches)
- **Zigzags** (competing directions for tip extension)

---

## Branch Geometry

### Measured Angles

| Statistic | Value | Source |
|-----------|-------|--------|
| Mean angle from parent | 16-20 degrees | Optical/radar studies |
| Standard deviation | 10-15 degrees | |
| 95% range | 5-45 degrees | |

Branches are rarely perpendicular (>60 deg) or nearly parallel (<5 deg) to the parent.

### Angular Distribution

The branch angle follows approximately:

```
P(theta) ~ exp(-(theta - theta_0)^2 / 2*sigma^2) * cos(theta)
```

Where:
- `theta_0` ~ 20 degrees (mean deviation)
- `sigma` ~ 12 degrees (spread)
- `cos(theta)` factor provides slight forward bias

---

## Branch Frequency

### Spatial Distribution

| Location | Branch Frequency |
|----------|-----------------|
| Upper channel (near cloud) | 1 branch per 100-200 m |
| Lower channel (near ground) | 0.5-1 branch per 100-200 m |

Branching decreases toward ground because:
- Field becomes more directed toward ground
- Fewer competing paths remain viable
- Connection is imminent

### Total Branch Count

| Statistic | Value |
|-----------|-------|
| Typical visible branches | 10-30 |
| Exceptional cases | 100+ |

---

## Branch Attenuation with Depth

Not all branches are equal. Intensity decreases with **branch depth**:

| Depth | Intensity (relative to main) |
|-------|------------------------------|
| 0 (main channel) | 100% |
| 1 (primary branch) | 50-70% |
| 2 (secondary branch) | 25-40% |
| 3+ (deeper branches) | Rarely visible |

This creates the characteristic appearance where the main channel and primary branches are prominent, with finer structure fading into the background.

---

## Branch Length Distribution

Branch lengths follow approximately exponential distribution:

```
P(length) ~ exp(-length / mean_length)
```

This means:
- **Many short branches** (exploration that quickly terminated)
- **Few long branches** (paths that almost became main channel)
- **Mean branch length / total flash length** ~ 0.1 to 0.3

### Implications

A branch that extends 30% of the total flash length was a serious competitor for main channel status. Most branches are much shorter.

---

## Main Channel Determination

The "main channel" does NOT exist until ground connection:

1. **During leader phase**: All active paths are potential main channels
2. **At connection**: First path to reach ground becomes main channel
3. **After connection**: Other paths are retroactively classified as "branches"

This is why authentic simulation requires:
- Multiple heads growing simultaneously
- Competition based on progress
- Winner determined by physics, not predetermined

---

## Return Stroke and Branches

During the return stroke:

| What Happens | Where |
|--------------|-------|
| Full illumination | Main channel |
| Brief illumination | Branches connected to main channel |
| No illumination | Terminated branches |

The return stroke current flows up the main channel. Branches briefly illuminate as current redistributes, but they don't carry the main current.

In photographs:
- **Stepped leader**: Complex branching tree visible
- **Return stroke**: Single bright channel (branching structure hidden by brightness)

---

## Simulation Approach: Event-Based Branching

Rather than checking each head for branching probability every step, we use **global branching events**:

### Why Event-Based?

Per-head probability creates:
- Too many early branches (more heads = more rolls)
- Exponential explosion of branches at start
- Artificial branch distribution

Event-based branching creates:
- Constant branch rate over time
- More natural spatial distribution
- "Bursts" of branching via noise modulation

### The Algorithm

```
Each step:
  1. Calculate base branch probability
  2. Modulate by spatial noise (creates burstiness)
  3. If event occurs, determine count (1-3 branches)
  4. Select which heads branch (random from eligible)
  5. Create branch segments from selected heads
```

Eligibility requires `generation < 4` to prevent infinite recursion.

---

## Competition-Based Death

Branches (and competing leaders) die based on **competition**, not timers:

```
survivalProb = base - lagRatio * penalty + progressRatio * bonus
```

Where:
- `base` ~ 0.97 (most survive each step)
- `lagRatio` = how far behind the leader
- `progressRatio` = how much progress made
- `penalty` ~ 0.3, `bonus` ~ 0.1

This creates:
- Leader always survives
- Stragglers have increasing death probability
- Progress partially compensates for lag

---

## Visual Quality Checklist

Authentic branching should show:

- [ ] Multiple paths visible during leader phase
- [ ] Varied branch lengths (many short, few long)
- [ ] Natural angle distribution (~20 deg mean)
- [ ] Branching throughout descent (not just at start)
- [ ] Clear winner emergence before ground connection
- [ ] Only main channel illuminates during return stroke
- [ ] Connected branches briefly flash during return stroke

---

## References

- ScienceDirect: "Better Understanding of Stepped Leaders" (space leader mechanism)
- AGU: "Numerical Simulation of Stepping and Branching"
- Nature: "Channel Branching and Zigzagging"
- Photography analysis of leader vs return stroke appearance
