# Distant Terrain Context → Revit Toposolid / D5 Terrain

**A free workflow for generating site-specific mountain backdrops for client renderings.**

Two routes off one shared QGIS pipeline, chosen by renderer:
- **Route A (Enscape / Revit-native):** USGS 3DEP DEM → QGIS (contours) → DXF → **Revit Toposolid**
- **Route B (D5 Render):** USGS 3DEP DEM → QGIS (16-bit heightmap PNG) → **D5 Terrain Tool**

Cost: $0. Tools: The National Map Downloader (free), QGIS (free), Revit 2027, D5 Render (already licensed).

> **Rev 2 — July 1, 2026.** Changes from Rev 1: clip moved *before* contouring (dissolves the Stage 2D selection snag); new elevation-normalization step (prevents terrain landing ~2,800m above the building); Revit positioning corrected to Center-to-Center (Origin-to-Internal-Origin will fail on UTM coordinates); 1 arc-second DEM now the default; D5 heightmap route added. Cesium/D5 streaming was evaluated and rejected — no Teams account or Cesium token.

---

## Reference: site coordinates

| Site | Lat / Long | View direction | Target context |
|---|---|---|---|
| Summerwood, Dillon | 39.606°N, -106.019°W | West, across Lake Dillon | Tenmile Range (down to ~39.47°N), Frisco |
| 49550 US-50, Gunnison | 38.512°N, -106.762°W | South | Ridges across the Gunnison/Tomichi valley |

Both sites fall in **UTM Zone 13N (EPSG:26913)** — the coordinate system you'll reproject to in QGIS.

> **Tip:** Start with Summerwood. Cleaner view, compact area. Once you've run it once, expect ~20 minutes per site.

---

## Stage 1 — Download the DEM from USGS

Source: **USGS 3DEP**, free and unrestricted.

**Default: the 1 arc-second DEM (~30m ground spacing).** At 5+ miles, your eye cannot distinguish 30m from 10m ridgeline fidelity, and 30m carries roughly 1/9th the data — which likely makes the polygon-count problem (and the deferred Grasshopper script) never happen at all.

*(If you've already downloaded the 1/3 arc-second / 10m tile for Summerwood, it works fine — the clip in Stage 2C keeps it manageable. Consider 1 arc-second for Gunnison.)*

1. **The National Map Downloader** → `apps.nationalmap.gov/downloader`
2. Left panel → **Datasets** → **Elevation Products (3DEP)** → check **1 arc-second DEM** (or 1/3 arc-second).
3. Pan/zoom to your site. Frame the box to include **both** the building location *and* the target mountains. For Summerwood, stretch west past the Tenmile ridgeline.
4. **Search by Map Extent** → **Search Products**.
5. Tile guide: Summerwood → 39–40°N / 106–107°W; Gunnison → 38–39°N / 106–107°W. If your view crosses a tile edge, grab both — merge in Stage 2A.
6. Download the **GeoTIFF (.tif)**.

> **What you've got:** a raster where every pixel's value is an elevation in meters. That's the whole trick of a DEM.

---

## Stage 2 — Process in QGIS (shared by both routes)

Install **QGIS** (free, `qgis.org` — Long Term Release).

### A. Load the data
- Drag the `.tif` file(s) into the canvas.
- **Two tiles?** `Raster → Miscellaneous → Merge` → select both → run. One seamless raster, no contour breaks at the seam.

### B. Reproject to a linear coordinate system
The DEM arrives in degrees of lat/long; both Revit and D5 need real distances.

- `Raster → Projections → Warp (Reproject)`
- **Target CRS:** `EPSG:26913` (UTM Zone 13N)
- **Resampling:** Bilinear
- Run. After this, 1 unit = 1 meter on the ground.

### C. Clip the raster — *before* anything else *(replaces the old contour-clipping step)*
Clipping the raster now, instead of selecting contours later, means every downstream product (contours or heightmap) is born already trimmed. No vector selection tools, no scratch polygons — the step that stalled Rev 1 no longer exists.

- `Raster → Extraction → Clip Raster by Extent`
- **Input:** the reprojected raster
- **Clipping extent:** click "Draw on Map Canvas" and drag a rectangle covering building + target mountains. For Summerwood: from the lake's east shore west through the Tenmile Range.
- Run.

> **Keeping this rectangle tight is the single biggest thing you can do for performance in Revit or D5.**
>
> **If you already generated full-tile contours in the Rev 1 run:** discard that contour layer. Clip the raster here, then regenerate contours in Stage 2E — it takes seconds and you skip the selection problem entirely.

### D. Normalize elevations to your site *(new — prevents the 2,800m-in-the-sky problem)*
The DEM carries true elevations — the Summerwood area sits roughly 2,700–2,800m above sea level, with Tenmile peaks near 4,200m. Your Revit internal origin (and D5's ground plane) is at the *building's* level 0. Without this step, imported terrain lands kilometers overhead.

1. Find your building's ground elevation: activate the clipped raster, use the **Identify Features** tool (the ⓘ cursor), and click on your site. The readout is the elevation in meters. Note it.
2. `Raster → Raster Calculator`
3. Expression: `"your_clipped_layer@1" - XXXX` where `XXXX` is that elevation.
4. Run. The output raster now reads **0 at your building**, with the mountains at their true *relative* heights above it.

> Terrain lower than your site (the lake surface, valley floors) will now carry small **negative** values. That's correct — Revit's Toposolid and D5's terrain both handle it fine.

**→ Route A (Revit/Enscape): continue to E–F. Route B (D5): skip to Stage 4.**

### E. Generate contours *(Route A only)*
- `Raster → Extraction → Contour`
- **Input:** the normalized raster from Step D
- **Interval:** **10** (meters); go **20** for distant peaks if the file feels heavy
- Confirm the elevation attribute field (default `ELEV`)
- Run.

### F. Export to DXF *(Route A only)*
- Right-click the contour layer → `Export → Save Features As`
- **Format:** AutoCAD DXF
- **CRS:** confirm `EPSG:26913`
- **Critical:** the export must keep **3D geometry** (Z-values). Flat contours in Revit = this setting was wrong.

---

## Stage 3 — Route A: Build the Toposolid in Revit 2027

1. **Link the DXF:** `Insert → Link CAD` (link, not import).
   - **Units:** Meters
   - **Positioning: Auto - Center to Center** ← *changed from Rev 1.* The contours carry UTM coordinates (values like 400,000 / 4,380,000 meters). "Origin to Internal Origin" tells Revit to honor those coordinates, and Revit refuses geometry that far from its internal origin. Center-to-Center drops the linked geometry at your model's center instead; nudge it into place by eye — camera-accurate is the whole standard here.
2. `Massing & Site → Toposolid → Create from Import → Select Import Instance` → click the linked DXF.
3. Choose the contour layer (carrying `ELEV`). Green check to finish.
4. Because Stage 2D normalized elevations, the terrain arrives at the right height relative to your building — no 2,800m vertical hunt.

Now it's a **native Toposolid**: assign materials, split into regions (snow caps / forested slopes / rock), and it casts and catches sun like any Revit element — visible to Enscape and to Revit sun studies alike.

---

## Stage 4 — Route B: Heightmap into D5's Terrain Tool

D5's Terrain Tool imports grayscale heightmaps and gives you sculpting and texture painting on the result — full material control, real USGS relief, no Revit weight, no Cesium account.

### A. Export the heightmap from QGIS
1. First, note two numbers from the **normalized** raster (Layer Properties → Information):
   - **Min and max elevation values** (e.g., −50 to +1,420)
   - **Extent dimensions in meters** (width × height of your clip — real meters, since we're in UTM)
2. `Raster → Conversion → Translate (Convert Format)`
   - **Input:** the normalized raster
   - **Output data type:** `UInt16`
   - Under **Advanced Parameters → Additional command-line parameters**, enter:
     `-scale MIN MAX 0 65535` (your actual min/max from step 1)
   - **Output file:** save as `.png`
   - Run.

> **Why 16-bit:** an 8-bit grayscale has only 256 height steps — across ~1,400m of relief that's visible terracing. 16-bit gives 65,536 steps: smooth slopes.

### B. Import and calibrate in D5
1. Open the D5 Terrain Tool → import the PNG as a heightmap.
2. **Calibrate with the two numbers from A1** — this is the step that keeps peaks from looking stretched or squashed:
   - Terrain **height range** = (max − min) from the raster, in meters
   - Terrain **horizontal size** = the clip extent's real width/height in meters
3. Position the terrain relative to your synced Revit model by eye through the camera — same camera-accurate standard as Route A.
4. Paint materials with D5's terrain texturing (snow, rock, forest) and scatter as desired.

> **Verify on first use** *(flagging honest uncertainty)*: whether D5's terrain patch enforces a square or capped footprint, and exactly which import fields its current version exposes for the height/size calibration — the tool's controls have been evolving across releases. If the patch must be square, simply draw a square clip rectangle back in Stage 2C. Two minutes of checking against your installed version beats trusting this doc's word for a moving target.

---

## Troubleshooting — the likely failure points

| Symptom | Cause | Fix |
|---|---|---|
| Revit rejects the link / geometry lands miles away | UTM coordinates + wrong positioning mode | Use **Auto - Center to Center** (Stage 3.1), move into place manually |
| Terrain floats ~2,800m above the building | Stage 2D normalization skipped | Run the Raster Calculator subtraction, regenerate contours/heightmap |
| Contours import **flat** in Revit | DXF exported as 2D | Re-export with Z preserved (Stage 2F) |
| Revit **grinds to a halt** | Too many contours | Clip tighter (2C), bump interval to 20m, or switch to the 1 arc-second DEM |
| D5 terrain looks terraced/banded | 8-bit heightmap | Re-export as UInt16 (Stage 4A) |
| D5 peaks look stretched or squashed | Height/size calibration mismatch | Re-enter the real min/max relief and extent dimensions from QGIS |
| Contours look **stepped** up close | Nature of 10–30m data | Fine for distant mountains; do **not** reuse for foreground site grading |

---

## If Revit still struggles with polygon count

That's the point where a small **Grasshopper / Rhino.Inside** mesh-decimation script would earn its place. Worth building only *after* the 1 arc-second DEM + tight clip still produces geometry too heavy for Revit. Not before — and with those two mitigations in place, likely never.

[[handoff]]
