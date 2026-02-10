# Globe Lightning Visualization Implementation & Optimization - May 2, 2025

## Implementation Summary
We successfully implemented a globe-based visualization for real-time lightning strikes using react-globe.gl and Three.js. The key features include:

1. **Real-time Lightning Visualization**:
   - Visualizing lightning strikes as they occur, with no initial batch loading
   - Each strike appears as a white point with a yellowish glow
   - Strikes remain visible for 5 minutes, with animation phases

2. **Animation Sequence**:
   - Initial phase: Large point (0.4 degrees) for 3 seconds
   - Transition phase: Shrinks to 1/5 size (0.08 degrees) over 3 seconds
   - Lingering phase: Remains visible at small size for ~4:54 minutes
   - Total lifespan: 5 minutes per strike

3. **Architecture Improvements**:
   - Created separate files for strike model and WebSocket service
   - Improved TypeScript typing with proper interfaces
   - Increased strike limit to 10,000

4. **Server Modifications**:
   - Removed initial data load to ensure we start with zero strikes
   - Only forwarding live strikes as they're detected

## Performance Optimization Priorities

Performance degradation becomes noticeable after several hundred strikes are rendered. Here are the optimization steps in order of priority:

### 1. Instanced Rendering
- **Description**: Use Three.js InstancedMesh to render many similar objects with a single draw call
- **Expected Benefit**: Dramatic reduction in draw calls, potentially enabling thousands of strikes with minimal performance impact
- **Implementation Approach**: Replace individual mesh objects with a shared geometry and instanced rendering

### 2. Freeze Animations for Older Strikes
- **Description**: Stop updating strikes once they've completed shrinking animation
- **Expected Benefit**: Reduces per-frame calculations for the majority of visible strikes
- **Implementation Approach**: Flag strikes as "static" after the initial animation phases and exclude them from frame-by-frame updates

### 3. Level of Detail (LOD)
- **Description**: Use simpler geometries for strikes based on distance from camera
- **Expected Benefit**: Reduces polygon count for distant objects
- **Implementation Approach**: Adjust geometry detail dynamically based on camera distance

### 4. Object Pooling
- **Description**: Reuse Three.js objects instead of creating new ones
- **Expected Benefit**: Reduces garbage collection and object creation overhead
- **Implementation Approach**: Maintain a pool of pre-created objects to be repositioned and reused

### 5. Spatial Partitioning
- **Description**: Only render strikes within the current view frustum
- **Expected Benefit**: Avoids rendering objects not currently visible
- **Implementation Approach**: Implement a spatial index structure (quadtree/octree) to efficiently query visible strikes

### 6. Admin Panel
- **Description**: Simple performance monitor for framerate and statistics
- **Implementation Approach**: Add Three.js Stats panel in a corner, toggleable via a flag in the code

## Future Enhancements
These optimizations should be considered for later implementation:

1. **Batch Processing**: Group updates to reduce state changes
2. **WebGL Optimizations**: Use shared materials to reduce shader switches
3. **WebWorkers**: Move animation calculations to background threads
4. **Strike Clustering**: Merge nearby strikes when zoomed out

By implementing these optimizations incrementally, we'll be able to monitor the impact of each change and ensure the visualization remains performant with thousands of strikes.

## User Notes:
- For optimization, both in performance and visuals, we can implement a system where multiple strikes are aggregated together when zoomed out, and shown individually when zoomed in. At the same time the size of the strike can be variable based on the zoom distance, with some max and min size possible, so that the max is still smaller than the original fresh strike size.

- TODO: get user current location and start the map there.
- TODO: set it to rotate automatically on load but not after they click or move it. Start out in wide view. Possibly only for the first time they load the page, just to get them to start interacting with it.
- TODO: Less spin and slower spin in start animation
- Add third state for "Connecting to Server" instead of starting off as "Disconnected" (also for when attempting to reconnect - and add more automatic attempts to reconnect, intermittent, more frequent at first, then slower)
