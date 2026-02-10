# Implementing a Streamlined Data Stream Architecture

## Summary of Changes

In this session, we focused on implementing a clean data stream architecture for the Lightning application. The goal was to create a flexible system that could handle multiple data sources while maintaining a clear separation of concerns. We specifically wanted to avoid overengineering while still making the system extensible for future data types.

## Design Approach

We considered two main architectural patterns:

1. **Data Stream Registry Pattern**:
   - A centralized registry manages all data streams
   - Layers subscribe to named data streams
   - Provides central coordination but adds complexity

2. **Injection Pattern (Selected)**:
   - Layers receive data streams through dependency injection
   - Direct connection between data sources and visualization
   - Simpler approach with clearer relationships

We chose the second pattern for its simplicity and clear separation of concerns. This approach gives us the flexibility we need without unnecessary abstraction layers.

## Implementation Details

### 1. Core Data Stream Classes

We implemented a core infrastructure for data streams:

- **WebSocketService**: Base class handling WebSocket connections
  - Manages connection lifecycle (connect/disconnect)
  - Tracks connection status and updates
  - Handles reconnection logic

- **LightningDataStream**: Extended WebSocketService for lightning data
  - Inherits connection management from WebSocketService
  - Adds lightning-specific data validation and processing
  - Manages lightning-specific subscriptions

The inheritance approach eliminated redundant code and created a clean architectural relationship between the classes.

### 2. Layer Integration

We enhanced the LightningLayer to work with the new data stream approach:

- Added a setDataStream method to inject a data stream
- Implemented proper subscription management
- Ensured clean resource cleanup when the layer is disposed

This allowed layers to receive data from their corresponding data streams without creating or managing the connections themselves.

### 3. React Integration

To make the system React-friendly, we created:

- A useLightningStream hook to easily use LightningDataStream in components
- Clean state management for connection status
- Proper lifecycle handling with useEffect and useCallback

The App component was updated to:
- Use the hook to create and manage the data stream
- Pass the data stream to the lightning layer
- Handle subscription and cleanup properly

### 4. Architectural Improvements

We made several architectural improvements:

- **Simplified Code**: Removed redundant delegation by using inheritance
- **Clear Separation**: Separated WebSocket logic from lightning-specific code
- **Removed Duplication**: Eliminated duplicated connection management code
- **Minimized Comments**: Removed unnecessary explanatory comments

## Benefits of the New Architecture

1. **Clean Separation of Concerns**:
   - Transport logic is separated from data processing
   - Data acquisition is decoupled from visualization

2. **Extensibility**:
   - New data types can easily extend WebSocketService
   - Layers can connect to any compatible data stream

3. **Simplified Maintenance**:
   - Code is more concise and easier to understand
   - Relationships between components are clearer

4. **Scalability**:
   - Architecture supports adding multiple data sources
   - Can be extended to different transport mechanisms (HTTP, files, etc.)

## Next Steps

The new architecture provides a solid foundation for future enhancements:

1. **Additional Data Types**:
   - Implement earthquake, weather, or other data streams using the same pattern
   - Create corresponding layers to visualize the new data

2. **Transport Options**:
   - Add HTTP polling transport for REST APIs
   - Implement file loading for historical data

3. **UI Controls**:
   - Add UI to toggle different data sources
   - Allow configuration of data stream parameters

4. **Performance Optimizations**:
   - Implement data buffering for high-frequency streams
   - Add spatial indexing for efficient lookups

This architecture balances current needs with future extensibility, providing a clean foundation for expanding the visualization capabilities of the application.


## User added Notes:

- In the future, have it load up with recent history of strike data already showing (should be in blitzortung data somewhere but not the ws)
- Use that data to have the globe load up to show the area with the most recent activity

- Add paths of satellites like ISS
- Allow viewing earth from POV of ISS or other satellites
   - In this view you'll be able to look around by dragging so you can see close up but looking to the side (or down too)

- Add sun, day/night cycle, sunset glow

- Update point marker look, maybe just a circle - Yellow-white circle for lightning.

- try adding back glow
