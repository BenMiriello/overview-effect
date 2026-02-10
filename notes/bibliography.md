# Lightning Visualization Bibliography

Working reference list for lightning simulation, rendering, and data sources.

---

## Simulation & Rendering (Academic)

### UNC Lightning Paper - Kim & Lin
- **URL**: http://gamma.cs.unc.edu/LIGHTNING/
- **PDF**: http://gamma.cs.unc.edu/LIGHTNING/lightning.pdf
- **Contents**: Dielectric breakdown model formulation, dancing arc animation via Helmholtz equation, convolution-based rendering competitive with Monte Carlo ray tracing
- **Best for**: Understanding DBM math, efficient rendering techniques, user-controllable parameters
- **Worth reading if**: You need the mathematical foundation or want to implement efficient glow/bloom effects

### RPI Student Project - Visual Simulation of Lightning
- **URL**: https://www.cs.rpi.edu/~cutler/classes/advancedgraphics/S17/final_projects/sam_ian.pdf
- **Contents**: Stepped leader simulation, branching probability, particle systems for rendering
- **Best for**: Implementation walkthrough, branching logic
- **Worth reading if**: You want a practical student-level implementation guide (PDF may require extraction)

### Reed & Wyvill - Visual Simulation of Lightning (1994)
- **URL**: https://dl.acm.org/doi/pdf/10.1145/192161.192256
- **Contents**: Early foundational work on lightning visualization
- **Best for**: Historical context, basic algorithms
- **Worth reading if**: You want to understand the evolution of techniques

### Warwick DBM/DLA Reference
- **URL**: https://warwick.ac.uk/fac/sci/physics/research/condensedmatt/imr_cdt/students/matthew_dale/dla/
- **Contents**: Dielectric breakdown model explanation, DLA comparison
- **Best for**: Understanding the physics model conceptually
- **Worth reading if**: You need intuition about why DBM creates realistic branching

---

## Simulation & Rendering (Practical/Code)

### Lichtenberg (C++/Python)
- **URL**: https://github.com/chromia/lichtenberg
- **Contents**: Multiple DBM implementations (Fast DBM, DLA, value noise), C++ core with Python bindings
- **Best for**: Reference implementation, comparing algorithm variants
- **Worth reading if**: You want working code to study or port

### SideFX Forum - Laplace Growth & DBM
- **URL**: https://www.sidefx.com/forum/topic/52813/
- **Contents**: Houdini implementation, "aiming location" for direction control, obstacles/rejection for path constraints, practical artist-oriented discussion
- **Best for**: Practical tips on controlling growth direction, preventing unwanted clustering
- **Worth reading if**: You're debugging direction/seeking behavior

### Three.js Lightning Strike Demo
- **URL**: https://yomboprime.github.io/lightning_strike_demo/webgl_lightningstrike.html
- **Contents**: Live WebGL demo of lightning effect
- **Best for**: Visual reference, seeing what's achievable in browser
- **Worth reading if**: You want to compare your results to a working example

### GameDev StackExchange - Lightning Bolt Effect
- **URL**: https://gamedev.stackexchange.com/questions/71397/how-can-i-generate-a-lightning-bolt-effect
- **Contents**: Community discussion of generation algorithms, midpoint displacement, etc.
- **Best for**: Quick overview of common techniques
- **Worth reading if**: You want alternative approaches to DBM

### ThaboModise Lightning Effects Tutorial
- **URL**: https://github.com/ThaboModise/Live_TutorialScript/blob/master/IntermediateTutorialSeries/Lightning_effects.html
- **Contents**: HTML/JS implementation
- **Best for**: Simple reference implementation
- **Worth reading if**: You want bare-bones working code

---

## Lightning Physics (Scientific)

### NOAA JetStream - Lightning Process
- **URL**: https://www.noaa.gov/jetstream/lightning/how-lightning-is-created/jetstream-max-lightning-process-keeping-in-step
- **Contents**: Stepped leader mechanics, return stroke physics, educational diagrams
- **Best for**: Understanding real lightning behavior
- **Worth reading if**: You need physics grounding for realistic animation timing

### NSSL Lightning Types
- **URL**: https://www.nssl.noaa.gov/education/svrwx101/lightning/types/
- **Contents**: Cloud-to-ground, intracloud, types of lightning
- **Best for**: Understanding different lightning forms
- **Worth reading if**: You want to simulate different lightning types

### Arizona Atmospheric Sciences - Stepped Leader & Return Stroke
- **URL**: http://www.atmo.arizona.edu/students/courselinks/spring08/atmo336s1/courses/fall19/atmo170a1s1/lecture_notes/nov26.html
- **Contents**: Academic lecture notes on leader/stroke physics
- **Best for**: Detailed timing and brightness characteristics
- **Worth reading if**: You need precise animation timing values

### Rolling Shutter Lightning Leader Timing
- **URL**: https://ermsta.com/posts/20200707
- **Contents**: Using camera rolling shutter to measure leader speed, brightness observations
- **Best for**: Real-world timing data (4-20ms leaders), brightness comparison (leader dim vs stroke bright)
- **Worth reading if**: You need empirical data for animation timing

### Lightning Photography - Leaders vs Return Strokes
- **URL**: https://www.dblanchard.net/blog/2010/07/lightning-images-return-strokes-and-stepped-leaders/
- **Contents**: Long-exposure photography showing branching in leaders vs single channel in return strokes
- **Best for**: Visual reference of what leaders actually look like, branching structure
- **Worth reading if**: You want to verify your branching looks realistic

---

## Lightning Data Sources

### Lightning Maps (Live)
- **URL**: https://www.lightningmaps.org/
- **Contents**: Real-time global lightning strike data
- **Best for**: Live data integration, geographic distribution
- **Worth reading if**: You need real strike locations for globe visualization

### Windy.com Thunderstorm Layer
- **URL**: https://www.windy.com/-Thunderstorms-thunder?thunder,14.495,-64.072,3
- **Contents**: Thunderstorm activity overlay, CAPE data
- **Best for**: Potential data source for storm activity visualization
- **Worth reading if**: You want to show storm probability, not just strikes

### Open-Meteo API
- **URL**: https://open-meteo.com/en/docs#hourly_parameter_definition
- **Contents**: Weather API with coordinate-based data
- **Best for**: Supplementary weather data by location
- **Worth reading if**: You need weather context for strikes

### Meteoblue City Climate
- **URL**: https://www.meteoblue.com/products/cityclimate
- **Contents**: City-level climate data
- **Best for**: Historical/statistical lightning data by city
- **Worth reading if**: You want aggregated statistics rather than live data

---

## Scientific Papers (Lightning Prediction)

### ScienceDirect - Lightning Prediction Papers
- https://www.sciencedirect.com/science/article/abs/pii/S0169809513000057
- https://www.sciencedirect.com/science/article/abs/pii/S037015731300375X?via%3Dihub
- **Best for**: Understanding correlation between atmospheric conditions and lightning
- **Worth reading if**: You want to predict/simulate where lightning might occur

### AGU - CAPE and Lightning Relationship
- **URL**: https://agupubs.onlinelibrary.wiley.com/doi/full/10.1029/2000JD000244
- **Contents**: Relationship between CAPE (convective energy) and lightning frequency
- **Best for**: Scientific basis for storm-to-lightning correlation
- **Worth reading if**: You want physics-based probability modeling

---

## Three.js / WebGL

### Three.js InstancedMesh
- **URL**: https://threejs.org/docs/#api/en/objects/InstancedMesh
- **Best for**: Efficient rendering of many similar objects
- **Worth reading if**: You need to optimize segment rendering

### Three.js Unreal Bloom
- **URL**: https://threejs.org/examples/#webgl_postprocessing_unreal_bloom
- **Best for**: Glow/bloom post-processing effect
- **Worth reading if**: You want dramatic flash effects

### Three.js Optimization Guide
- **URL**: https://threejs.org/manual/#en/optimize-lots-of-objects
- **Best for**: Performance optimization strategies
- **Worth reading if**: Globe view is lagging

### Globe.gl / React-Globe.gl
- **URL**: https://github.com/vasturiano/globe.gl
- **URL**: https://github.com/vasturiano/react-globe.gl
- **Best for**: Globe visualization foundation
- **Worth reading if**: You need to understand the underlying globe renderer

---

## OSP Physics Simulations

### Dielectric Breakdown Model (Interactive)
- **URL**: https://www.compadre.org/osp/items/detail.cfm?ID=11998
- **Docs**: https://www.compadre.org/OSP/document/ServeFile.cfm?ID=11998&DocID=2814
- **Contents**: Interactive Java simulation of DBM, educational tool
- **Best for**: Playing with parameters interactively to build intuition
- **Worth reading if**: You want to experiment before coding
