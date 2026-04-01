---
status: active
priority: high
area: globe
---

# Globe View: Strike Scale and Appearance

## Problem

On the globe view, lightning strikes appear as short, fat, chunky squiggles when zoomed out. They don't look like lightning -- they look like thick scribbles. The line width and segment scale don't adjust properly for the globe's small rendering size.

## Goal

Strikes on the globe should look like fine, crisp lightning bolts at any zoom level. Thinner lines, more refined appearance. When zoomed out, strikes should be visible but delicate, not chunky blobs.

## Testing Challenge

This requires navigating to specific positions on the globe where strikes are occurring. The test approach should:
- Wait for the server to report strike locations
- Navigate the Puppeteer camera to a region with active strikes
- Capture at different zoom levels
- Compare line width relative to globe size

## Acceptance Criteria

- [ ] Strikes at default zoom look like fine lightning, not thick squiggles
- [ ] Line width scales appropriately with zoom level
- [ ] Strikes remain visible but not chunky when fully zoomed out
- [ ] Branching structure is visible, not lost in thick lines
