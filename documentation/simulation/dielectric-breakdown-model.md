# Dielectric Breakdown Model (DBM)

The Dielectric Breakdown Model is the mathematical foundation for our lightning simulation. Introduced by Niemeyer, Pietronero, and Wiesmann (1984), it captures how electrical breakdown follows paths that maximize local electric field.

---

## Physical Basis

Electrical breakdown in dielectrics (like air) occurs when the local electric field exceeds a threshold. The path follows regions of maximum field intensity. The stochastic element arises from microscopic fluctuations in material properties.

---

## The Core Equation

### Growth Probability

The probability of the leader extending to candidate site `i` is:

```
P(i) = |E_i|^eta / sum_j(|E_j|^eta)
```

Where:
- `P(i)` = probability of growth to site i
- `E_i` = electric field magnitude at site i
- `eta` = growth exponent (key parameter)
- `sum_j` = sum over all candidate sites

### In Code

```typescript
function computeDBMProbabilities(
  candidates: Candidate[],
  eta: number
): number[] {
  const powers = candidates.map(c =>
    Math.pow(Math.abs(c.fieldValue), eta)
  );
  const sum = powers.reduce((a, b) => a + b, 0);

  if (sum === 0) {
    // Uniform distribution if all fields are zero
    return candidates.map(() => 1 / candidates.length);
  }

  return powers.map(p => p / sum);
}
```

---

## The Eta Parameter

The exponent `eta` is the single most important parameter for visual character.

### Effect on Behavior

| eta Value | Fractal Dimension D | Behavior | Visual Description |
|-----------|---------------------|----------|-------------------|
| eta = 0 | D = 2 | Random walk (Eden model) | Plane-filling |
| eta = 1 | D ~ 1.7 | DLA (Diffusion-Limited Aggregation) | Dense branching, bushy |
| eta = 2 | D ~ 1.5 | **Realistic lightning** | Moderate branching, natural |
| eta = 3 | D ~ 1.3 | Greedy selection | Sparse branching, straighter |
| eta = 4+ | D ~ 1.1 | Nearly deterministic | Nearly straight |

### Why eta = 2?

Measured fractal dimension of real lightning: **D ~ 1.5**

The relationship between eta and fractal dimension:
```
D ~ 2 - 0.27 * eta  (approximate, for 2D DBM, eta in range 1-4)
```

Solving for D = 1.5 gives `eta ~ 1.85-2.0`.

### Probability Ratios

With `eta = 2`, probability ratios become:
- If field A is 2x field B: P(A)/P(B) = 4x
- If field A is 1.5x field B: P(A)/P(B) = 2.25x

Higher eta = more "greedy" selection of best candidate.

---

## Field Computation

### The Full Laplace Problem

True electrostatics requires solving Laplace's equation:
```
nabla^2 phi = 0
```

With boundary conditions:
- phi = 0 on leader channel (conductor)
- phi = 0 on ground plane
- phi -> -E_0 * z as z -> infinity (uniform background field)

**This is too slow for real-time.**

### Our Approximation: Distance-Based Field

Instead of solving Laplace, we use superposition:

```typescript
function computeFieldAtPoint(
  point: Vec3,
  channelPoints: Vec3[],
  groundY: number,
  atmosphere: AtmosphericModel,
  config: FieldConfig
): number {
  // 1. Background field (uniform, pointing toward ground)
  let field = config.backgroundField;

  // 2. Channel influence (inverse distance to existing channel)
  for (const cp of channelPoints) {
    const dist = distance(point, cp);
    if (dist > config.epsilon) {
      field += config.channelInfluence / (dist + config.epsilon);
    }
  }

  // 3. Ground proximity (image charge effect)
  const groundDist = point.y - groundY;
  if (groundDist > 0) {
    field += config.groundInfluence / (groundDist + config.epsilon);
  }

  // 4. Ground charge attraction (distance-modulated)
  if (groundDist < GROUND_CHARGE_RANGE) {
    const proximityFactor = 1 - groundDist / GROUND_CHARGE_RANGE;
    const chargeValue = atmosphere.groundCharge.getValue(point);
    field += chargeValue * proximityFactor * GROUND_CHARGE_WEIGHT;
  }

  // 5. Atmospheric noise
  const noise = noise3D(point.x * scale, point.y * scale, point.z * scale);
  field *= (1 + noise * config.noiseAmplitude);

  return field;
}
```

### Why This Works

- **Channel influence**: Approximates the field enhancement near conductors
- **Ground proximity**: Approximates image charge attraction
- **Atmospheric noise**: Creates stochastic variation matching real atmosphere
- **Ground charge**: Guides final termination toward charge peaks

The approximation is physically motivated and produces visually correct results at real-time speeds.

---

## Directional Bias

The base field computation doesn't discriminate between directions. We add asymmetric bias:

```typescript
if (direction.y > 0) {
  // Heavy penalty for upward movement
  field *= 0.2;
} else {
  // Mild bonus for downward
  const downwardness = -direction.y;  // 0 to 1
  field *= 1 + downwardness * 0.15;
}
```

### Rationale

- Lightning strongly prefers downward movement
- Upward movement is rare but not impossible
- Horizontal movement is neutral
- The 0.15 factor was tuned from 0.6 (too straight) to create natural tortuosity

---

## Candidate Generation

At each step, we generate candidate directions in a cone:

```typescript
function generateCandidateDirections(
  currentDir: Vec3,
  count: number,        // 8-16 typical
  coneHalfAngle: number // PI/6 typical (30 degrees)
): Vec3[] {
  // Sample uniformly within cone around currentDir
  // Each candidate is a unit vector
}
```

### Cone Half-Angle Effects

| Angle | Effect |
|-------|--------|
| 22.5 deg (PI/8) | Tight - very directional, less natural |
| 30 deg (PI/6) | **Moderate - balanced** |
| 45 deg (PI/4) | Wide - can go sideways, chaotic |

---

## Algorithm Summary

```
1. Initialize heads at starting points (from ceiling charge peaks)
2. While not connected and not terminated:
   a. For each active head:
      - Generate N candidate directions in cone
      - Compute field at each candidate position
   b. Compute DBM probabilities across all candidates
   c. Sample winner(s) from distribution
   d. Check for branching event
   e. Update channel points, create segments
   f. Check for ground connection
   g. Filter heads by competition
   h. Increment step counter
3. Trace main channel from connection point
4. Assign depth and intensity to all segments
5. Return complete BoltGeometry
```

---

## Performance Considerations

### Globe (< 1ms budget)
- 80 max steps, 8 candidates each
- ~640 field evaluations
- ~5 channel points per evaluation
- ~3200 distance operations (trivial)

### Showcase (< 16ms budget)
- 200 max steps, 16 candidates each
- ~3200 field evaluations
- ~25 channel points per evaluation
- ~80000 distance operations
- Optional: spatial grid for O(1) nearby point lookup

---

## Connection to DLA

DBM with eta = 1 is mathematically equivalent to **Diffusion-Limited Aggregation**:

- Random walkers (representing field lines) diffuse from infinity
- They attach on contact with the structure
- Attachment probability = harmonic measure (related to electric field)

The connection provides theoretical grounding:
- DLA is well-studied with known fractal properties
- eta > 1 reduces branching below the DLA limit
- eta = 2 produces D ~ 1.5, matching lightning observations

---

## References

- Niemeyer, L., Pietronero, L., & Wiesmann, H. J. (1984). Fractal dimension of dielectric breakdown. *Physical Review Letters*, 52(12), 1033.
- Femia, N., Niemeyer, L., & Tucci, V. (1993). Fractal characteristics of electrical discharges. *Journal of Physics D*, 26(4), 619.
- Warwick DBM/DLA Reference: https://warwick.ac.uk/fac/sci/physics/research/condensedmatt/imr_cdt/students/matthew_dale/dla/
