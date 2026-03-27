# Lightning Visualization - Technical Documentation

This documentation captures the physics, mathematics, simulation algorithms, and design decisions behind the lightning visualization system. It serves as both a reference for development and foundation for user-facing educational content.

---

## Document Structure

### `/physics/` - Real Lightning Phenomena
The actual physics of lightning as understood from atmospheric science research.

- **`stepped-leader.md`** - The stepped leader phase: how lightning initiates, propagates, and branches
- **`return-stroke.md`** - The visible flash: return stroke physics and luminosity
- **`charge-distribution.md`** - Atmospheric charge: cloud structure, induced ground charge, electrostatic induction
- **`branching-mechanics.md`** - Space leaders, streamer zones, and why lightning branches
- **`cloud-charge-dynamics.md`** - Cumulonimbus structure, tripole model, charge separation mechanisms

### `/simulation/` - Mathematical Models
How we translate physics into computable models.

- **`dielectric-breakdown-model.md`** - The core DBM algorithm: growth probability, the eta parameter, field computation
- **`voronoi-field-system.md`** - Smooth scalar fields for atmospheric properties
- **`multi-leader-competition.md`** - Leader spawning, competition-based death, winner determination
- **`atmospheric-model.md`** - Ceiling charge, ground charge, and their coupling
- **`wind-model.md`** - Atmospheric wind profiles, shear, turbulence, and implementation
- **`rain-model.md`** - Raindrop physics, wind interaction, particle rendering approach

### `/rendering/` - Visualization Techniques
How simulation output becomes visible graphics.

- **`animation-phases.md`** - Leader stepping, return stroke wave, subsequent strokes, flickering
- **`line-rendering.md`** - WebGL line width, LineSegments2, depth-based styling
- **`charge-visualization.md`** - Shader-based field visualization for charge planes
- **`charge-field-visualization.md`** - Single-plane shader rendering, noise-warped boundaries, metaball merging

### `/design-decisions/` - Why We Did What We Did
Key architectural and algorithmic choices with rationale.

- **`pre-computation-vs-realtime.md`** - Why simulation runs to completion before animation
- **`scale-normalization.md`** - Converting real-world meters to normalized simulation units
- **`parameter-tuning.md`** - Hard-won knowledge about what parameter values work
- **`emergent-vs-configured.md`** - Which behaviors emerge from physics vs explicit configuration

### `/architecture/` - System Overview
How the pieces fit together.

- **`timeline-worker-system.md`** - Web Worker timeline architecture, adaptive pacing, buffer management, and why strikes must be computed sequentially
- **`module-structure.md`** - File organization and module boundaries (TODO)
- **`data-flow.md`** - From simulation input to rendered frame (TODO)

---

## Quick Reference

### Core Equations

**DBM Growth Probability:**
```
P(i) = |phi_i|^eta / sum_j(|phi_j|^eta)
```

**Voronoi Cell Falloff (sinusoidal blending):**
```
value(point) = intensity * (cos(distance/radius * pi) + 1) / 2
```

**Distance-Based Field Approximation:**
```
E(r) = E_background + sum_i(k / |r - r_i|) + k_ground / (y - y_ground) + noise
```

### Key Scale Factors

| Real World | Simulation Units |
|------------|------------------|
| 50 meters (step length) | 0.008 units |
| 500 meters (charge pocket radius min) | 0.08 units |
| 1.5 km (charge pocket radius max) | 0.24 units |
| 6.25 km (full cloud-to-ground) | 1.0 unit |

### Parameter Ranges

| Parameter | Globe | Showcase | Effect |
|-----------|-------|----------|--------|
| eta | 2.0 | 2.0 | Higher = less branching, straighter paths |
| stepLength | 0.02 | 0.008 | Coarser vs finer geometry |
| maxSteps | 80 | 200 | Segment count limit |
| coneHalfAngle | PI/6 | PI/6 | Direction deviation range |

---

## Sources

Primary literature:
- Dwyer & Uman (2014) - "The physics of lightning" - Physics Reports
- Rakov & Uman (2003) - "Lightning: Physics and Effects" - Cambridge University Press
- Niemeyer, Pietronero, Wiesmann (1984) - Original DBM paper - Physical Review Letters

See `notes/bibliography.md` for complete reference list.

---

*Documentation created: 2026-02-10*
*Last updated: 2026-02-18*
