# Distant Terrain Context → Revit Toposolid / D5 Terrain — Rev 3

**A free workflow for generating site-specific mountain backdrops for client renderings.**

Two routes off one shared QGIS pipeline, chosen by renderer:

- **Route A (Enscape / Revit-native):** USGS 3DEP DEM → QGIS (**points CSV**) → **Revit Toposolid (Create from Import > Points File)**
- **Route B (D5 Render):** USGS 3DEP DEM → QGIS (16-bit heightmap PNG) → **D5 Terrain Tool**

Cost: $0. Tools: The National Map Downloader (free), QGIS (free), Revit 2027, D5 Render (already licensed).

> **Rev 3 — July 7, 2026.** Route A rebuilt: the toposolid is now generated from a **points CSV** instead of contour linework in a DXF. Why: Revit only samples points at contour polyline *vertices* and triangulates straight lines between them, so the resulting surface cuts corners through curves and lands points off the contours — the cleanup problem that motivated this revision. A CSV import places a toposolid point at *every exact XYZ in the file*. No interpretation, no vertex sampling, no cleanup. The contour → DXF route is retained as a **documented fallback** (Appendix A), primarily for civil-engineer-supplied DWGs where a DEM isn't the source.
>
> **New critical step:** XY rebasing in QGIS (Stage 3A-3). The CSV import has no positioning dialog — the Center-to-Center trick from Rev 2 doesn't exist for points files. Raw UTM coordinates (~400 km from origin) would blow past Revit's distance-from-origin limit, so X and Y are now rebased to the site in QGIS, the same way elevations already are.

---

## Reference: site coordinates

| Site | Lat / Long | View direction | Target context | UTM zone |
|---|---|---|---|---|
| **Summerwood**, Dillon | 39.606°N, -106.019°W | West, across Lake Dillon | Tenmile Range (down to ~39.47°N), Frisco | 13N (EPSG:26913) |
| **49550 US-50**, Gunnison | 38.512°N, -106.762°W | South | Ridges across the Gunnison/Tomichi valley | 13N (EPSG:26913) |

> **Tip:** Summerwood remains the learning case. Once run once, expect ~20 minutes per site — Rev 3 is slightly *faster* than Rev 2 because contour generation and DXF export are gone.

---

## Stage 1 — Download the DEM from USGS

*(Unchanged from Rev 2.)*

Source: **USGS 3DEP** via The National Map Downloader. Free and unrestricted.

**Default: the 1 arc-second DEM (~30 m ground spacing).** More than enough for distant context, and it keeps everything downstream light. The 1/3 arc-second (~10 m) product is available if a foreground ridge ever needs more definition, but start at 30 m.

1. Go to The National Map Downloader (apps.nationalmap.gov/downloader).
2. Zoom to the site area. Draw an extent generously — clipping happens later in QGIS.
3. Under Elevation Products (3DEP), check **1 arc-second DEM**, format GeoTIFF.
4. Search products → download the tile(s) covering the site *and* the target terrain (for Summerwood, the tile must reach the Tenmile Range to the west/southwest).

---

## Stage 2 — Shared QGIS processing (both routes)

*(Same sequence as Rev 2, with one addition to Stage 2B: set the output resolution here — this is Rev 3's point-thinning control.)*

### 2A — Load the DEM

Drag the GeoTIFF into QGIS. It arrives in EPSG:4326 (lat/long degrees) — do not use it in this state.

### 2B — Reproject to UTM **and set resolution**

`Raster → Projections → Warp (Reproject)`

- **Target CRS:** EPSG:26913 (NAD83 / UTM Zone 13N)
- **Resampling method:** Bilinear
- **Output file resolution in target georeferenced units:** ← **NEW in Rev 3.** This one field controls the final toposolid point count. See the density table in Stage 3A-2 before choosing; **90 is the recommended default for distant context.** Leave blank only if you deliberately want native ~30 m density (heavy).

> **Why here:** every pixel of the raster becomes exactly one toposolid point in Route A. Setting resolution during the reproject means one dialog does two jobs, and everything downstream (clip, normalize, convert) inherits the thinned grid automatically. Route B is unaffected — heightmaps tolerate any resolution.

> **Reprojection is still mandatory** — skip it and the terrain arrives stretched. This has not changed.

### 2C — Clip the raster

`Raster → Extraction → Clip Raster by Extent`

Draw the extent covering the site through the target terrain (Summerwood: lake's east shore through the Tenmile Range). **Clip tight** — every excess kilometer is wasted points (Route A) and wasted pixels (Route B).

### 2D — Normalize elevations

The DEM carries true elevations (site ~2,700–2,800 m ASL, peaks ~4,200 m). Left alone, the terrain imports kilometers above the building.

1. **Identify tool** on the site location → note the elevation value (call it `Z₀`).
2. `Raster → Raster Calculator`: `"clipped_layer@1" - Z₀` → save as `terrain_normalized`.

Negative values below the site are correct and harmless. The site itself now sits at elevation ≈ 0, matching the Revit project's ground reference.

---

## Stage 3, Route A — Points CSV → Revit Toposolid  ★ NEW

### 3A-1 — Get the site's UTM coordinates (for XY rebasing)

Hover the cursor over the site location and read the **Coordinate** display in the QGIS status bar (bottom of the window — confirm the project CRS is EPSG:26913 so the readout is in meters). Note both numbers, rounded to whole meters:

- `X₀` = easting (Summerwood ≈ 6-digit number starting ~4xx,xxx)
- `Y₀` = northing (≈ 7-digit number starting ~4,3xx,xxx)

Log these in the handoff doc per site — they're the site's permanent rebasing constants for any future re-run.

> **Why:** Revit places CSV points at their literal coordinates relative to the internal origin, and there is **no positioning dialog** for points files. UTM coordinates sit hundreds of kilometers out — far past Revit's geometry-distance limit. Subtracting `X₀`/`Y₀` puts the site at (0, 0), so the terrain lands centered on the internal origin. This is the XY twin of the elevation normalization in 2D: after both, the site sits at (0, 0, 0).

### 3A-2 — Confirm the point budget

Rough point count = (clip width ÷ resolution) × (clip height ÷ resolution). Check against this table *before* converting — if it's too heavy, re-run Stage 2B with a coarser resolution rather than fighting a bloated toposolid in Revit.

| Clip extent | Resolution (Stage 2B) | Point count | Verdict |
|---|---|---|---|
| 10 × 10 km | 30 m (native) | ~111,000 | Too heavy — Revit will chug |
| 10 × 10 km | 60 m | ~28,000 | Workable ceiling |
| 10 × 10 km | **90 m** | **~12,000** | **Recommended for distant context** |
| 10 × 10 km | 120 m | ~7,000 | Fine for far ridgelines only |
| 5 × 5 km | 30 m (native) | ~28,000 | Workable if foreground detail matters |

Rule of thumb: **stay under ~20,000 points; target 8,000–15,000.** Terrain kilometers from the camera does not reward 30 m fidelity.

### 3A-3 — Convert raster pixels to points

`Processing Toolbox → Vector creation → Raster pixels to points`

- **Input:** `terrain_normalized` (the Stage 2D output — elevations already rebased)
- Output: a point layer with one attribute, `VALUE` (the normalized elevation)

Right-click the layer → *Show Feature Count* to verify the point budget matches the table above.

### 3A-4 — Rebase X and Y (Refactor Fields)

`Processing Toolbox → Vector table → Refactor fields`

Delete the existing `VALUE` row's default mapping and build exactly three output fields, **in this order** (Revit reads columns positionally as X, Y, Z):

| Field name | Type | Expression | What it does |
|---|---|---|---|
| `x` | Decimal (double) | `$x - X₀` | Easting rebased to site (plain-language: "this point's east-west position, measured from the site instead of the UTM zone origin") |
| `y` | Decimal (double) | `$y - Y₀` | Northing rebased to site |
| `z` | Decimal (double) | `"VALUE"` | Elevation — already normalized in Stage 2D, passes straight through |

`$x` and `$y` are QGIS's built-in variables for each point's own coordinates — no typing coordinates per point, the expression runs across all points automatically. Substitute the actual numbers noted in 3A-1 for `X₀`/`Y₀`.

Run → output is a new layer (`Refactored`).

### 3A-5 — Export the CSV

Right-click the refactored layer → `Export → Save Features As…`

- **Format:** Comma Separated Value (CSV)
- **Geometry:** *No geometry* (the x/y/z columns carry everything; exporting geometry too would add duplicate coordinate columns and confuse the column order)
- **Fields:** confirm only `x`, `y`, `z`, in that order
- Save as e.g. `summerwood_terrain_points.csv`

> ⚠️ **Uncertainty flag — header row.** QGIS writes a header line (`x,y,z`) at the top of the CSV. I have not verified whether Revit 2027's points-file parser tolerates a header or errors on it. **If the import fails or throws a parse error: open the CSV in Notepad, delete the first line, save, retry.** Resolve this on the Summerwood run and update this doc with the answer.

### 3A-6 — Import into Revit

1. `Massing & Site → Toposolid → Create from Import → Points File`
2. Select the CSV.
3. **Units: meters** (the whole pipeline is metric UTM; Revit converts internally to the project's display units).
4. Pick the toposolid type when prompted.

Revit creates one sub-element point per CSV row, at exact coordinates, and auto-generates the boundary sketch around the point set. The terrain lands centered on the internal origin with the site location at (0, 0, 0) — move/position it relative to the building as needed (camera-accurate, not survey-accurate, remains the standard; no shared coordinates required).

### 3A-7 — Acceptance checks

- [ ] Spot-check 2–3 known elevations: a Tenmile summit should read ≈ (true summit ASL − `Z₀`). Peak One ≈ 3,940 m ASL → expect ≈ +1,150–1,250 m relative, depending on the exact `Z₀`.
- [ ] Toposolid contour display roughly matches the QGIS contour preview (generate throwaway contours in QGIS to compare if desired — they're no longer a pipeline stage, just a QA aid).
- [ ] Revit stays responsive orbiting the 3D view. If not: coarser resolution or tighter clip, re-run from Stage 2B.
- [ ] Terrain reads correctly through the actual render camera — the only test that matters.

---

## Stage 3, Route B — D5 heightmap

*(Unchanged from Rev 2.)*

1. `terrain_normalized` → GDAL Translate to **UInt16 PNG**, scaled `-scale MIN MAX 0 65535` (8-bit causes terracing — always 16-bit).
2. D5 Terrain Tool import → calibrate height range and horizontal size against the QGIS raster's min/max and extent numbers.
3. *(Carried uncertainty flag)* Confirm against the installed D5 version whether the terrain patch enforces a square footprint and which calibration fields the importer exposes — controls have shifted across releases. If square is required, draw a square clip in Stage 2C.

---

## Key gotchas (Rev 3)

- **Reprojection is mandatory** — warp to EPSG:26913 before anything downstream, or terrain arrives stretched. *(Carried.)*
- **NEW — XY rebasing is mandatory for Route A.** Points files have no positioning dialog; Center-to-Center only exists for CAD links. Raw UTM coordinates will fail. Subtract `X₀`/`Y₀` in Refactor Fields.
- **Elevation normalization** — subtract `Z₀` in Raster Calculator or the terrain imports kilometers overhead. Negative values are fine. *(Carried.)*
- **NEW — column order matters.** CSV must read x, y, z left to right. If terrain imports mirrored or scrambled, check the field order in the Refactor/export steps.
- **NEW — point budget.** Set resolution at Stage 2B, verify count at 3A-3, stay under ~20k.
- **NEW (unverified) — header row** may need manual deletion. See 3A-5.
- **D5 terracing** = 8-bit heightmap. Export UInt16. *(Carried.)*
- **Revit slow** = coarser resolution (90–120 m) and/or tighter clip. Grasshopper decimation remains the deferred last resort — with the resolution dial now built into Stage 2B, likely never needed. *(Carried, updated.)*

---

## Appendix A — Fallback: contour → DXF route (Rev 2's Route A)

**When to use:** the terrain source is a civil engineer's contour DWG rather than a DEM, or a Revit-visible contour drawing of the terrain is itself a deliverable. **Do not use for DEM-sourced context terrain — the CSV route is strictly more accurate for that case.**

Condensed steps (full detail in the Rev 2 doc):

1. From the normalized raster: `Raster → Extraction → Contour` (10–20 m interval for distant context).
2. **Rev 3 improvement if this route is ever used:** run `Processing Toolbox → Vector geometry → Densify by Interval` on the contour layer (~10–15 m interval) before export. This adds vertices along curves so Revit has more points to sample, directly reducing the corner-cutting that motivated Rev 3.
3. Export DXF — EPSG:26913, **3D geometry preserved** (flat contours in Revit = the DXF exported 2D; re-export with Z).
4. Revit: Link CAD with **Auto - Center to Center** ("Origin to Internal Origin" fails on UTM coordinates) → `Toposolid → Create from Import → Import Instance`.
5. For engineer-supplied DWGs additionally: no closed contour loops, no contours stacked in Z, no splines/arcs (explode to polylines), all vertices carrying elevations. Better yet, ask the civil for a **points file or triangulated surface export** and jump to the CSV route.

---

## Appendix B — Close-up terrain from 1 m seamless data (contour → DXF)  ★ NEW

**When to use:** the site itself (not distant context) needs accurate near-field terrain, and the area is covered by USGS **3DEP 1-meter seamless** lidar-derived DEM. Output target here is a **contour `.dxf`**, so this runs on the Appendix A route — *not* the default points-CSV route in Stage 3A.

The shared QGIS front end is the same (load → clip → normalize → contour → export DXF → Revit). The list below is only the **deltas** versus the 30 m distant workflow. Where a step isn't mentioned, it's unchanged.

### B-1 — Check the CRS *before* reprojecting (biggest divergence)

Stage 2A assumes the DEM arrives in **EPSG:4326** (lat/long). That's true for the 1 arc-second (30 m) product but **not** for the 1 m product: 3DEP 1 m tiles are almost always delivered **already projected in UTM meters** (NAD83, local zone).

1. Drop the `.tif` into QGIS and read the layer CRS (bottom-right, or Layer Properties → Information).
2. **If it's already UTM 13N (EPSG:26913):** the Warp/Reproject step (2B) is a no-op — **skip it.** Do not reproject an already-projected raster back through 4326.
3. **If the zone or datum realization differs** (e.g. a different UTM zone, or NAD83(2011) vs. the project's 26913): run 2B once, Bilinear, target EPSG:26913.

### B-2 — Clip *tight* (1 m is ~900× denser than 30 m)

Per unit area, 1 m data carries ~900 pixels where the 30 m product had one. Close-up means a small extent anyway — keep it that way. A loose extent produces an enormous contour set and a heavy DXF. Clip to just the near-field site footprint you actually need contoured.

### B-3 — Confirm bare-earth, then plan to smooth

The 3DEP 1 m seamless is bare-earth (DTM), which is correct. But at a fine contour interval, 1 m lidar contours come out **jagged and busy** — every micro-bump and any residual vegetation/structure artifact shows. Mitigate with **one** of:

- **Smooth the raster before contouring:** a low-pass / small focal-mean / Gaussian filter (`Raster → Analysis`, or a focal statistics tool). Preferred — cleans the source once.
- **Smooth the contour lines after:** `Processing Toolbox → Vector geometry → Smooth`.

This step does not exist in the distant workflow, where 30 m data is already smooth.

### B-4 — Set the contour interval (this is the "finer contours" knob)

`Raster → Extraction → Contour` on the **normalized** raster (Stage 2D output). Set the **interval** field to your target:

- **1 m** or **0.5 m** for metric close-up work,
- **2 ft** if matching a survey standard (remember the pipeline is in meters — pick the metric equivalent or reconcile units downstream).

Appendix A's 10–20 m interval is for distant context; ignore it here.

### B-5 — Normalization: decide deliberately

Distant context subtracts `Z₀` to drop terrain onto the project origin. For a close-up **site**, decide whether you want:

- **Camera-accurate context** → keep subtracting `Z₀` as in Stage 2D, or
- **Real / survey-tied elevations** → skip or adjust the normalization and tie to your benchmark instead.

Don't blindly subtract `Z₀` if this terrain is becoming actual site context that needs to read at true datum.

### B-6 — Export DXF (same gotchas, one relaxed)

- Export **EPSG:26913, 3D geometry preserved (Z)** — flat contours in Revit = exported 2D.
- **Densify by Interval is now largely unnecessary:** at a 1 m contour interval the vertices are already dense, so Revit's corner-cutting problem is small. Skip densify unless the imported surface still looks faceted.
- Revit: Link CAD with **Auto – Center to Center** (UTM coordinates sit ~400 km from origin; "Origin to Internal Origin" fails). Unchanged from Appendix A.

### B-7 — Acceptance checks

- [ ] CRS confirmed before any reproject; no accidental 4326 round-trip.
- [ ] Contour count / DXF size is sane — if it's huge, the clip was too loose or the interval too fine.
- [ ] Contours read clean, not shredded by lidar noise — if noisy, apply B-3 smoothing and re-contour.
- [ ] Contours import into Revit with correct Z (not flat).

---

## Rejected alternatives (carried, unchanged)

- *Cesium streaming into D5:* requires Cesium token + D5 for Teams; SAI has neither. Online-only, season-locked satellite texture. **Don't revisit unless SAI acquires both.**
- *Custom DWG-generation script:* reinvents what QGIS does free.
- *Equator:* $67–169/mo; ruled out on cost.
- *Scenery mesh:* no material control, unpredictable in renders.
- *Grasshopper/Rhino.Inside decimation:* deferred; the Stage 2B resolution control likely makes it permanently unnecessary.

---

## Revision log

- **Rev 3.1 (July 9, 2026):** Added Appendix B — close-up terrain from 3DEP 1 m seamless lidar data on the contour→DXF route. Key deltas: 1 m tiles arrive already projected (check CRS, likely skip the reproject), clip tight (~900× denser than 30 m), smooth to tame lidar noise, fine contour interval, and densify no longer needed.
- **Rev 3 (July 7, 2026):** Route A rebuilt from contour DXF to points CSV (root cause: Revit vertex-sampling/triangulation inaccuracy on contour imports). XY rebasing step added (no positioning dialog for points files). Resolution control added to Stage 2B warp as the point-density dial. Point-budget table added. Contour route demoted to Appendix A fallback with densify improvement. Header-row uncertainty flagged for field verification.
- **Rev 2 (July 1, 2026):** Clip moved before contouring; elevation normalization added; Revit positioning corrected to Center-to-Center; 1 arc-second DEM default; D5 heightmap route added; Cesium rejected.
- **Rev 1 (Opus):** Original pipeline, alternatives analysis, site table, troubleshooting base.

---

## Related files

- `terrain-handoff.md` — session-level handoff (update its "immediate next steps" to point at the Rev 3 stages).
