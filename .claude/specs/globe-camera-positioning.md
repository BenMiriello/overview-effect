---
status: active
priority: medium
area: globe
---

# Globe View: Camera Positioned at Active Strike Clusters

## Problem

The globe loads with a hardcoded camera position. It does a nice swoop animation on first load, but the position has no relation to where strikes are actually happening. The user sees empty ocean or land with no activity.

## Goal

On load, the globe should animate to the region with the most recent/prominent lightning activity. This requires:
1. The backend to track where strikes are clustering
2. Logic to determine the "hottest" region in a recent time window
3. The frontend to receive this position and animate the camera there

## Design Considerations

- **Clustering logic**: Consider a sliding time window (last 5-10 minutes). Weight recent strikes higher. Use spatial clustering (e.g., simple grid-based density or centroid of recent strikes).
- **Backend API**: The server should expose an endpoint or include in the data stream a "suggested view position" with lat/lng/zoom.
- **Animation**: Reuse the existing swoop animation but target the dynamic position.
- **Fallback**: If no recent strikes, use the hardcoded default.
- **Updates**: Optionally, periodically re-center if the user hasn't manually interacted (auto-tour mode).

## Acceptance Criteria

- [ ] Globe loads centered on a region with active lightning
- [ ] Backend provides strike cluster position data
- [ ] Camera smoothly animates to the active region
- [ ] Falls back to default position if no recent data
- [ ] Manual user interaction overrides auto-positioning
