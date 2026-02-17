# Lightning Physics Reference

Hard facts about real lightning that inform simulation and animation parameters.

## Stepped Leader

- **Step size**: ~50 meters
- **Step duration**: ~1 microsecond
- **Pause between steps**: 20-50 microseconds
- **Total leader duration**: 10-20 milliseconds
- **Sensing range**: Tip only "sees" charges within ~50m (why path is stochastic)
- **Speed**: ~200,000 mph (with pauses), would be faster without pauses

## Branching Mechanism

Branching is NOT random decoration. It comes from **space leaders**:
1. "Space stems" form in the streamer zone ahead of the main channel tip
2. Some evolve into "space leaders" (ionized channels)
3. Multiple space leaders form **in parallel** during a single step
4. Space leaders connecting to the **tip** extend the channel
5. Space leaders connecting to **lateral surface** create branches

This parallel formation is why lightning both branches AND zigzags.

## Branch Angles

- **Mean angle**: 16-20 degrees from parent direction
- **Distribution**: Tight, most branches within 12-28 degrees
- Branches that deviate more extremely tend to die quickly

## Main Channel Determination

The "main channel" doesn't exist until ground connection:
- All branches are potential main channels during leader phase
- First path to reach ground wins
- That path becomes the main channel for return stroke
- Other paths remain as visible branches

## Return Stroke

- **Trigger**: Only occurs when connection to ground is made
- **Direction**: Travels UP from ground
- **Speed**: ~1/3 speed of light (~100,000 km/s)
- **Duration**: ~100 microseconds
- **Brightness**: Orders of magnitude brighter than leader
- **Appearance**: Single bright channel (the branching structure isn't visible during stroke)

## Visual Characteristics

| Phase | Brightness | Branching Visible | Duration |
|-------|------------|-------------------|----------|
| Stepped leader | Dim | Yes, extensive | 10-20ms |
| Return stroke | VERY bright | No, single channel | ~100μs |

## Implications for Simulation

1. Leader phase should show multiple exploring paths, not one predetermined snake
2. Leader should be dim with bright tip only
3. Return stroke should illuminate only the winning path
4. Branches should have natural exponential length distribution (many short, few long)
5. Animation timing: leader ~800ms (scaled), return stroke ~80ms (scaled)
