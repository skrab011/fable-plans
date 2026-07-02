# Project Handoff — Distant Terrain Context for Renderings

**Last updated:** July 1, 2026 (Rev 2 — Fable review folded in)

---

## The problem

Generating accurate, site-specific **distant mountain context** (peaks 5+ miles away, across valleys) for client renderings. This terrain falls outside Enscape's context radius, so it must exist as real geometry — in Revit or in the renderer's own environment.

**Requirements that drove the approach:**
- Full **material control** (paint it, split into regions) — rules out raw imported meshes.
- Must **cast shadows and interact with sun**.
- **Lowest possible cost** — free pipeline strongly preferred.
- **Camera-accurate** appearance is the standard; survey-accurate georeferencing is *not* required. No shared-coordinate setup needed.
- ~10–30m resolution is plenty for distant context.

---

## The chosen pipeline (forked by renderer)

One shared QGIS front end: **USGS 3DEP DEM → reproject (EPSG:26913) → clip raster → normalize elevations to site.** Then:

- **Route A — Enscape projects (and anything needing Revit-native terrain):** contours → DXF → **Revit Toposolid**. Renderer-agnostic, visible in Revit sun studies and drawings.
- **Route B — D5 projects:** 16-bit heightmap PNG → **D5 Terrain Tool**. Real USGS relief with D5's terrain texture painting and scatter; zero Revit weight.

Cost: $0 either way. Decision rule per project: **which renderer is this deliverable going through?**

**Why this over alternatives:**
- *Rejected: Cesium streaming into D5 (D5 2.11 feature).* Would stream real terrain + satellite imagery straight to the site lat/long — but requires a Cesium token (and launched via D5 for Teams); SAI has neither. Also streams online-only (nothing stored locally) and satellite texture is season-locked. **Don't revisit unless SAI acquires D5 for Teams / a Cesium account.**
- *Rejected: custom DWG-generation script.* Reinvents what QGIS does free; doesn't address the actual hard part.
- *Rejected: Equator (paid).* Clean but $67–169/mo; ruled out on cost.
- *Rejected: scenery mesh.* No material control, unpredictable in renders.
- *Deferred: Grasshopper/Rhino.Inside decimation.* Only if Revit chokes after both mitigations (1 arc-second DEM + tight clip). With those in place, likely never needed.

---

## Two active sites

| Site | Lat / Long | View | Target context | UTM zone |
|---|---|---|---|---|
| **Summerwood**, Dillon | 39.606°N, -106.019°W | West across Lake Dillon | Tenmile Range (to ~39.47°N), Frisco | 13N (EPSG:26913) |
| **49550 US-50**, Gunnison | 38.512°N, -106.762°W | South | Ridges across Gunnison/Tomichi valley | 13N (EPSG:26913) |

**Summerwood is the learning case.** Prove the workflow here, then apply to Gunnison.

**Open item:** confirm which renderer each site's deliverables run through — this picks Route A vs. B per project.

---

## Current state

Summerwood, mid-pipeline. Rev 1 stalled at the old Stage 2D (couldn't select contours to clip — raster layer was likely active instead of the vector layer). **Rev 2 makes that step extinct:** the clip now happens on the *raster, before* contours exist (`Raster → Extraction → Clip Raster by Extent`).

**Note for resuming:** the full-tile contour layer already generated in Rev 1 should be discarded, not clipped. Clip the raster, run the new normalization step, regenerate contours. Seconds of compute; skips the snag entirely.

---

## Immediate next steps

1. **Clip the raster** (new Stage 2C) — draw the extent rectangle, lake's east shore through the Tenmile Range.
2. **Normalize elevations** (new Stage 2D) — Identify tool on the site → Raster Calculator, subtract that value. Skipping this puts the terrain ~2,800m above the building.
3. **Route A:** regenerate contours → export DXF (EPSG:26913, **3D geometry preserved**) → Revit Link CAD with **Auto - Center to Center** → Toposolid Create from Import.
   **Route B:** Translate to UInt16 PNG (`-scale MIN MAX 0 65535`) → D5 Terrain import → calibrate height range + horizontal size against the QGIS numbers.
4. **Complete Summerwood end-to-end**, then repeat for Gunnison (consider the 1 arc-second DEM there).
5. **If Revit struggles** with polygon count → 1 arc-second DEM and tighter clip first; Grasshopper decimation remains the deferred last resort.

---

## Key gotchas (carried + new)

- **Reprojection is mandatory** — warp to EPSG:26913 *before* anything downstream, or terrain arrives stretched.
- **NEW — Revit positioning:** UTM coordinates sit ~400km from the DXF origin. "Origin to Internal Origin" **fails**; use **Auto - Center to Center** and place manually.
- **NEW — elevation normalization:** the DEM carries true elevations (site ~2,700–2,800m ASL, peaks ~4,200m). Subtract the site's base elevation in Raster Calculator or the terrain imports kilometers overhead. Negative values below the site are correct and harmless.
- **Flat contours in Revit** = DXF exported 2D. Re-export with Z preserved.
- **D5 terracing** = 8-bit heightmap. Export UInt16.
- **Revit slow** = clip tighter, 20m interval, or 30m DEM.
- **D5 Terrain Tool verification** *(uncertainty flag)*: confirm against the installed D5 version whether the terrain patch enforces a square footprint and which calibration fields the importer exposes — controls have shifted across releases. If square is required, draw a square clip in Stage 2C.

---

## Revision log

- **Rev 2 (July 1, 2026):** Clip moved before contouring; elevation normalization step added; Revit positioning corrected to Center-to-Center; 1 arc-second DEM made the default recommendation; D5 heightmap route added; Cesium/D5 streaming evaluated and rejected (no Teams account or token).
- **Rev 1 (Opus):** Original pipeline, alternatives analysis, site table, troubleshooting base.

---

## Related files

- `distant-terrain-workflow.md` — the full step-by-step reference (both routes, troubleshooting table).
