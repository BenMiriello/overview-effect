# Globe Implementation and Data Fields - May 1, 2025

## Globe Implementation

### Context
We're implementing a 3D globe visualization in the Lightning project using react-globe.gl library. The goal is to start with a basic implementation that we can later extend with different data layers.

### Implementation Decisions
1. **Selected Library**: [react-globe.gl](https://github.com/vasturiano/react-globe.gl) by vasturiano - A React component for globe data visualization using ThreeJS/WebGL.

2. **Approach**:
   - Started with the most basic implementation (similar to the basic example)
   - Used full HTTPS URLs for resources to avoid loading issues
   - Handled errors that occur due to Three.js version incompatibilities

3. **Implementation Challenges**:
   - Encountered error: `Uncaught TypeError: object3.removeFromParent is not a function` - This is likely due to Three.js version mismatches
   - TypeScript declaration files missing, causing "Could not find a declaration file for module 'react-globe.gl'" warnings

4. **Solutions**:
   - Created a TypeScript declaration file (`react-globe.gl.d.ts`) to handle TS warnings
   - Used try/catch blocks to handle potential errors from Three.js API
   - Added state tracking with `onGlobeReady` to ensure we only access the globe after initialization
   - Used full HTTPS URLs instead of protocol-relative URLs for resources

5. **Visual Configuration**:
   - Used blue marble earth image for the globe
   - Added topographical bumps for relief
   - Set night sky as background
   - Implemented auto-rotation for better visual appeal
   - Set initial point of view to North America

### Future Enhancements
- Add multiple data visualization layers (points, arcs, polygons)
- Implement interactive features (clicking on locations, etc.)
- Add controls

## Data Field Understanding

After analyzing the WebSocket data, we've identified meanings for some of the lightning strike fields:

- **timestamp/time**: Time measurements in Unix epoch format (milliseconds and nanoseconds)
- **lat/lon/alt**: Geographic coordinates and altitude of the strike
- **pol**: Likely polarity (positive/negative charge)
- **mds**: Maximum Deviation in nanoseconds (shown as "max deviation" in the UI)
- **mcg**: Minimum Cycle Gap, likely measured in degrees (shown as "min cycle gap" in UI)
- **status**: Status code for the strike
- **region**: Geographic region identifier

There doesn't appear to be a direct intensity measurement in the current data structure, though intensity might be derivable from other fields or may require additional data.
