# Globe Visualization Rendering Order Issues

## Context
We've successfully implemented a globe visualization that displays:
1. The Earth globe
2. Lightning strikes as zigzag lines emanating from the clouds
3. Point markers that persist after lightning strikes
4. A cloud layer at 0.053 altitude (approximately 1/3 of the original height)

However, we encountered rendering order issues where the cloud layer appeared to block out the lightning strikes and point markers, even though the Earth surface remained visible underneath the clouds. Additionally, the Earth glow effect seemed to emanate from the cloud layer rather than the Earth's surface.

## Findings

### Rendering Order and THREE.js Scene Graph
The issue was caused by the way THREE.js handles the rendering order of transparent objects:

1. **Default Rendering Order**: THREE.js typically renders objects in the order they're added to the scene, but this is complicated when transparent objects are involved.

2. **Transparency and Depth Testing**: When transparent objects are rendered, THREE.js still performs depth testing but doesn't automatically handle transparency layering correctly in all cases. This is why we could see the Earth through the clouds but not the lightning or point markers.

3. **Material Properties**: The cloud layer was using a `MeshPhongMaterial` with transparency enabled, which meant it was participating in depth testing but still blocking objects that were added to the scene after it.

## Solution Implemented

We implemented the following changes to fix the rendering order issues:

1. **Cloud Layer Adjustments**:
   - Added `depthWrite: false` to the cloud material to prevent it from writing to the depth buffer
   - Set `renderOrder = 10` for the cloud mesh

2. **Lightning Effect Adjustments**:
   - Added `depthWrite: false` to the lightning material
   - Set `renderOrder = 20` for the lightning group (higher than clouds)
   - Confirmed starting altitude matches the cloud layer (0.053)

3. **Point Marker Adjustments**:
   - Set `renderOrder = 30` for the point markers (highest priority)
   - Used the existing `depthWrite: false` setting

These changes established a clear rendering hierarchy:
- Earth globe: renderOrder = default (0)
- Cloud layer: renderOrder = 10
- Lightning strikes: renderOrder = 20
- Point markers: renderOrder = 30

## Results
The fix was successful. With these changes:
- The lightning strikes are now visible through the cloud layer
- The point markers appear above both the clouds and lightning strikes
- All elements respect their physical position on or above the globe
- The visual depth and layering appear correct

## Key Insights
1. THREE.js's rendering pipeline requires explicit control when working with multiple transparent objects
2. The `renderOrder` property is essential for establishing visual hierarchy
3. The `depthWrite` property allows objects to be visible through transparent layers
4. A small set of targeted changes can fix complex rendering issues without major refactoring

## Glow Effect Removal and Potential Reimplementation

As part of our refactoring, we removed the glow effect that was previously implemented in the LightningEffect class. Here's what was removed and how it could be reimplemented:

### Original Implementation
- The original glow was implemented using a THREE.PointLight attached to each lightning strike
- The light was positioned near the ground point of the lightning (at config.endAltitude * 0.5)
- It used a blue-white color (0x88ccff) with an intensity of 2.0 and distance of 5 units
- The glow would animate: intensify for 0.7 seconds, then fade out over 0.3 seconds
- This was controlled by a `showGlow` flag in the LightningManager

### Issues with the Original Approach
- Point lights were not being properly disposed when lightning effects ended
- Too many active lights could impact performance
- The glow appeared to emanate from the cloud layer rather than illuminating the strike path
- Lights were rendered based on physical position, not visual hierarchy

### Potential Reimplementation
1. **Alternative Light Setup**:
   - Instead of point lights, use a custom sprite/mesh with emissive materials
   - Position these along the path of the lightning to create a more distributed glow effect
   - Set proper renderOrder values to ensure it appears at the correct visual layer

2. **Post-Processing Option**:
   - Implement a bloom post-processing effect that would create a natural glow around bright objects
   - Assign high emissive values to lightning materials to make them bloom
   - This would require adding an EffectComposer from THREE.js

3. **Combined Approach**:
   - Use subtle point lights with short range for local illumination
   - Complement with glowing meshes along the strike path
   - Set appropriate render orders (between lightning [20] and markers [30])
   - Set depthWrite: false on all glow elements

The key insight from our current fix is that we need to properly manage the rendering order and depth writing properties to ensure all visual elements appear in the correct order regardless of their physical position in 3D space.
