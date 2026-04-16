# Moon surface rendering

End-to-end pipeline: source selection, tiling, storage, and shading of the
lunar surface. Covers the color mosaic, the DEM (digital elevation model),
how they combine in the shader, and the known artifact modes.

## Rendering approach

The moon is rendered as a tiled sphere (`SlippyMapGlobe`, equirectangular
projection). Each tile is a sphere patch whose vertices are displaced
radially by DEM samples (real geometric topography — silhouette moves) and
whose fragments compute per-pixel normals from the DEM gradient for
crater-scale shading (the lit face shows directional shadows that move
with the sun).

Key code:
- `client/src/services/moonMesh.ts` — builds the color and DEM tile engines,
  wires DEM fetch onto color-tile load.
- `client/src/services/moonMaterial.ts` — vertex displacement, fragment
  gradient normal reconstruction, sun dot shading.

## Illumination requirement: shadow-free color source

The fragment shader's Lambert sun term is the **only** source of shading on
the moon. That means the color mosaic must have no baked-in illumination —
otherwise we get doubled shadows that don't move with our simulated sun.

Trek's `LRO_WAC_Mosaic_Global_303ppd_v02` has solar shading baked into
pixel values and is therefore unusable here despite its convenient WMTS
availability. We instead slice a Hapke photometrically normalized mosaic
(every pixel rendered as if lit at `i=60deg, e=0deg, g=60deg`) and serve
it locally.

## Color tiles

### Source

- **Dataset**: NASA SVS "CGI Moon Kit" 16K color TIFF, derived from the
  LROC team's Hapke-normalized WAC mosaic with polar caps filled by LDAM
  albedo.
- **URL**: <https://svs.gsfc.nasa.gov/vis/a000000/a004700/a004720/lroc_color_16bit_srgb_16k.tif>
- **Size**: 909 MB, 16384 x 8192, 16-bit sRGB, equirectangular.
- **License**: NASA public domain.

A full-resolution variant (`lroc_color_16bit_srgb.tif`, 27360x13680,
~2.6 GB) exists in the same directory. Not currently used — see
**Resolution ceiling** below.

### Tile pyramid

Sliced into 6 levels (0..5) of 256x256 JPEGs at
`client/public/moon-tiles/{level}/{y}/{x}.jpg`. Convention matches
`SlippyMapGlobe` equirectangular: `gx = 2 * 2^L` columns, `gy = 2^L` rows,
NW pixel origin. Level 5 (~760 m/px) reaches source-native resolution.

Total output: 2730 tiles, ~22 MB.

### Slicer

`client/scripts/build-moon-tiles.ts` downloads the source on first run,
caches it in `client/vendor-assets/`, then produces all levels in a single
pass. Logs per-step elapsed time and source metadata; refuses to produce
upscaled levels with a warning.

Regeneration is not automatic — tiles are in the repo. Run manually only
when bumping the source or slicer parameters:

```sh
rm -rf client/public/moon-tiles
npm --prefix client run build:moon-tiles
git add client/public/moon-tiles
git commit -m "Regenerate moon tiles (<reason>)"
```

## DEM tiles

### Source

NASA Moon Trek WMTS, served live (no local slicing required):
`LRO_LOLA_DEM_Global_128ppd_v04` at `default028mm/{level}/{y}/{x}.png`.

- **Projection**: equirectangular
- **Bit depth**: 8-bit PNG (root cause of the terrace artifact — see below)
- **Native resolution**: 128 pixels per degree (~330 m/px at equator at
  native, but tiles quantized by level)

### Level binding

DEM and color max levels are locked together at L5 in `moonMesh.ts`. Every
color tile loads a matching DEM tile at the same (x, y, level). Decoupling
would require compositing multiple DEM tiles per color tile — unnecessary
while both support the same max.

## Storage: Git LFS

Tiles are committed to the repo via git LFS (see `client/.gitattributes`).
LFS stores binary blobs out-of-band so git pack size stays small.
Regenerating the pyramid produces identical bytes for unchanged regions;
LFS deduplicates by content hash so repeated regens don't bloat history.

First-time clone requires LFS:

```sh
brew install git-lfs   # or equivalent
git lfs install        # one-time
git clone <repo>       # LFS fetch happens automatically
```

Cloudflare Pages pulls LFS blobs during deploy automatically.

The slicer writes `public/moon-tiles/.complete` on success. That sentinel
is gitignored — it's dev-machine state, not repo content.

## Resolution ceiling

Color caps out at L5 because the 16K source goes native at that level.
Pushing to L6 from the 16K source is pure bilinear upscale — no new
information. The full-res 27K source would enable L6 at only ~1.2x upscale
(marginally real detail), but:

1. The DEM is the binding constraint, not color. LOLA 128 ppd is ~330 m/px
   native; higher color detail without matching DEM detail just makes the
   terrace artifacts (below) more visible against a sharper albedo.
2. A true resolution upgrade requires upgrading **both** color and DEM.
   See `artifacts-and-upgrade-paths.md` (WIP) for the full picture.

## Known artifacts

### Terraced contour rings on plains

Concentric ring patterns visible on gentle mare basalt plains, especially
at close zoom. Root cause: the LOLA DEM is 8-bit, so ~20 km of total
elevation range quantizes into ~80 m steps. On steep terrain (crater rims)
these steps are invisible; on plains the actual slope is smaller than a
single step, and the integer-value boundary between adjacent plateaus
becomes a sharp local gradient that the fragment shader renders as a
ridge.

The fragment-shader gradient normal mapping (introduced to make craters
read as 3D) amplifies this artifact because it treats every discrete DEM
jump as a real slope. Mitigation strategies are tracked in
`artifacts-and-upgrade-paths.md`.

### Polar distortion

Equirectangular projection compresses tiles near the poles. Cos(lat)
scaling in `applyDemToMoonMaterial` handles the horizontal stretch in
the gradient computation; at the geometric poles the tangent frame is
degenerate and the shader falls back to the sphere normal.

## Future upgrade paths

Tracked in `artifacts-and-upgrade-paths.md`. Summary of dimensions that
can independently improve:

- Color resolution (swap to 27K source, bump to L6)
- DEM bit depth (upgrade from 8-bit PNG to 16-bit encoding)
- DEM resolution (upgrade to higher-ppd LOLA variant)
- Shader smoothing (wider gradient stencil, gradient clamp)

Do shader smoothing first — cheapest and directly targets the worst
visible artifact.
