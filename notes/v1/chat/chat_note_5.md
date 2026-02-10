# Lightning Optimization Challenges - May 3, 2025

## Summary of Current Implementation and Issues

We've been working on improving the lightning strike visualization while also optimizing for performance. However, our latest implementation has significant performance issues that need addressing:

1. **Performance Problems**:
   - Significant lag after just 5-15 strikes
   - The globe surface disappears after accumulating strikes (approximately 250)
   - Performance may be worse than the initial implementation

2. **Visual Issues**:
   - Lightning strike squiggles are persisting and not disappearing as requested
   - Glow effect looks good but is causing severe performance problems

3. **Current Implementation Approaches**:
   - Object pooling for lightning effects (partially implemented)
   - Point lights for glow effects (very performance-intensive)
   - Persistent circle markers for historical strikes

## Requirements for Next Implementation

1. **Visual Requirements**:
   - Lightning effect should consist of squiggles and glow
   - Strike animation should appear then fade out after 1-2 seconds
   - Small circle marker (1/4 size of original) should remain after the strike
   - Glow should fade out after 3-5 seconds

2. **Performance Requirements**:
   - Must handle hundreds of strikes without lag
   - Must prevent globe surface from disappearing
   - Needs to reuse objects efficiently

## Optimization Ideas to Explore

1. **Remove individual point lights completely**:
   - Point lights in Three.js are extremely expensive
   - Replace them with a single shared light or use a shader-based glow effect
   - Consider using a post-processing bloom effect instead of individual point lights

2. **Implement true instanced rendering**:
   - Use Three.js InstancedMesh for all persistent circle markers
   - Avoid creating individual meshes for each strike point

3. **Limited strike capacity**:
   - Implement a hard cap on concurrent active lightning strikes (e.g., 20-30 max)
   - Remove older strikes when new ones arrive to maintain this cap

4. **Simplified geometry**:
   - Reduce polygon count of all geometries
   - Use lower resolution for circles and lightning

5. **Dynamic LOD (Level of Detail)**:
   - Simplify or disable effects for strikes that are not in the camera's current view
   - Add distance-based detail reduction

## Critical Resource Links for Next Implementation

1. Three.js instanced mesh documentation:
   - https://threejs.org/docs/#api/en/objects/InstancedMesh

2. Post-processing bloom effect for glow without point lights:
   - https://threejs.org/examples/#webgl_postprocessing_unreal_bloom

3. Three.js performance tips:
   - https://threejs.org/manual/#en/optimize-lots-of-objects

4. Globe.gl API documentation:
   - https://github.com/vasturiano/globe.gl

## Instructions for Next Steps

1. Focus first on fixing the persisting squiggle lines issue - they must disappear after 2 seconds
2. Eliminate all point lights and replace with alternative glow techniques
3. Implement true instanced rendering for the circle markers using Three.js InstancedMesh
4. Add a hard cap on concurrent active effects (20-30) and implement proper object recycling
5. Test with incremental changes, measuring performance impact of each change

The lightning effect should be visually appealing with a bright glow, but performance must be the top priority. A less flashy effect that runs smoothly is preferable to a beautiful effect that causes the page to become unresponsive.
