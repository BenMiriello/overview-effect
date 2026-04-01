# Visual Refinement Guide

Living document for iterative visual improvement of the lightning simulation.
Updated continuously as work progresses. Not committed to git.

---

## Goal

Transform the simulation from a technical demo into something that looks and feels like a real thunderstorm scene. Every visual layer -- sky, clouds, atmosphere, charge fields, lightning bolts, ground, illumination -- should work together to create a convincing, atmospheric composition.

The simulation already has solid physics and rendering infrastructure. This work is about tuning, layering, and polishing the visual output until it reads as "stormy night sky with lightning" rather than "WebGL tech demo."

---

## Process: How to Iterate Effectively

### Capture -> Analyze -> Change -> Verify -> Commit

1. **Capture**: Use Puppeteer script (`node tmp/capture.mjs <label>`) to screenshot current state
2. **Analyze**: Look at the screenshot with fresh eyes. What's the most jarring issue?
3. **Prioritize**: Fix the highest-impact visual problem first (usually composition/color before detail)
4. **Change**: Make a focused code change addressing ONE issue
5. **Verify**: Recapture, compare before/after
6. **Commit**: If improved without regressions, commit with descriptive message
7. **Repeat**: Go back to step 1

### Self-Check Questions (ask after every change)

- Does this look MORE like a real thunderstorm? Or more like a tech demo?
- Am I seeing the scene as a whole, or fixating on one element?
- Is the change visible in the screenshot, or am I imagining improvement?
- Did I introduce any visual regressions? (check ALL layers, not just what I changed)
- Is the contrast/brightness appropriate? Can I see what matters?
- Would a non-technical person recognize this as a storm?

### Testing Discipline

- **Test ALL layer visibility combinations** -- toggle atmospheric, moisture,
  ionization, charge on/off. Each must look good alone and together.
- **Wait for strikes** -- don't just capture ambient. Catch leader, return
  stroke, decay, and post-strike recovery phases.
- **Check performance** -- watch for frame drops, stuttering, GPU memory.
- **Test at different speeds** -- slow motion reveals animation artifacts.
- **Check the globe view too** -- `/` route. Strikes should look clean at
  small scale without black trails or artifacts.
- **Think like a viewer** -- what would a non-technical person notice first?
  Fix that, not what's technically interesting.

### Mindset

This is not a checklist to complete. It's an ongoing craft of making a
simulation look and feel real. Keep thinking critically. Keep adapting.
Don't just fix reported issues -- proactively find and fix anything a
reasonable viewer would notice. The goal is: someone opens this page and
says "that looks like a real thunderstorm."

### When Stuck

- Re-read the relevant documentation (docs are thorough and contain hard-won lessons)
- Web search for reference images of real lightning / storm photography
- Web search for Three.js / GLSL techniques for the specific effect
- Step back and look at the overall composition before diving into shader details
- Check parameter-tuning.md for known failure modes
- Review the charge-field-visualization.md for architectural decisions that were already tried and failed

### What NOT to Do

- Don't tweak magic numbers without understanding the system (per CLAUDE.md)
- Don't optimize performance prematurely -- get it looking right first
- Don't add complexity when a simpler approach might work
- Don't forget to check the docs before re-attempting an approach that already failed
- Don't make changes to multiple layers at once -- isolate variables

---

## Architecture Overview

```
ShowcasePage.tsx          -- React wrapper with Canvas
  Scene.tsx               -- Three.js scene setup, camera, orbit controls
    GroundPlane.tsx        -- Shader-based ground with flash glow
    LightningController   -- Orchestrates timeline player + rendering
      TimelinePlayer      -- Web Worker that pre-computes strikes
      ChargeFieldRenderer -- Ceiling/ground metaball planes + volumetric fields
      LightningBoltEffect -- Per-strike rendering (BoltRenderer, FlashEffect, etc.)
```

Key files for visual changes:
- `ShowcasePage.tsx` / `ShowcasePage.css` -- Background, overall composition
- `Scene.tsx` -- Lighting, camera, scene-level effects
- `GroundPlane.tsx` -- Ground appearance
- `ChargeFieldRenderer.ts` -- Charge field rendering architecture
- `chargeFieldShaders.ts` -- GLSL shaders for charge planes + volumetrics
- `BoltRenderer.ts` -- Lightning bolt line rendering
- `LightningMaterials.ts` -- Bolt material/width settings
- `FlashEffect.ts` -- Lightning flash illumination
- `CoronaRenderer.ts` -- Glow around bolt
- `WindStreakRenderer.ts` -- Wind visualization

---

## Visual Layers & Priorities

### Priority 1: Scene Composition (Sky + Ground)

**Current State:** Pure black background, ground is invisible except during flash.
**Problem:** No sense of space, atmosphere, or environment. Lightning floats in void.

**Target:**
- Dark stormy sky gradient replacing pure black
- Ground plane with subtle base appearance (not just flash glow)
- Horizon line where sky meets ground
- Sense of depth and distance

**Approach:**
- Background: CSS gradient or full-screen shader quad behind the Canvas
- Ground: Add base color/noise to GroundPlane.tsx fragment shader
- Sky: Consider adding a sky dome or gradient mesh in Scene.tsx
- Colors: Deep navy (#0a0a1a) to dark gray (#1a1a2e) for sky

### Priority 2: Cloud Layer

**Current State:** No visible clouds. Ceiling is invisible plane where charge renders.
**Problem:** Lightning comes from nowhere -- no visual source.

**Target:**
- Dark, turbulent cloud mass at ceiling height
- Illuminated from below during lightning strikes
- FBM noise-based shapes with organic edges
- Partially transparent, layered

**Approach:**
- Add cloud mesh/shader plane at or near ceiling Y (1.5)
- FBM noise in fragment shader for cloud shape
- Uniform for flash intensity to illuminate during strikes
- Subtle animation (slow drift with wind)

### Priority 3: Charge Field Appearance

**Current State:** Contour lines with metaball merging (ceiling/ground), ray-marched volumes (atmospheric/moisture/ionization)
**Problem:** May look too technical. Need to verify with screenshot.

**Target per docs:**
- Continuous merged regions (not isolated circles)
- Irregular organic boundaries
- Wind stretch visible
- Subtle animation

**Tuning Points:**
- `chargeFieldShaders.ts`: threshold, edgeWidth, lineWidth, fadeWidth, opacity values
- `ChargeFieldRenderer.ts`: color values, opacity multipliers
- Volumetric: step count, alpha accumulation, boundary emphasis

### Priority 4: Lightning Bolt Quality

**Current State:** LineSegments2 with depth-based grouping, color decay white->orange->red
**Need to verify:** Does it look convincing? Width, brightness, branching.

**Tuning Points:**
- `LightningMaterials.ts` -- line widths per depth
- `BoltRenderer.ts` -- color transitions
- `BoltAnimator.ts` -- timing and brightness curves

### Priority 5: Flash & Illumination

**Current State:** PointLight flash + ground UV glow
**Target:** Flash that illuminates clouds, ground, and atmosphere together

**Tuning Points:**
- `FlashEffect.ts` -- intensity, falloff curve, duration
- `GroundPlane.tsx` -- glow radius, color, falloff
- Scene ambient light changes during strike

### Priority 6: Atmospheric Polish

- Rain particles (if implemented)
- Wind streaks
- Subtle fog/haze
- Post-processing (bloom, tone mapping)

---

## Key Parameters & Their Locations

| Parameter | File | Current | Notes |
|-----------|------|---------|-------|
| Background color | ShowcasePage.tsx | #000 (CSS) | Also in Canvas style prop |
| Ambient light | Scene.tsx:42 | 0.15 | Very dim |
| Camera position | ShowcasePage.tsx:27 | [0, 0, 6] | |
| Camera FOV | ShowcasePage.tsx:27 | 50 | |
| Ground Y | GroundPlane.tsx:101 | -1.8 | |
| Ceiling Y | LightningController.tsx:41 | 1.5 | |
| Ground glow radius | GroundPlane.tsx:44 | 0.15 (smoothstep) | |
| Charge opacity | ChargeFieldRenderer.ts:110 | 0.2 | |
| Ceiling color | ChargeFieldRenderer.ts:105 | (0.7, 0.85, 1.0) | Blue-white |
| Ground charge color | ChargeFieldRenderer.ts:106 | (0.9, 0.7, 0.5) | Warm orange |
| Contour thresholds | chargeFieldShaders.ts:83-86 | 0.2, 0.4, 0.6, 0.8 | |
| Volumetric steps | chargeFieldShaders.ts:151 | 24 | |
| Flash intensity | FlashEffect.ts:22 | config.intensity * 100 | |

---

## Research Notes

### Puppeteer + WebGL
- Must use `preserveDrawingBuffer: true` on Canvas GL context (done)
- Use `headless: 'shell'` mode for reliable WebGL on macOS
- `--use-angle=metal` flag for macOS GPU acceleration
- Wait for canvas element before capturing
- Multiple timed captures to catch different simulation states

### Three.js Visual Techniques (potential additions)
- UnrealBloomPass for glow effects (could add bolt glow without fog wash)
- Tone mapping (ACESFilmicToneMapping) for HDR look
- Particle systems for rain/debris
- Post-processing pipeline for bloom around bolt

### Lessons Learned
- Fog (THREE.FogExp2) causes additive-blended bolt segments to scatter
  brightness everywhere -- DO NOT use Three.js fog with additive blending
- ScreenFlashEffect (white overlay) at any visible opacity washes out the
  careful atmosphere. Disabled for now. Better approach: illuminate clouds
  via shader uniforms (already done in CloudLayer)
- PointLight for flash is unused (FlashEffect class exists but never
  instantiated). Ground/cloud flash works via CustomEvent dispatch
- Charge field contour lines look "topographic map" -- replaced with soft
  glow approach that blends into atmosphere
- Noise warp on charge field boundaries breaks circular symmetry nicely

---

## Progress Log

### Session 1 (2026-03-26)

**Completed:**
1. Puppeteer tooling: capture scripts (quick, burst), screenshots dir
2. SkyDome: FBM noise-based dark storm sky gradient sphere
3. CloudLayer: FBM cloud plane at ceiling Y with flash illumination
4. GroundPlane: FBM terrain noise with flash-revealed detail
5. Charge field shader: contour lines -> soft organic glow
6. Charge field shader: noise warp for irregular boundaries
7. Charge field: reduced opacity/brightness for atmospheric integration
8. Desaturated ground charge color
9. Fixed flash whiteout: disabled screen overlay, removed fog
10. preserveDrawingBuffer for WebGL captures

**Current State:**
- Scene reads as convincing dark stormy atmosphere
- Cloud layer has natural turbulent appearance with wind drift
- Charge fields blend subtly into clouds/ground (not tech-demo contours)
- Lightning bolt shape and branching look realistic
- Ground has subtle terrain texture
- Cloud illumination during strike works via shader uniform

**Remaining Work (Priority Order):**
1. More cloud depth/layers (single plane is flat from some angles)
2. Rain particles (documented but not yet implemented)
3. Wind streaks (renderer exists but visual quality unknown)
4. Bolt color transition during decay phase (white->orange->red)
5. Post-processing: bloom, tone mapping for HDR look
6. Globe view: verify strike rendering at small scale

**Fixed in this session:**
- Strike timing: removed charge throttling, fixed late geometry rescheduling
- Strikes now fire every 5-10 wall seconds (first at ~8s)
- Volumetric layers: fixed performance (24 steps -> 8), removed flat band
- Ground charge: cool blue-gray palette instead of warm orange
