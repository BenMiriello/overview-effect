# Lightning Physics and Mathematical Models for Simulation

A comprehensive technical reference for implementing physically-based lightning rendering.

---

## Table of Contents

1. [Real Lightning Physics](#1-real-lightning-physics)
2. [Mathematical Models for Simulation](#2-mathematical-models-for-simulation)
3. [Key Formulas in Mathematical Notation](#3-key-formulas-in-mathematical-notation)
4. [User-Tunable Parameters](#4-user-tunable-parameters)
5. [Simplification Strategy for Detail Levels](#5-simplification-strategy-for-detail-levels)
6. [References](#6-references)

---

## 1. Real Lightning Physics

### 1.1 The Complete Cloud-to-Ground Discharge Sequence

A typical negative cloud-to-ground (CG) lightning flash involves multiple distinct phases, each with characteristic timescales and physical mechanisms.

#### Charge Separation

Before any discharge can occur, charge separation must develop within the cloud. The primary charging mechanism is the **non-inductive ice-graupel collision process**:

- Smaller ice crystals acquire positive charge and are carried upward by updrafts
- Larger graupel particles acquire negative charge and fall due to gravity
- This creates the classical tripole structure: main negative charge center at -10 to -25°C (typically 5-8 km altitude), main positive center above, and a smaller lower positive charge center

**Typical charge magnitudes:**
- Main negative charge center: 20-300 Coulombs
- Total charge neutralized per flash: 1-5 C (typical), up to 300+ C (exceptional)
- Electric field at ground before discharge: 5-20 kV/m
- Field at cloud base: 100-400 kV/m
- Breakdown field in air at sea level: ~3 MV/m (but reduces at altitude due to lower density)

#### Stepped Leader Initiation

The discharge begins with a **preliminary breakdown** process inside the cloud, typically lasting 1-10 ms. This creates the conditions for stepped leader initiation.

**Initiation characteristics:**
- Occurs near the boundary of the main negative charge region
- Initial current pulses: 1-10 kA
- Duration of preliminary breakdown: 1-10 ms
- Altitude of initiation: 4-8 km (typical for temperate latitudes)

#### Stepped Leader Propagation

The **stepped leader** is the most visually distinctive and simulation-relevant phase. It propagates in discrete jumps toward ground.

**Measured values (from Rakov & Uman 2003, Dwyer & Uman 2014):**

| Parameter | Typical Value | Range |
|-----------|---------------|-------|
| Step length | 50 m | 3-200 m |
| Pause time between steps | 50 μs | 20-100 μs |
| Step duration | 1 μs | 0.5-2 μs |
| Average propagation speed | 2 × 10⁵ m/s | 1-20 × 10⁵ m/s |
| Total duration (cloud to ground) | 20-30 ms | 10-100 ms |
| Leader current (average) | 100-200 A | 50-500 A |
| Leader current (step tip) | 1-5 kA | |
| Channel potential | 50-100 MV | |
| Leader charge deposited | 3-5 C | 1-20 C |
| Channel radius | 1-10 cm | |
| Temperature | 10,000-30,000 K | (estimated) |

**Step mechanics:**
The stepping process involves:
1. Electric field intensification at the leader tip
2. Formation of a **space stem** (luminous region) 10-100 m ahead of the tip
3. Rapid connection between leader tip and space stem
4. Pause while ionization wave propagates back up the channel
5. Repeat

#### Streamer Zone

Ahead of the stepped leader, a **streamer zone** extends 10-100 m, consisting of:
- Cold plasma filaments (corona streamers)
- Temperature: 300-1000 K
- Provides preliminary ionization and conductivity enhancement
- Streamers propagate at ~10⁶ m/s

This zone is what "feels out" the available paths, leading to the tortuous, branching structure.

#### Attachment Process

As the leader approaches ground (within ~100 m):
- **Upward connecting leaders** initiate from ground objects
- Upward leader speed: 1-5 × 10⁵ m/s
- Connection occurs at the **striking distance** (30-200 m depending on charge)
- Connection creates a conducting path for the return stroke

**Striking distance formula (common approximation):**
```
d_s = 10 × I_p^0.65 (in meters, I_p in kA)
```
Where I_p is the peak return stroke current.

#### Return Stroke

The **return stroke** is the bright, high-current wave that propagates up the established leader channel.

**Measured values:**

| Parameter | Typical Value | Range |
|-----------|---------------|-------|
| Propagation speed | c/3 (1 × 10⁸ m/s) | (0.1-0.5)c |
| Peak current | 30 kA | 10-200 kA (first stroke) |
| Current rise time | 5 μs | 1-10 μs |
| Current decay (1/e time) | 50-100 μs | |
| Total duration | 100-200 μs | |
| Channel temperature | 30,000 K | 25,000-35,000 K |
| Peak channel radius | 1-3 cm | |
| Energy per meter | 10³-10⁴ J/m | |
| Peak power | 10¹² W (briefly) | |

The luminosity profile follows the current closely:
- Peak luminosity at base, propagates upward
- Upper portions remain bright for ~100 μs after passage
- Optical emission dominated by N II, O II, H alpha lines

#### Dart Leaders and Subsequent Strokes

Most flashes contain 3-5 strokes (range: 1-26). After the first stroke:

1. **J-process**: Current flows in cloud between strokes (40-100 ms)
2. **K-changes**: Recoil streamers in cloud
3. **Dart leader**: Continuous (not stepped) leader along existing channel
   - Speed: 1-2 × 10⁷ m/s (10× faster than stepped leader)
   - Duration: 1-2 ms
   - Current: 1-2 kA
4. **Subsequent return stroke**: Similar to first but usually lower current
   - Peak current: 10-15 kA (median)
   - Faster rise time: 0.5-1 μs

**Interstroke interval:** 40-100 ms (creates characteristic "flickering")

### 1.2 Branching Statistics

Branching is a fundamental characteristic arising from the probabilistic nature of the streamer zone and competition between possible paths.

#### Branch Geometry

**Measured branch angles (from optical and radar studies):**
- Mean angle from parent: 16-20°
- Standard deviation: 10-15°
- Range: 5-45° (95% of branches)
- Branches are rarely perpendicular (>60°) or nearly parallel (<5°)

**Branch angle distribution:** Approximately Gaussian with slight forward bias:
```
P(θ) ∝ exp(-(θ - θ₀)² / 2σ²) × cos(θ)
```
Where θ₀ ≈ 20° and σ ≈ 12°.

#### Branch Frequency and Attenuation

**Branch-to-trunk ratio:**
- Number of branches per unit length decreases with distance from cloud
- Upper channel: 1 branch per 100-200 m
- Lower channel: 0.5-1 branch per 100-200 m
- Total visible branches: 10-30 (typical), up to 100+ (exceptional)

**Branch attenuation with depth:**
- Primary branches (from main channel): Full intensity
- Secondary branches: 50-70% intensity
- Tertiary branches: 25-40% intensity
- Branches beyond depth 3 rarely visible

**Length distribution:**
- Most branches are short (10-100 m)
- Branch length follows approximately exponential distribution
- Mean branch length / total flash length ≈ 0.1-0.3

**Luminosity ratio:**
- Failed branches: 1-10% of main channel luminosity
- Successful branch (becomes new main channel): 100%
- Branch luminosity during return stroke: Branches illuminate briefly as current redistributes

### 1.3 Fractal Dimension

Lightning exhibits fractal (self-similar) structure across scales from meters to kilometers.

**Measured fractal dimensions:**

| Measurement Type | Fractal Dimension D | Source |
|-----------------|---------------------|--------|
| 2D projection (box-counting) | 1.51 ± 0.05 | Niemeyer et al. (1984) |
| 2D projection (correlation) | 1.34-1.73 | Various studies |
| 3D reconstruction | 1.5-1.7 | Radar studies |
| DBM simulation (η=1) | 1.75 | (DLA limit) |
| DBM simulation (η=2) | 1.5-1.6 | Matches observations |
| DBM simulation (η→∞) | 1.0 | (straight line) |

The fractal dimension D relates to how structure fills space:
- D = 1: A straight line
- D = 1.5: Moderately tortuous, some branching
- D = 2: Plane-filling (dense branching)

For lightning, D ≈ 1.5 means:
- Moderate tortuosity
- Sparse but significant branching
- Self-similar structure across ~3 orders of magnitude (1 m to 1 km)

### 1.4 Why Lightning is Tortuous

The stepped leader does not follow a straight path because:

1. **Atmospheric conductivity variations**: Humidity, temperature, and ion density vary spatially, creating regions of lower breakdown threshold.

2. **Stochastic streamer competition**: Multiple streamers ahead of the leader tip compete; the winning streamer determines the next step direction.

3. **Space charge effects**: Previous ionization modifies the local field, pushing subsequent steps in new directions.

4. **Pre-existing ionization**: Cosmic ray tracks, previous discharges, or other ionization sources create "paths of least resistance."

5. **Scale of variations**: Relevant fluctuations occur at scales of 1-100 m, matching the step length, creating tortuosity at all observable scales.

**Quantitatively:**
- Mean angle between successive steps: 10-20°
- Standard deviation: 15-30°
- Correlation length: 2-5 steps (steps are not independent but have short-range correlation)

---

## 2. Mathematical Models for Simulation

### 2.1 Dielectric Breakdown Model (DBM)

The **Dielectric Breakdown Model**, introduced by Niemeyer, Pietronero, and Wiesmann (1984), is the foundation of most lightning simulations.

#### Physical Basis

The model is based on the observation that electrical breakdown in dielectrics follows paths that maximize the local electric field. The stochastic element arises from microscopic fluctuations in material properties.

#### The NPW Growth Model

**Setup:**
- Define a lattice (2D or 3D grid)
- Initialize with one or more "seed" points representing the leader origin (conductor potential φ = 0)
- Set boundary conditions: ground plane at φ = 0, initial field E₀ in z-direction
- Leader sites are conductors (φ = 0 or floating potential)

**Growth algorithm:**

1. **Identify growth candidates**: All non-leader sites adjacent to existing leader sites

2. **Solve for potential field**: On the lattice, solve Laplace's equation:
   ```
   ∇²φ = 0
   ```
   With boundary conditions:
   - φ = 0 on leader sites (conducting channel)
   - φ = 0 on ground plane
   - φ → E₀z far from leader (uniform background field)

3. **Compute growth probability for each candidate site i:**
   ```
   P(i) = |E_i|^η / Σⱼ |E_j|^η
   ```
   Where:
   - E_i = |∇φ|_i = local electric field magnitude at site i
   - η = growth exponent (key parameter)
   - Sum is over all candidate sites j

4. **Select growth site**: Sample from probability distribution, add to leader

5. **Repeat**: Continue until leader reaches ground or specified length

#### The η Parameter

The exponent η controls the branching density and tortuosity:

| η Value | Behavior | Fractal Dimension D |
|---------|----------|---------------------|
| η = 0 | Random walk (Eden model) | D = 2 (plane-filling) |
| η = 1 | Diffusion-limited aggregation (DLA) | D ≈ 1.7 |
| η ≈ 2 | Realistic lightning | D ≈ 1.5 |
| η ≈ 3-4 | Sparse branching | D ≈ 1.2-1.3 |
| η → ∞ | Straight line (maximum field only) | D = 1 |

**For realistic lightning:** η = 2.0-2.5 is commonly used.

**Physical interpretation:** Higher η means the leader more strongly "prefers" the highest-field direction, suppressing exploration of suboptimal paths.

#### Solving Laplace's Equation

The computational bottleneck is solving ∇²φ = 0 at each growth step.

**Exact methods (for reference):**
- Relaxation (Jacobi, Gauss-Seidel): O(N² log ε⁻¹) per step
- Successive Over-Relaxation (SOR): O(N^1.5) per step
- Multigrid: O(N) per step
- FFT-based (periodic boundaries): O(N log N)

Where N = number of lattice points, ε = convergence tolerance.

**For real-time graphics, we need approximations (see Section 2.4).**

#### Handling Conductors

When a new site joins the leader, its potential becomes φ = 0 (or a floating conductor potential if modeling the channel's finite conductivity).

**Floating conductor approach (more physical):**
- Leader tip potential: High (50-100 MV)
- Potential decreases along channel toward cloud
- Requires solving for charge distribution, more complex

**Grounded conductor approach (simpler, common):**
- All leader sites at φ = 0
- Overestimates field near older channel sections
- Works well for visual appearance

### 2.2 Stochastic Stepped Leader Model

The Bazelyan-Raizer approach (1998) models individual step decisions more explicitly.

#### Algorithm

1. **At current leader tip**, compute the local electric field enhancement factor:
   ```
   E_tip = E₀ × k
   ```
   Where k is the enhancement factor (typically 10-100 depending on tip geometry).

2. **Generate candidate directions** in a cone around the field direction:
   - Cone half-angle: 30-60°
   - Sample N candidate directions (typically 10-50)

3. **For each candidate direction θᵢ:**
   - Compute the field component in that direction: E_i = E_tip · cos(θᵢ)
   - Add atmospheric noise: E'_i = E_i × (1 + ξ_i) where ξ ~ N(0, σ_atm)

4. **Compute step probability:**
   ```
   P(θᵢ) = max(0, E'_i - E_th)^η / Σⱼ max(0, E'_j - E_th)^η
   ```
   Where E_th is the threshold field for breakdown.

5. **Sample step direction** from probability distribution.

6. **Advance tip** by step length L_step in chosen direction.

7. **Branch decision:**
   - With probability P_branch = α × (E_tip / E_ref)^β, spawn a secondary leader
   - Secondary leaders follow the same process but with reduced charge

#### Key Parameters

- **Step length L_step**: 30-100 m (scale appropriately)
- **Atmospheric noise σ_atm**: 0.2-0.5 (20-50% fluctuations)
- **Threshold field E_th**: 500-1000 kV/m
- **Branch probability base α**: 0.01-0.05 per step
- **Field enhancement for branching β**: 1-2

### 2.3 Diffusion-Limited Aggregation (DLA) Connection

The DBM with η = 1 is mathematically equivalent to **Diffusion-Limited Aggregation**:

- Random walkers (representing diffusing electric field lines) start far from the structure
- They attach upon contact with the growing structure
- The probability of attachment at any site equals the harmonic measure (related to electric field)

**Why this matters:**
- DLA is well-studied; results apply to lightning
- Fractal dimension D ≈ 1.7 for DLA in 2D
- η > 1 reduces branching below the DLA limit
- η = 2 gives D ≈ 1.5, matching observations

**The connection:**
- Electric field lines "diffuse" from infinity to the conductor
- Field magnitude at a point = probability density of random walkers passing through
- Higher field = more likely growth = preferential attachment

### 2.4 Hybrid and Simplified Approaches for Real-Time

Full DBM is too slow for real-time rendering. Several approximations are viable:

#### Midpoint Displacement with Field Bias

The simplest approach, suitable for background bolts or low-detail views:

1. **Start with straight line** from cloud to ground
2. **Recursively subdivide:**
   - At each midpoint, displace perpendicular to segment
   - Displacement magnitude: d = h × 2^(-H×n) × ξ
   - Where h = scale, H = Hurst exponent (H ≈ 0.5 for D ≈ 1.5), n = recursion level, ξ ~ N(0,1)
3. **Add downward bias:** Displacements weighted toward ground
4. **Add branches** probabilistically at midpoints

**Advantages:** O(N) for N segments, trivially parallelizable
**Disadvantages:** Doesn't respond to geometry (buildings, terrain)

#### Stein & Sbert (2015) Real-Time Approach

A hybrid combining DBM principles with efficient approximations:

1. **Coarse DBM grid** (32³ or 64³) to establish main path
2. **Field approximation:** Instead of solving Laplace, use superposition:
   ```
   φ(r) = Σᵢ qᵢ / |r - rᵢ| + E₀ · r
   ```
   Where charges qᵢ are placed at leader segments (method of images).

3. **Hierarchical refinement:**
   - Solve coarse grid
   - Interpolate path
   - Add fine-scale noise using midpoint displacement

4. **GPU acceleration:** Field computation is embarrassingly parallel

#### Fast Multipole / Distance-Based Field Approximation

For moderate accuracy without solving Laplace:

1. **Leader channel as line charges:** Represent channel as sequence of point charges
2. **Image charges:** For ground plane, add mirror charges
3. **Approximate field at candidate point:**
   ```
   E(r) ≈ E₀ẑ + Σᵢ (qᵢ/ε₀) × (r - rᵢ)/|r - rᵢ|³
   ```

4. **Fast Multipole Method (FMM):** For many charges, use FMM to achieve O(N) instead of O(N²)

5. **Distance-based fallback:** For very fast approximation:
   ```
   E_z(r) ≈ E₀ × (1 + k/(d_channel + ε))
   ```
   Where d_channel = distance to nearest channel point.

---

## 3. Key Formulas in Mathematical Notation

### 3.1 Core DBM Growth Probability

The fundamental equation determining where the leader grows:

```
            |φᵢ|^η
P(i) = ─────────────
        Σⱼ |φⱼ|^η
```

**Variables:**
- **P(i)**: Probability of growth at candidate site i
- **φᵢ**: Electric potential (or field magnitude |∇φ|) at site i [V or V/m]
- **η**: Branching exponent (dimensionless)
- **Σⱼ**: Sum over all candidate growth sites

**Typical values:**
- η = 2.0 for realistic branching
- φᵢ computed from Laplace solution

**Visual effect of η:**
- η = 1: Dense branching, bushy appearance
- η = 2: Moderate branching, natural appearance
- η = 3+: Sparse/no branching, straighter paths

### 3.2 Laplace's Equation for Potential Field

The electric potential in the region around the leader satisfies:

```
∇²φ = ∂²φ/∂x² + ∂²φ/∂y² + ∂²φ/∂z² = 0
```

**Variables:**
- **φ(x,y,z)**: Electric potential [V]
- **∇²**: Laplacian operator

**Boundary conditions:**
- φ = 0 on leader channel (conductor)
- φ = 0 on ground plane (z = 0)
- φ → -E₀z as z → ∞ (uniform field)

**Typical values:**
- E₀ = 10-50 kV/m (background field)
- Ground at z = 0
- Cloud base at z = 5-10 km

**Discretized (for numerical solution on grid spacing h):**
```
φᵢ,ⱼ,ₖ = (1/6)(φᵢ₊₁,ⱼ,ₖ + φᵢ₋₁,ⱼ,ₖ + φᵢ,ⱼ₊₁,ₖ + φᵢ,ⱼ₋₁,ₖ + φᵢ,ⱼ,ₖ₊₁ + φᵢ,ⱼ,ₖ₋₁)
```

### 3.3 Electric Field from Potential

The electric field is the negative gradient of potential:

```
E = -∇φ = -(∂φ/∂x, ∂φ/∂y, ∂φ/∂z)
```

**Magnitude (used in growth probability):**
```
|E| = √((∂φ/∂x)² + (∂φ/∂y)² + (∂φ/∂z)²)
```

**Discrete approximation (central differences):**
```
Eₓ ≈ -(φᵢ₊₁,ⱼ,ₖ - φᵢ₋₁,ⱼ,ₖ)/(2h)
```

### 3.4 Step Direction Probability Distribution

For the stochastic stepped leader model, probability of stepping in direction θ (angle from vertical):

```
P(θ) ∝ exp(-(θ - θ₀)²/(2σ²)) × max(0, E(θ) - E_th)^η
```

**Variables:**
- **θ**: Candidate step direction (angle from vertical) [radians]
- **θ₀**: Mean deviation angle ≈ 0 (slight downward bias)
- **σ**: Direction spread ≈ 0.3-0.5 rad (17-29°)
- **E(θ)**: Field magnitude in direction θ [V/m]
- **E_th**: Threshold field for breakdown [V/m]
- **η**: Exponent controlling field-following strength

**Typical values:**
- σ = 0.35 rad (20°)
- E_th = 500-1000 kV/m
- η = 2

**Visual effect:**
- Larger σ: More tortuous path
- Higher E_th: Stronger tendency to follow field
- Higher η: Less random deviation

### 3.5 Branch Probability Function

Probability of initiating a branch at a given step:

```
P_branch = P₀ × (E_local/E_ref)^β × f(depth)
```

**Where the depth attenuation function:**
```
f(depth) = exp(-depth/λ_branch)
```

**Variables:**
- **P₀**: Base branch probability per step [dimensionless]
- **E_local**: Local electric field magnitude [V/m]
- **E_ref**: Reference field scale [V/m]
- **β**: Field enhancement exponent [dimensionless]
- **depth**: Number of branching levels from main channel
- **λ_branch**: Branch depth decay constant [dimensionless]

**Typical values:**
- P₀ = 0.02-0.05 (2-5% per step on main channel)
- E_ref = 1 MV/m
- β = 1.5
- λ_branch = 2.0 (branches decay by 1/e every 2 levels)

**Visual effect:**
- Higher P₀: More branches overall
- Higher β: Branches concentrate where field is strong
- Lower λ_branch: Branches die out faster with depth

### 3.6 Return Stroke Current Waveform

The current at the base of the channel during return stroke:

```
I(t) = I₀ × (t/τ₁)² × exp(-t/τ₂) / ((t/τ₁)² + 1)
```

**(Heidler function, commonly used)**

**Variables:**
- **I(t)**: Current at time t [A]
- **I₀**: Peak current scaling [A]
- **τ₁**: Rise time constant [s]
- **τ₂**: Decay time constant [s]

**Typical values:**
- I₀ = 30,000 A (30 kA peak for first stroke)
- τ₁ = 1-2 μs
- τ₂ = 50-100 μs

**For subsequent strokes:**
- I₀ = 10,000-15,000 A
- τ₁ = 0.5 μs (faster rise)
- τ₂ = 30-50 μs

### 3.7 Luminosity Along Channel

Luminosity as function of height and time during return stroke:

```
L(z, t) = L₀ × I(t - z/v_rs) × exp(-z/λ_L)
```

**Variables:**
- **L(z, t)**: Luminosity at height z, time t [W/m or relative]
- **L₀**: Luminosity scaling constant
- **I(·)**: Current waveform function
- **z**: Height above ground [m]
- **v_rs**: Return stroke propagation speed [m/s]
- **λ_L**: Luminosity decay length [m]

**Typical values:**
- v_rs = 1 × 10⁸ m/s (c/3)
- λ_L = 2000-5000 m (luminosity decays with height)

**Visual effect:**
- Return stroke appears as bright wave traveling upward at v_rs
- Upper channel remains luminous but dimmer
- Total flash duration ~100-200 μs

### 3.8 Channel Radius vs Current (Braginskii Model)

The radius of the hot, luminous channel core:

```
r(t) = r₀ × (I(t)/I₀)^α × (t/t₀)^β
```

**Simplified Braginskii (1958) scaling:**
```
r ∝ (I²t/ρ)^(1/4)
```

**Variables:**
- **r(t)**: Channel radius at time t [m]
- **I(t)**: Current at time t [A]
- **ρ**: Air density [kg/m³]
- **r₀, I₀, t₀**: Reference scales
- **α ≈ 0.5**: Current exponent
- **β ≈ 0.25**: Time exponent

**Typical values:**
- Peak radius: 1-3 cm (during return stroke peak)
- Initial radius: 1-5 mm (stepped leader channel)

**For rendering:**
- Core (hot plasma): 1-3 cm → renders as brightest region
- Corona sheath: 10-50 cm → renders as glow
- Visible channel width: 1-10 m (bloom/glow effects)

### 3.9 Fractal Dimension Constraint

To achieve fractal dimension D ≈ 1.5, the simulation should satisfy:

```
D = d - η/(η + 1)  (approximate, for DBM in d dimensions)
```

**Or empirically calibrated:**
```
D ≈ 2 - 0.27 × η  (for 2D DBM, η in range 1-4)
```

**For D = 1.5 in 2D:**
```
η ≈ 1.85-2.0
```

**For 3D, similar relationship but shifted.**

### 3.10 Step Length Distribution

Step lengths follow approximately lognormal distribution:

```
P(L) = (1/(L × σ_L × √(2π))) × exp(-(ln L - μ_L)²/(2σ_L²))
```

**Variables:**
- **L**: Step length [m]
- **μ_L**: Log-mean of step length
- **σ_L**: Log-standard deviation

**Typical values:**
- Median step length: 50 m
- μ_L = ln(50) ≈ 3.9
- σ_L ≈ 0.5-0.7

**Alternative (exponential, simpler):**
```
P(L) = (1/L₀) × exp(-L/L₀)
```
With L₀ = 50 m mean step length.

---

## 4. User-Tunable Parameters

### 4.1 Branching Exponent (η)

**Physical meaning:** Controls how strongly the leader follows the electric field gradient vs exploring alternative paths.

**Visual effect:**
- η = 1.0: Dense, bushy structure with many branches (DLA-like)
- η = 2.0: Natural lightning appearance, moderate branching
- η = 3.0: Sparse branching, more direct path
- η = 4.0+: Nearly straight bolt with rare forks

**Reasonable range:** 1.0 - 4.0

**Default for realistic lightning:** η = 2.0

**Implementation note:** This is the single most important parameter for visual character.

### 4.2 Step Length

**Physical meaning:** Average distance the leader advances in each discrete step.

**Visual effect:**
- Shorter steps: Smoother, more continuous appearance; finer tortuosity
- Longer steps: More angular, "jagged" appearance; coarser structure

**Reasonable range:**
- Physical scale: 10-200 m
- Normalized (0-1 cloud-ground distance): 0.005-0.05

**Default:** 50 m (or ~0.01 normalized)

**Implementation note:** Should scale with grid resolution. For a 100-cell grid, use 1 cell per step.

### 4.3 Branch Angle Distribution

**Physical meaning:** Angular deviation of branches from parent channel direction.

**Visual effect:**
- Narrow distribution (σ ~ 10°): Branches nearly parallel to main channel
- Wide distribution (σ ~ 30°): Branches spread widely, more "explosive" appearance
- Mean angle affects whether branches angle upward, outward, or downward

**Parameters:**
- Mean angle from parent: θ₀ = 15-25°
- Standard deviation: σ = 10-20°

**Reasonable range:**
- θ₀: 0° - 45°
- σ: 5° - 30°

**Default:** θ₀ = 20°, σ = 12°

### 4.4 Maximum Branch Depth

**Physical meaning:** How many generations of branches are allowed (main channel = depth 0).

**Visual effect:**
- Depth 1: Only primary branches from main channel
- Depth 2: Branches can branch once more
- Depth 3+: Complex, tree-like structure

**Reasonable range:** 1 - 5

**Default:** 3 (rarely see branches beyond this in nature)

**Performance note:** Computation scales exponentially with depth if not capped.

### 4.5 Branch Probability Decay

**Physical meaning:** How quickly branch probability decreases with depth.

**Formula:** P(depth) = P₀ × exp(-depth/λ)

**Visual effect:**
- λ = 1: Rapid decay, few secondary branches
- λ = 2: Moderate decay, natural appearance
- λ = 3+: Slow decay, many deep branches (unrealistic)

**Reasonable range:** λ = 0.5 - 3.0

**Default:** λ = 2.0

### 4.6 Field Noise Scale (Atmospheric Turbulence)

**Physical meaning:** Magnitude of random fluctuations in breakdown threshold representing atmospheric variations.

**Visual effect:**
- Low noise (σ = 0.1): Smooth, predictable paths following field lines
- Medium noise (σ = 0.3): Natural variation, realistic tortuosity
- High noise (σ = 0.5+): Erratic, wandering paths

**Reasonable range:** 0.05 - 0.5 (as fraction of mean field)

**Default:** 0.25

**Implementation note:** Applied as multiplicative noise to field magnitude before probability calculation.

### 4.7 Return Stroke Speed

**Physical meaning:** Propagation speed of the luminous return stroke wave.

**Visual effect:**
- Slower (0.1c): Visible upward propagation, dramatic effect
- Faster (0.5c): Nearly instantaneous illumination
- For real-time, may need to slow down for visibility

**Physical value:** ~c/3 (1 × 10⁸ m/s)

**Reasonable range for visualization:** 0.1c - 0.5c, or slower for dramatic effect

**Default:** 0.3c (matches observations)

**Implementation note:** At physical speed, a 5 km channel illuminates in 50 μs—imperceptible. For animation, may use 10-100 ms.

### 4.8 Luminosity Decay Rate

**Physical meaning:** How quickly the channel dims after return stroke passage.

**Visual effect:**
- Fast decay (τ = 50 μs): Brief flash, minimal afterglow
- Slow decay (τ = 200 μs): Lingering glow, more visible structure

**Physical value:** τ = 50-100 μs (return stroke), longer for continuing current

**Reasonable range for visualization:** 50-500 ms (scaled up for visibility)

**Default:** 100 ms (visual time scale)

### 4.9 Number of Subsequent Strokes

**Physical meaning:** Total number of return strokes in the flash (creates "flickering").

**Visual effect:**
- 1 stroke: Single flash
- 3-5 strokes: Characteristic flicker
- 10+ strokes: Prolonged, dramatic display

**Physical value:** Mean = 3-4, range = 1-26

**Reasonable range:** 1 - 10

**Default:** 4

### 4.10 Interstroke Interval

**Physical meaning:** Time between successive strokes.

**Visual effect:**
- Short (30 ms): Rapid flicker
- Long (100 ms): Distinct separate flashes

**Physical value:** 40-100 ms

**Reasonable range:** 30-150 ms

**Default:** 60 ms

### 4.11 Channel Glow Radius

**Physical meaning:** Visual extent of the luminous channel and corona.

**Physical value:** Core ~2 cm, visible glow ~1-10 m (due to atmospheric scattering)

**Visual effect:**
- Thin: Sharp, electrical appearance
- Thick: Dramatic, glowing appearance

**Reasonable range:** Depends on rendering scale

**Default:** Adjust to taste based on scene scale

### 4.12 Color Temperature

**Physical meaning:** Spectral distribution of emitted light.

**Physical value:** ~30,000 K (hot plasma, appears white-blue)

**Visual effect:**
- ~8000 K: Slightly warm white
- ~15000 K: Pure white
- ~30000 K: Blue-white (physical)

**Reasonable range:** 8000 K - 50000 K (or direct color specification)

**Default:** 25000 K (slightly blue-white)

---

## 5. Simplification Strategy for Detail Levels

### 5.1 Two-Tier Detail System

The application requires two rendering contexts with different computational budgets:

| Context | Resolution | Computation Budget | Distance |
|---------|------------|-------------------|----------|
| Globe view | Low | <1 ms | 100+ km |
| Showcase view | High | <16 ms (60 fps) | <1 km |

### 5.2 Globe View (Coarse Detail)

**Grid resolution:** 16³ or 32³ (maximum)

**Simplifications:**
1. **No field solve:** Use distance-based field approximation
   ```
   E(r) ≈ E₀ × (1 + k/d_ground)
   ```

2. **Midpoint displacement only:**
   - Start with straight cloud-to-ground line
   - 4-6 levels of recursive subdivision
   - Add branches probabilistically (not field-weighted)

3. **Fixed branching:**
   - Pre-determined branch points
   - Max depth = 2
   - No field-dependent branch probability

4. **Coarse geometry:**
   - 20-50 line segments total
   - Billboard sprites instead of volumetric glow
   - No corona sheath

5. **Simplified animation:**
   - Instant or single-pass illumination
   - No return stroke wave
   - Binary on/off with fade

**Computational cost:** <0.5 ms on GPU

### 5.3 Showcase View (Fine Detail)

**Grid resolution:** 64³ to 128³

**Full features:**
1. **Approximate field solve:**
   - Use line charge superposition
   - 2-3 relaxation iterations per step (not full convergence)
   - GPU-parallel field evaluation

2. **Full DBM with η ≈ 2:**
   - Proper growth probability calculation
   - 100-200 growth steps

3. **Dynamic branching:**
   - Field-weighted branch probability
   - Max depth = 3-4
   - Branch attenuation with depth

4. **Fine geometry:**
   - 500-2000 line segments
   - Variable thickness along channel
   - Corona glow shader

5. **Full animation:**
   - Stepped leader with visible steps (optional)
   - Return stroke wave
   - Multiple strokes with interstroke intervals
   - Luminosity decay per segment

**Computational cost:** 5-15 ms total

### 5.4 Pre-computation vs Real-Time Tradeoffs

#### What to Pre-compute

1. **Bolt path geometry:**
   - Generate paths offline or in loading phase
   - Store as vertex buffers
   - Multiple pre-computed variants for randomization

2. **Branch structure:**
   - Tree topology (parent-child relationships)
   - Branch depths and attenuation factors

3. **Noise textures:**
   - Perlin/simplex noise for displacement
   - Pre-baked, tile, and sample at runtime

#### What to Compute in Real-Time

1. **Animation state:**
   - Current stroke phase (leader, return, decay)
   - Per-segment luminosity

2. **Shader effects:**
   - Glow/bloom
   - Atmospheric scattering
   - Flickering

3. **LOD transitions:**
   - Smooth interpolation between detail levels

### 5.5 Level-of-Detail (LOD) Scaling

**Distance-based LOD:**

```
LOD_level = clamp(floor(log2(distance / d_ref)), 0, LOD_max)
```

| LOD Level | Distance | Segments | Branches | Glow |
|-----------|----------|----------|----------|------|
| 0 (highest) | <1 km | 500+ | Full | Volumetric |
| 1 | 1-5 km | 100-200 | Depth 2 | Billboard |
| 2 | 5-20 km | 50-100 | Depth 1 | Simple |
| 3 (lowest) | >20 km | 20-50 | None | Point |

**Screen-space LOD:**

```
LOD_level = clamp(floor(-log2(screen_size / s_ref)), 0, LOD_max)
```

Where screen_size = projected angular size of bolt.

### 5.6 GPU-Specific Optimizations

1. **Instanced rendering:**
   - Single bolt mesh, instanced for multiple bolts
   - Per-instance: position, rotation, scale, phase

2. **Geometry shaders (if available):**
   - Input: line segments
   - Output: camera-facing quads with glow

3. **Compute shaders for field:**
   - Parallel field evaluation at all candidate points
   - Reduction for probability normalization

4. **Texture-based lookup:**
   - Pre-compute field for simple geometries
   - Sample from 3D texture

---

## 6. References

### Primary Literature

1. **Dwyer, J. R., & Uman, M. A. (2014).** The physics of lightning. *Physics Reports*, 534(4), 147-241.
   - Comprehensive review of lightning physics
   - Source for measured values of leader and return stroke parameters

2. **Rakov, V. A., & Uman, M. A. (2003).** *Lightning: Physics and Effects.* Cambridge University Press.
   - Definitive textbook on lightning
   - Detailed measurements and statistics

3. **Bazelyan, E. M., & Raizer, Y. P. (1998).** *Spark Discharge.* CRC Press.
   - Stochastic model for leader progression
   - Physical basis for stepped leader mechanics

4. **Niemeyer, L., Pietronero, L., & Wiesmann, H. J. (1984).** Fractal dimension of dielectric breakdown. *Physical Review Letters*, 52(12), 1033.
   - Original DBM paper
   - Growth probability formula

### Simulation and Rendering

5. **Mansell, E. R., & Ziegler, C. L. (2000).** Computer modeling of lightning channel evolution. *ISCCP Regional Satellite Studies*, Conference Paper.
   - 3D lightning modeling in cloud context

6. **Gulyas, L. (2009).** 3D simulation of the lightning path using a mixed physical-probabilistic model. *Journal of Electrostatics*, 67(2-3), 509-513.
   - Hybrid simulation approach

7. **Stein, J., & Sbert, M. (2015).** Real-Time Procedural Lightning using Hybrid Techniques. *Eurographics 2015 - Short Papers*.
   - GPU-friendly approximations
   - Practical real-time techniques

### Fractal Analysis

8. **Femia, N., Niemeyer, L., & Tucci, V. (1993).** Fractal characteristics of electrical discharges: experiments and simulation. *Journal of Physics D: Applied Physics*, 26(4), 619.
   - Fractal dimension measurements

### Channel Physics

9. **Braginskii, S. I. (1958).** Theory of the development of a spark channel. *Soviet Physics JETP*, 34(7), 1068-1074.
   - Channel radius model

10. **Heidler, F. (1985).** Traveling current source model for LEMP calculation. *Proceedings of the 6th Symposium on Electromagnetic Compatibility*, 29-33.
    - Return stroke current waveform

---

## Appendix A: Quick Reference Tables

### A.1 Physical Constants and Scales

| Quantity | Value | Unit |
|----------|-------|------|
| Breakdown field (sea level) | 3 | MV/m |
| Breakdown field (cloud altitude) | 1-2 | MV/m |
| Speed of light | 3 × 10⁸ | m/s |
| Typical cloud-ground distance | 5-10 | km |
| Air density (sea level) | 1.2 | kg/m³ |
| Air density (5 km altitude) | 0.6 | kg/m³ |

### A.2 Stepped Leader Parameters

| Parameter | Typical | Range | Unit |
|-----------|---------|-------|------|
| Step length | 50 | 3-200 | m |
| Pause time | 50 | 20-100 | μs |
| Speed | 2 × 10⁵ | 1-20 × 10⁵ | m/s |
| Current (mean) | 100-200 | 50-500 | A |
| Current (step peak) | 1-5 | - | kA |
| Channel radius | 1-10 | - | cm |

### A.3 Return Stroke Parameters

| Parameter | Typical | Range | Unit |
|-----------|---------|-------|------|
| Speed | 1 × 10⁸ | 0.3-1.5 × 10⁸ | m/s |
| Peak current (1st) | 30 | 10-200 | kA |
| Peak current (subsequent) | 12 | 5-50 | kA |
| Rise time (1st) | 5 | 1-10 | μs |
| Rise time (subsequent) | 0.5 | 0.2-2 | μs |
| Temperature | 30,000 | 25,000-35,000 | K |

### A.4 DBM Parameter Effects

| η Value | Fractal Dimension | Visual Description |
|---------|-------------------|-------------------|
| 1.0 | 1.7 | Dense branching (DLA) |
| 1.5 | 1.6 | Heavy branching |
| 2.0 | 1.5 | Natural lightning |
| 2.5 | 1.4 | Moderate branching |
| 3.0 | 1.3 | Sparse branching |
| 4.0 | 1.1 | Nearly straight |

---

*Document prepared as technical reference for lightning simulation implementation.*
*Based on peer-reviewed literature and established simulation methodologies.*
