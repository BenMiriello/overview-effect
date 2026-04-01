---
status: paused
priority: low
area: globe
---

# Globe View: Progressive Terrain Detail

## Problem

When zooming into the globe, the terrain texture becomes extremely blurry very quickly. There's no level-of-detail system -- it's a single texture stretched over the entire globe.

## Goal

Implement progressive terrain loading: as the user zooms in, load higher-resolution terrain tiles for the visible region. Like Google Maps but for a 3D globe.

## Research Needed

This is a significant feature requiring research into:
- Open-source tile servers (e.g., OpenStreetMap, Mapbox, Natural Earth)
- Tile pyramid formats (TMS, slippy tiles, quadtree)
- Three.js globe tile rendering (existing libraries?)
- Texture streaming and caching
- We want photographic/satellite imagery, NOT street maps or topographic

## This is a future feature -- do not start until higher-priority specs are complete.

## Acceptance Criteria

- [ ] Research doc with chosen tile source and approach
- [ ] Basic tile loading at 2-3 zoom levels
- [ ] Smooth transitions between detail levels
- [ ] Texture quality acceptable at city-level zoom
- [ ] No visible tile seams or loading artifacts
