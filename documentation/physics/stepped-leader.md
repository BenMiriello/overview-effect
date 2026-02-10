# Stepped Leader Physics

The stepped leader is the initial, downward-propagating phase of cloud-to-ground lightning. It establishes the ionized channel through which the return stroke will travel.

---

## Overview

Lightning doesn't travel in a continuous motion. The stepped leader advances in discrete jumps called "steps," with brief pauses between each step. This stepping behavior is fundamental to lightning's characteristic tortuous, branching structure.

---

## Measured Physical Values

From Rakov & Uman (2003), Dwyer & Uman (2014):

| Parameter | Typical Value | Range |
|-----------|---------------|-------|
| Step length | 50 m | 3-200 m |
| Pause time between steps | 50 microseconds | 20-100 microseconds |
| Step duration | 1 microsecond | 0.5-2 microseconds |
| Average propagation speed | 2 x 10^5 m/s | 1-20 x 10^5 m/s |
| Total duration (cloud to ground) | 20-30 ms | 10-100 ms |
| Leader current (average) | 100-200 A | 50-500 A |
| Leader current (step tip) | 1-5 kA | - |
| Channel potential | 50-100 MV | - |
| Leader charge deposited | 3-5 C | 1-20 C |
| Channel radius | 1-10 cm | - |
| Temperature | 10,000-30,000 K | (estimated) |

---

## The Stepping Mechanism

Each step involves a complex sequence:

### 1. Field Intensification
Electric field concentrates at the leader tip, reaching breakdown threshold.

### 2. Space Stem Formation
A luminous region called a "space stem" appears 10-100 m ahead of the tip. This is NOT directly connected to the leader.

### 3. Rapid Connection
The leader tip connects to the space stem in approximately 1 microsecond. This creates the visible "step."

### 4. Ionization Wave
An ionization wave propagates back up the newly formed channel segment, preparing it for current flow.

### 5. Pause
The process pauses for 20-100 microseconds while conditions rebuild for the next step.

### 6. Repeat
Steps continue until ground connection or the leader terminates.

---

## Why Stepped Leaders are Tortuous

The leader doesn't follow a straight path due to several factors:

### Atmospheric Conductivity Variations
Humidity, temperature, and ion density vary spatially. Regions with lower breakdown thresholds become preferred paths.

### Stochastic Streamer Competition
Multiple streamers ahead of the leader tip compete. The "winning" streamer determines the direction of the next step. This is fundamentally probabilistic.

### Space Charge Effects
Previous ionization modifies the local electric field, pushing subsequent steps in new directions.

### Pre-existing Ionization
Cosmic ray tracks, previous discharges, and other ionization sources create "paths of least resistance."

### Scale of Variations
Relevant atmospheric fluctuations occur at scales of 1-100 m, matching the step length. This creates tortuosity observable at all scales.

**Quantitatively:**
- Mean angle between successive steps: 10-20 degrees
- Standard deviation: 15-30 degrees
- Correlation length: 2-5 steps (steps are not independent but have short-range correlation)

---

## Leader Sensing Range

A critical insight for simulation:

> The leader tip only "senses" charges within approximately 50 meters.

This limited sensing range is why the path is stochastic. The leader cannot "see" ground from cloud altitude; it explores based on local conditions only. This creates the characteristic branching and wandering behavior.

---

## Visual Characteristics

The stepped leader is **relatively dim** compared to the return stroke:
- Luminosity: 1-10% of return stroke
- Color: Blue-white (N II, O II emission lines)
- Appearance: Extensive branching visible
- Duration: Long enough for branches to be photographed (10-20 ms)

In long-exposure photographs, the stepped leader appears as a complex branching tree, while the return stroke illuminates only the single winning path.

---

## Initiation: Preliminary Breakdown

Before the stepped leader emerges from the cloud base, a "preliminary breakdown" process occurs inside the cloud:
- Duration: 1-10 ms
- Location: Near the boundary of the main negative charge region
- Initial current pulses: 1-10 kA
- Altitude of initiation: 4-8 km (typical for temperate latitudes)

This preliminary breakdown creates the conditions for the stepped leader to begin propagating downward.

---

## Simulation Implications

For our simulation:

1. **Step-based progression**: Geometry is built in discrete segments, each representing one step.

2. **Stochastic direction selection**: Each step samples from a probability distribution based on local field conditions.

3. **Limited lookahead**: The algorithm only considers local information (within ~stepLength radius), not the global field.

4. **Parallel exploration**: Multiple potential paths can exist simultaneously, with one eventually "winning."

5. **Appropriate timing**: Leader phase animation should be ~800ms (scaled from ~20ms real time) to make the stepping visible.

---

## References

- Rakov, V. A., & Uman, M. A. (2003). *Lightning: Physics and Effects.* Cambridge University Press.
- Dwyer, J. R., & Uman, M. A. (2014). The physics of lightning. *Physics Reports*, 534(4), 147-241.
- Bazelyan, E. M., & Raizer, Y. P. (1998). *Spark Discharge.* CRC Press.
