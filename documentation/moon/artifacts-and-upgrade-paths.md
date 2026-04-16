# Moon surface: artifacts and upgrade paths

Companion to `surface.md`. Tracks known visual artifacts, mitigation
options, and the roadmap for upgrading color and DEM source fidelity.

## Current artifact inventory

### Terraced contour rings on plains

Concentric step-like ridges visible on gentle mare basalt plains at close
zoom, especially near the terminator where grazing sun angle maximizes
shading contrast.

**Root cause:** The LOLA DEM from Trek WMTS is 8-bit grayscale. With
~20 km of lunar elevation range quantized into 256 discrete values, each
step is ~78 m. On steep terrain (crater rims, mountain slopes), real slope
variation far exceeds 78 m/texel and the quantization is invisible. On
plains, real slope is smaller than a single step — adjacent DEM pixels
snap to the same integer, and the boundary where the value jumps N to N+1
becomes a sharp local gradient that the fragment shader renders as a
ridge.

The per-fragment gradient normal mapping (commit `29789e4`, 2026-04-16)
amplifies this because it reads every discrete DEM integer boundary as a
real surface slope. Before that commit, the sphere-normal shading ignored
DEM entirely on the lit face, so the artifact was latent.

**Severity:** Moderate. Noticeable at close zoom on mare terrain. Does not
affect crater rims, highlands, or far-zoom viewing.

### Polar distortion

Equirectangular projection compresses tiles near the poles. The shader
handles this via cos(lat) scaling in `applyDemToMoonMaterial`; at the
geometric poles the tangent basis is degenerate and the shader falls back
to the sphere normal. No visible artifact at typical viewing angles but
polar close-ups show slight stretching.

**Severity:** Low. Poles are rarely viewed at close zoom.

## Mitigation options

### A: Widen gradient stencil — ACTIVE (N=2)

Sample DEM at +/-N texels instead of +/-1 when computing the finite
difference. Quantization boundaries get averaged across 2N texels.

- **Cost:** Zero perf impact, same 4 texture taps
- **Benefit:** Terraces soften from sharp ridges to gentle waves
- **Trade-off:** Crater rim shading loses slight sharpness (1-2 pixels)
- **Status:** Active at N=2. Alone (without C) had minimal effect — the
  real mitigation came from C.

### B: Soft-clamp per-texel slope — NOT NEEDED

Cap |slope| at a physically-reasonable max (tan(45deg) or higher). Real
lunar slopes rarely exceed that; anything steeper is likely quantization.

- **Cost:** Two clamp calls, zero perf impact
- **Benefit:** Kills remaining hard stair-step edges
- **Trade-off:** If cap is too aggressive, tall crater rims lose shadow
- **Status:** Not implemented. A+C was sufficient; holding in reserve if
  terracing reappears at deeper zooms or different sun angles.

### C: Pre-smooth DEM samples — ACTIVE (SMOOTH_RADIUS=3)

Each height sample is a 4-tap bilinear-offset average before the gradient
is computed. Offsetting by half a texel puts each tap on a corner where
hardware bilinear returns the 4-neighbor mean for free; scaling the
offset widens the effective footprint. SMOOTH_RADIUS=3 gives a ~6-texel
average per sample.

- **Cost:** 16 texture taps (up from 4). Acceptable on modern GPUs.
- **Benefit:** Very smooth plains; dequantizes the height signal so the
  gradient sees a continuous surface rather than 78 m stairsteps.
- **Trade-off:** Combined with wide stencil, crater rims can soften
- **Status:** Active at SMOOTH_RADIUS=3. User confirmed acceptable on
  plains with crater rims still reading sharply (2026-04-16).

### D: 16-bit DEM pipeline — DEFERRED

Replace Trek 8-bit WMTS tiles with locally-sliced 16-bit tiles encoded as
two-channel PNG (R=high byte, G=low byte, decode in shader). Root-cause
removal: 65536 steps over 20 km = 0.3 m/step.

- **Cost:** Significant (new slicer, new tile format, shader decode, LFS)
- **Benefit:** Complete artifact removal
- **Trade-off:** Day of implementation work; pairs with color upgrade (E)
- **Status:** Deferred until A/B/C results are evaluated

## Upgrade paths

### Color resolution upgrade (Step E)

Current: SVS 16K TIFF (16384 wide, ~760 m/px at L5, 909 MB download).
Upgrade: SVS full-resolution TIFF (27360 wide, ~450 m/px, ~2.6 GB).

Enables L6 tiles at ~1.2x upscale from native (vs 2x upscale from 16K).

Originally blocked on DEM upgrade (sharper color against quantized DEM
would make terracing more visible). That concern is resolved now that
Steps A+C mask the DEM quantization; color upgrade can proceed
independently.

Changes required:
- `client/scripts/build-moon-tiles.ts`: swap `SOURCE_URL` and `INPUT`
  filename, set `MAX_LEVEL = 6`
- `client/src/services/moonMesh.ts`: `MOON_COLOR_MAX_LEVEL = 6`,
  `MOON_DEM_MAX_LEVEL = 6` (only if DEM also upgraded)
- Regen: `rm -rf public/moon-tiles && npm run build:moon-tiles`
- Commit new tiles via LFS (dedupes unchanged L0-L5 blobs)

Expected: ~10920 tiles, ~150-200 MB in LFS. Download takes ~30 min from
NASA SVS (slow server, ~530 KB/s observed 2026-04-16).

### DEM precision upgrade (Step D)

See mitigation option D above. Full workflow TBD if we get there.

Key unknowns:
- Exact Trek URL for 16-bit LOLA GeoTIFF tiles (needs verification)
- Whether to use Trek's tiled WMTS or download a single full-globe
  GeoTIFF and slice locally (like we do for color)
- LFS size estimate for two-channel DEM tiles at L5

## Decision log

| Date | Decision | Outcome |
|------|----------|---------|
| 2026-04-16 | Implemented per-fragment DEM gradient normals (Step 1 of prior plan) | Craters show real directional shadows. Terracing artifact surfaced on plains. |
| 2026-04-16 | Switched color tiles to local SVS CGI Moon Kit (shadow-free Hapke) | Eliminated double-shadow artifact from Trek WAC mosaic. |
| 2026-04-16 | Committed tiles via git LFS (22 MB) instead of submodule | Simpler workflow, no extra repo, dedupes on regen. |
| 2026-04-16 | Capped at L5 / 16K source | Full-res 27K source only 1.67x wider; not worth 2.6 GB download while DEM is the visual bottleneck. |
| 2026-04-16 | Removed predev/prebuild hooks | Tiles in LFS; no auto-slice on npm run dev. |
| 2026-04-16 | Tried Step A (N=2) alone | Minimal effect. Widening the stencil moves sample points but each still reads a quantized integer, so magnitude of boundary jumps is unchanged. |
| 2026-04-16 | Added Step C (4-tap bilinear pre-smoothing per sample, SMOOTH_RADIUS=2) | Noticeable improvement but plains still showed some terracing. |
| 2026-04-16 | Bumped SMOOTH_RADIUS to 3 | Plains read as smooth. Crater rims still sharp. Accepted as sufficient. |
