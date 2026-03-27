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

---

## Charge Field & Ionization Physics (Added 2026-02-18)

### Pawar & Kamra (2002) - Charge Recovery After Lightning
- **URL**: https://agupubs.onlinelibrary.wiley.com/doi/10.1029/2002GL015675
- **Contents**: Recovery curves of surface electric field after lightning discharges, charge pocket to negative charge center dynamics
- **Best for**: Post-strike charge recovery timescales (~5 seconds time constant)
- **Worth reading if**: You want accurate charge field animation after strikes

### Maggio et al. (2009) - Charge and Energy in Lightning
- **URL**: https://agupubs.onlinelibrary.wiley.com/doi/full/10.1029/2008JD011506
- **Contents**: Estimations of charge transferred (~5 C) and energy released by lightning flashes
- **Best for**: Quantitative charge values for leader channels
- **Worth reading if**: You want realistic charge deposition amounts

### Cruz et al. (2025) - Leader Speed and Peak Current Correlation
- **URL**: https://agupubs.onlinelibrary.wiley.com/doi/full/10.1029/2024GL111594
- **Contents**: Correlation between stepped leader speed and peak current of return stroke
- **Best for**: Linking leader propagation speed to flash intensity
- **Worth reading if**: You want physics-based animation speed/brightness relationships

### Saunders (2006) - Cloud Charge Separation
- **URL**: https://rmets.onlinelibrary.wiley.com/doi/abs/10.1256/qj.05.218
- **Contents**: Laboratory studies of graupel/crystal charge transfer in thunderstorm electrification
- **Best for**: Understanding the ice-graupel collision mechanism that creates charge regions
- **Worth reading if**: You need the physics behind charge generation

### Kamra (1997) - Humidity and Air Conductivity
- **URL**: https://rmets.onlinelibrary.wiley.com/doi/pdf/10.1002/qj.49712354108
- **Contents**: Effect of relative humidity on electrical conductivity of marine air
- **Best for**: Understanding how moisture affects breakdown threshold
- **Worth reading if**: You want physics-based moisture/charge interaction

### Williams (1989) - Tripole Structure of Thunderstorms
- **Contents**: Classic paper describing the three-layer charge structure (positive-negative-positive)
- **Best for**: Understanding vertical charge distribution in storms
- **Worth reading if**: You need the foundational model for cloud charge layers

### Rakov & Uman (2003) - Lightning Physics and Effects
- **Contents**: Comprehensive textbook on lightning physics (Cambridge University Press)
- **Best for**: Authoritative reference on all aspects of lightning
- **Worth reading if**: You need detailed technical information on any lightning phenomenon

### NOAA NWS - Stepped Leader Initiation
- **URL**: https://www.weather.gov/safety/lightning-science-initiation-stepped-leader
- **Contents**: Detailed explanation of how stepped leaders form and propagate
- **Best for**: Understanding leader mechanics for animation
- **Worth reading if**: You need step timing and propagation details

### NOAA NWS - Dart Leaders
- **URL**: https://www.weather.gov/safety/lightning-science-dart-leaders
- **Contents**: How dart leaders reuse ionization from previous strokes
- **Best for**: Understanding subsequent stroke behavior
- **Worth reading if**: You want realistic multi-stroke flash animation

### NOAA NWS - Continuing Current
- **URL**: https://www.weather.gov/safety/lightning-science-continuing-current
- **Contents**: The 40-500ms current flow after return strokes (100-1000A, orange glow)
- **Best for**: Post-stroke visual effects (continuing current glow)
- **Worth reading if**: You want the orange-red afterglow effect

### NOAA NWS - Electrification
- **URL**: https://www.weather.gov/safety/lightning-science-electrification
- **Contents**: How thunderstorms become electrically charged
- **Best for**: Understanding charge buildup mechanisms
- **Worth reading if**: You need charge field evolution physics

### ScienceDirect (2022) - Lightning Channel Initial Radius
- **URL**: https://www.sciencedirect.com/science/article/abs/pii/S0169809522001478
- **Contents**: Initial radius and discharge intensity of lightning return strokes
- **Best for**: Channel diameter values for rendering (1-4 cm core)
- **Worth reading if**: You need accurate channel geometry

### ScienceDirect (2018) - Channel Expansion Theory
- **URL**: https://www.sciencedirect.com/science/article/abs/pii/S1364682618306515
- **Contents**: Lightning channel expansion via supersonic shock in microseconds
- **Best for**: Understanding how the narrow channel expands
- **Worth reading if**: You want ionization dispersion physics

### AGU (2021) - Lightning Temperature Profile
- **URL**: https://agupubs.onlinelibrary.wiley.com/doi/full/10.1029/2020JD034438
- **Contents**: Vertical temperature profile of natural lightning return strokes
- **Best for**: Temperature decay timeline (30,000K peak to cool)
- **Worth reading if**: You want color/intensity decay curves

### Springer Plasma Physics (2025) - Lightning Channel Plasma
- **URL**: https://link.springer.com/article/10.1134/S1063780X25603025
- **Contents**: Plasma behavior in conducting channel, continuing current maintains ionization
- **Best for**: Understanding why dart leaders can reuse channels
- **Worth reading if**: You need ionization persistence physics

### AGU/GRL (2022) - Lightning Attachment Process
- **URL**: https://agupubs.onlinelibrary.wiley.com/doi/10.1029/2022GL101482
- **Contents**: Close view of how leaders connect to ground streamers
- **Best for**: Understanding the final connection mechanism
- **Worth reading if**: You want accurate ground-strike animation

### Wikipedia - St. Elmo's Fire
- **URL**: https://en.wikipedia.org/wiki/St._Elmo's_fire
- **Contents**: Corona discharge from pointed objects in high electric fields
- **Best for**: Understanding pre-strike ground phenomena
- **Worth reading if**: You want to add St. Elmo's fire effects

### Britannica - Bead Lightning
- **URL**: https://www.britannica.com/science/bead-lightning
- **Contents**: Rare phenomenon where lightning channel appears as string of beads
- **Best for**: Understanding unusual lightning forms (skip for now)
- **Worth reading if**: You want to simulate rare variations

### Southwest Research Institute - Seeing Thunder
- **URL**: https://www.swri.org/newsroom/technology-today/technology-today/seeing-thunder
- **Contents**: Visualizing acoustic wavefronts from lightning
- **Best for**: Thunder visualization (acoustic rings)
- **Worth reading if**: You want to add thunder wave effects

---

## Shader & Visualization Techniques (Added 2026-02-18)

### Quilez - Smooth Minimum
- **URL**: https://iquilezles.org/articles/smin/
- **Contents**: Smooth min function for blending distance fields (metaballs)
- **Best for**: Making charge blobs merge smoothly
- **Worth reading if**: You want clean metaball rendering

### Quilez - Fractal Brownian Motion
- **URL**: https://iquilezles.org/articles/fbm/
- **Contents**: FBM noise for natural-looking shapes
- **Best for**: Organic boundary warping in shaders
- **Worth reading if**: You want natural charge field boundaries

### The Book of Shaders - FBM
- **URL**: https://thebookofshaders.com/13/
- **Contents**: Interactive FBM tutorial with code
- **Best for**: Learning FBM implementation
- **Worth reading if**: You're new to procedural noise in shaders

### Wong (2016) - Metaballs and WebGL
- **URL**: https://jamie-wong.com/2016/07/06/metaballs-and-webgl/
- **Contents**: Metaball threshold math, GPU implementation
- **Best for**: Understanding metaball field computation
- **Worth reading if**: You want the math behind charge field merging

### Codrops (2025) - Interactive Metaballs with Three.js
- **URL**: https://tympanus.net/codrops/2025/06/09/how-to-create-interactive-droplet-like-metaballs-with-three-js-and-glsl/
- **Contents**: Modern Three.js metaball implementation
- **Best for**: Ready-to-use code patterns
- **Worth reading if**: You want current Three.js metaball examples

### Observable - GLSL Contour Lines
- **URL**: https://observablehq.com/@stwind/glsl-contour-lines
- **Contents**: Anti-aliased contour line rendering in fragment shaders
- **Best for**: Edge-emphasis rendering technique
- **Worth reading if**: You want clean threshold edges in charge fields

### ClickToRelease - Cross-hatching GLSL
- **URL**: https://www.clicktorelease.com/code/cross-hatching/
- **Contents**: Hatching shader for non-photorealistic rendering
- **Best for**: Alternative visualization styles
- **Worth reading if**: You want scientific/illustrative look

### GameIdea.org - Fresnel Effect GLSL
- **URL**: https://gameidea.org/short-posts/fresnel-effect-glsl/
- **Contents**: Edge glow based on view angle
- **Best for**: Rim lighting effects on charge regions
- **Worth reading if**: You want depth/edge enhancement

### Three.js Roadmap - Rim Lighting Shader
- **URL**: https://threejsroadmap.com/blog/rim-lighting-shader
- **Contents**: Rim lighting implementation in Three.js
- **Best for**: Adding edge glow to 3D charge regions
- **Worth reading if**: You want Three.js-specific rim lighting code

### InspirNathan - Glow Shader in Shadertoy
- **URL**: https://inspirnathan.com/posts/65-glow-shader-in-shadertoy/
- **Contents**: Simple glow effect implementation
- **Best for**: Corona/glow effects on lightning
- **Worth reading if**: You want leader tip corona envelope

### Shadertoy - Edge Glow Tutorial
- **URL**: https://www.shadertoy.com/view/Mdf3zr
- **Contents**: Interactive edge glow shader example
- **Best for**: Seeing glow techniques in action
- **Worth reading if**: You want to experiment with glow parameters

### LearnOpenGL - Bloom Post-Processing
- **URL**: https://learnopengl.com/Advanced-Lighting/Bloom
- **Contents**: Full bloom pipeline explanation
- **Best for**: Understanding post-process bloom
- **Worth reading if**: You want full-scene bloom effects

### Benjamin Cheng - Electric Fields with WebGL
- **URL**: https://bcheng.me/blog/visualising-electric-fields-with-webgl-kinda/
- **Contents**: Visualizing electric field lines in WebGL
- **Best for**: Alternative field visualization approaches
- **Worth reading if**: You want field line rendering

### HellerWeather - Data Visualization Color
- **URL**: https://hellerweather.com/data-visualization-and-the-overused-rainbow-color-table/
- **Contents**: Why rainbow colormaps are problematic, better alternatives
- **Best for**: Choosing perceptually accurate color schemes
- **Worth reading if**: You want scientifically sound color choices

### BAMS (2023) - Radar Visualization for Color Vision Deficiency
- **URL**: https://journals.ametsoc.org/view/journals/bams/105/8/BAMS-D-23-0056.1.xml
- **Contents**: Effective visualization strategies for accessibility
- **Best for**: Making visualization accessible to all users
- **Worth reading if**: You care about color accessibility

### Wikipedia - Line Integral Convolution
- **URL**: https://en.wikipedia.org/wiki/Line_integral_convolution
- **Contents**: LIC technique for visualizing vector fields
- **Best for**: Understanding flow visualization fundamentals
- **Worth reading if**: You want animated wind flow effects

---

## Particle Systems & GPGPU (Added 2026-02-18)

### Codrops (2024) - GPGPU Particle Effects
- **URL**: https://tympanus.net/codrops/2024/12/19/crafting-a-dreamy-particle-effect-with-three-js-and-gpgpu/
- **Contents**: Modern GPGPU particle system with Three.js
- **Best for**: High-performance particle rendering
- **Worth reading if**: You want GPU-accelerated wind streaks

### Three.js Journey - GPGPU Flow Field Particles
- **URL**: https://threejs-journey.com/lessons/gpgpu-flow-field-particles-shaders
- **Contents**: Flow field particle advection with GPGPU
- **Best for**: Wind streak particle system
- **Worth reading if**: You want particles that follow wind vectors

### anvaka/fieldplay - Vector Field Explorer
- **URL**: https://github.com/anvaka/fieldplay
- **Contents**: Interactive vector field visualization tool
- **Best for**: Experimenting with field visualizations
- **Worth reading if**: You want inspiration for wind visualization

### apbodnar/WebGL_LIC - Line Integral Convolution
- **URL**: https://github.com/apbodnar/WebGL_LIC
- **Contents**: WebGL implementation of LIC
- **Best for**: Flow visualization reference code
- **Worth reading if**: You want full LIC implementation

### philogb/LIC - LIC in JS/Canvas/WebGL
- **URL**: https://github.com/philogb/LIC
- **Contents**: Multiple LIC implementations
- **Best for**: Comparing different LIC approaches
- **Worth reading if**: You want simpler LIC options

### Will Usher - Volume Rendering with WebGL
- **URL**: https://www.willusher.io/webgl/2019/01/13/volume-rendering-with-webgl/
- **Contents**: Volume ray marching in WebGL
- **Best for**: True volumetric charge field rendering (expensive)
- **Worth reading if**: You want full 3D volumetric effects later
