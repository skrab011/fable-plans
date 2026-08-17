# D5 Terrain Heightmap — Standalone Checklist (Route B)

**A free, step-by-step method for generating a site-specific mountain backdrop for a D5 Render scene — including very distant ranges out to ~20 miles.**

This is a **standalone guide** for the **D5 route only** (Route B in `distant-terrain-workflow-rev3.md`). You do not need the Revit/Enscape docs. If you have never opened QGIS, you can still follow this.

- **Cost:** $0. USGS data and QGIS are free; you already have D5.
- **Time:** ~20–30 minutes the first time, ~15 once you've done one.
- **What you get:** the silhouette and mass of a distant range, correctly scaled, as a real USGS-relief heightmap you can paint, scatter, and light inside D5.

---

## 0. Why D5 makes the ~20-mile case *easier*, not harder

The reason very-distant terrain (>15 km / 10+ miles) needs special handling on the **Revit** route is that Revit refuses accurate geometry more than ~20 miles (~33 km) from its internal origin. The whole `very-distant-terrain-workflow.md` doc exists to work around that with a "rebase to the **midpoint** of the view" trick.

**None of that applies in D5.** A D5 heightmap is an image draped over a terrain patch — there is no origin, no distance-from-origin tolerance, no coordinate limit. So for your 20-mile view you get to **drop three things** the Revit route needs:

| Revit-route step | Why it's not needed in D5 |
|---|---|
| Midpoint XY rebasing (`X₀`/`Y₀`) | No origin limit — nothing to keep close to (0,0). |
| Elevation normalization (subtract `Z₀`) | The heightmap export remaps min→0 / max→65535 anyway, so subtracting `Z₀` first produces a byte-identical PNG. It's a genuine no-op here. |
| Point-budget / resolution thinning (90–200 m) | Heightmaps tolerate any resolution; you're limited by PNG size, not a toposolid point count. Keep native 30 m. |

Two Revit/Enscape techniques are actually **wrong** to copy into D5 — see the "Do NOT port from the Revit route" box in §7.

Net: for D5 at 20 miles you do a **single clip** covering the whole corridor and export one heightmap. That's it.

---

## 1. Tools (install once)

| Tool | What it's for | Where |
|---|---|---|
| **The National Map Downloader** | Free USGS elevation data | apps.nationalmap.gov/downloader (browser) |
| **QGIS** | Free GIS app; all the data prep | qgis.org — get the "Long Term Release" |
| **D5 Render** | Renders the terrain | You have it |

QGIS bundles GDAL and an **OSGeo4W Shell**, which is all you need for the heightmap export in §6.

---

## 2. Write down your job's numbers first

| Item | What it means | How to get it | Example |
|---|---|---|---|
| **Viewpoint** | Where the D5 camera stands | Lat/long from Google Maps (right-click → coords) | 39.606° N, −106.019° W |
| **View direction** | Which way the camera looks | The design / the PM | West across Lake Dillon |
| **Target range** | Mountains the client wants to see | Google Maps / local knowledge | Tenmile Range |
| **Distance** | Viewpoint → range, straight line | Google Maps "Measure distance" | up to ~20 mi / ~32 km |
| **UTM zone** | Metric grid for the area | Colorado mountains = **Zone 13N** | EPSG:26913 |

> **EPSG:26913** = "NAD83 / UTM Zone 13N," a coordinate system measured in **meters**. All of central Colorado uses it. Elsewhere, look up the local UTM zone.

---

## 3. Download the DEM from USGS

Source: **USGS 3DEP**, free and unrestricted.

1. Go to **apps.nationalmap.gov/downloader**.
2. Zoom so you can see **both the viewpoint AND the distant range** — for a 20-mile view, that box is large.
3. Left panel → **Elevation Products (3DEP)** → check **1 arc-second DEM** (~30 m spacing, plenty for distant context), format **GeoTIFF**.
4. Draw a box generously covering the viewpoint *through* the range (err large; you'll clip later).
5. **Search Products** → download the tile(s).

> **You may get more than one tile.** These are 1°×1° squares. A 20-mile corridor often crosses a tile edge — download **both** and merge in §4.

---

## 4. QGIS: project CRS, load, reproject

**4A — Set the project CRS to metric.** Bottom-right CRS button → type **26913** → pick **NAD83 / UTM zone 13N** → OK. Every coordinate readout is now in meters.

**4B — Load (and merge tiles).** Drag the `.tif`(s) into QGIS.
- One tile → skip to 4C.
- Two or more → **Raster → Miscellaneous → Merge** → check all tiles → Run. Use the new **"Merged"** layer from here on.

**4C — Reproject to UTM.** **Raster → Projections → Warp (Reproject)**
- **Input:** the Merged (or single) layer.
- **Target CRS:** EPSG:26913.
- **Resampling:** Bilinear.
- **Output resolution:** **leave blank** (native ~30 m). Do **not** thin to 90–200 m — that's a Revit point-budget concern; heightmaps want the detail, and a 32 km corridor at 30 m (~1,000 px deep) is a perfectly reasonable PNG.
- Run. Use the new **"Reprojected"** layer.

> **Don't skip the reproject** — export straight from lat/long and the terrain comes into D5 stretched sideways.

---

## 5. Clip the corridor (one clip — no midpoint trick)

**Raster → Extraction → Clip Raster by Extent**

1. **Input:** the Reprojected layer.
2. **Clipping extent** → dropdown → **Draw on Map Canvas** → drag a rectangle from the **viewpoint out to just past the range** (for a 20-mile view, ~32 km deep), wide enough to cover everything the camera sees left-to-right.
3. Run → a **"Clipped"** layer appears.

> **Square footprint?** Some D5 versions force the terrain patch to a **square** footprint (see the carried flag in §7). If yours does, **draw a square clip here** so the aspect ratio matches — otherwise the relief comes in stretched. If your D5 lets you set width and depth independently, any rectangle is fine.

---

## 6. Read the calibration numbers, then export the 16-bit heightmap

D5 needs two real-world measurements to scale the patch correctly. Get them from the **Clipped** layer, then export.

**6A — Vertical range (for D5's height calibration).**
Read the min and max elevation of the clip:
- **Layer Properties → Information** (scroll to statistics; click "Gather statistics" if blank), **or**
- OSGeo4W Shell: `gdalinfo -mm clipped.tif` → read `Computed Min/Max`.

Record **`MIN`** and **`MAX`** (meters). Your D5 **height range = MAX − MIN**.

**6B — Horizontal size (for D5's footprint calibration).**
From **Layer Properties → Information**, read the extent (`xmin, ymin, xmax, ymax`, in meters because CRS = 26913):
- **Width** = xmax − xmin
- **Depth** = ymax − ymin

(If you made a square clip in §5, width = depth.)

**6C — Export UInt16 PNG.** In the **OSGeo4W Shell** (Start menu → QGIS folder), `cd` to your working folder and run — substituting your real `MIN`/`MAX`:

```
gdal_translate -ot UInt16 -of PNG -scale MIN MAX 0 65535 clipped.tif heightmap.png
```

This maps the lowest elevation to 0 and the highest to 65535 across the full 16-bit range.
*(GUI equivalent: **Raster → Conversion → Translate (convert format)** → Output data type **UInt16**, output filename `heightmap.png`, and add `-scale MIN MAX 0 65535` in the "Additional command-line parameters" box.)*

> **Always 16-bit.** An 8-bit heightmap **terraces** — stair-stepped ridgelines. `UInt16` is mandatory; this is the single most common D5-terrain mistake.

---

## 7. Import into D5 and calibrate

1. **D5 → Terrain Tool → import heightmap** → select `heightmap.png`.
2. **Height range:** enter **MAX − MIN** (meters) from §6A. This makes the vertical relief real-world accurate.
3. **Horizontal size:** enter the **Width × Depth** (meters) from §6B (or the single side length if square).
4. Position the patch so the range sits where it belongs on the horizon **through your actual D5 camera** — camera-accurate is the standard for backdrop terrain; no survey georeferencing needed.

> **⚠️ Carried uncertainty — verify against your installed D5 version.** Which calibration fields the importer exposes, and whether the patch enforces a **square footprint**, have shifted across D5 releases. Confirm the field names on your first run and update this doc. If square is enforced and you didn't clip square in §5, re-clip.

> **🚫 Do NOT port these two Revit/Enscape techniques into D5:**
> - **Viewshed culling** (`r.viewshed`, Step 8A of the very-distant doc). That punches *hidden* cells to null. On the **points/toposolid** route null cells are simply skipped — but in a **heightmap** a null cell becomes 0, i.e. a hole punched down to your terrain's floor. It would gouge artificial pits into your relief. Skip it entirely on the D5 route.
> - **Elevation normalization** (subtract `Z₀`). Harmless but pointless here — the `-scale` in §6C already remaps min→0, so it produces an identical PNG. Don't bother.

---

## 8. Render check in D5

- **Lean on atmosphere, not polygons.** Real mountains 20 miles out are pale, hazy, low-contrast. Turn up D5's **fog / atmosphere / volumetric haze** so the range reads as genuinely distant — this is what sells a coarse 30 m mesh at this range. Haze does the work, not geometry.
- **Paint and scatter** the patch with D5's terrain material tools if the ridgelines are close enough to read texture; for a true 20-mile backdrop, a simple hazed rock/snow blend is usually enough.

---

## Acceptance checks

- [ ] Heightmap is **16-bit** (`UInt16`) — no visible terracing on ridgelines in D5.
- [ ] **Height range** in D5 = MAX − MIN from §6A; a known summit reads roughly its true height above the valley floor.
- [ ] **Footprint** width/depth match §6B; the range isn't horizontally stretched or squashed.
- [ ] If D5 forces square and the relief looks stretched → re-clip **square** in §5 and re-export.
- [ ] Through the **actual D5 camera**, the range reads as the right shape in the right place. *The only test that ultimately matters.*
- [ ] Distant range reads **hazy/atmospheric**, not crisp-and-fake (§8).

---

## Gotchas (quick list)

- **16-bit or it terraces.** `gdal_translate -ot UInt16`. Never 8-bit.
- **Reproject before exporting**, or the relief arrives stretched sideways.
- **Square-footprint flag** — verify on your D5 version; clip square if enforced.
- **No midpoint trick, no XY rebasing, no normalization, no point-thinning** — those are Revit-route needs; D5 has no origin limit.
- **No viewshed culling** — it gouges holes in a heightmap. (It's a points-route technique only.)
- **Get MIN/MAX and extent from the *clipped* layer**, not the full DEM, or the calibration is wrong.

---

## Related files

- `distant-terrain-workflow-rev3.md` — the full two-route (Revit + D5) pipeline this checklist is extracted from. Route B there is the source for these steps.
- `very-distant-terrain-workflow.md` — the standalone **Revit/Enscape** >15 km guide. Useful for the QGIS fundamentals and the worked-example thinking, but its midpoint-rebasing, point-budget, and viewshed steps are Revit-specific — see §0 and §7 for why they don't carry to D5.
- `terrain-handoff.md` — session-level handoff / decision record (renderer decision rule, rejected alternatives like Cesium streaming).

---

## Revision log

- **Rev 1 (Aug 17, 2026):** Extracted the D5 (Route B) path from `distant-terrain-workflow-rev3.md` into a standalone checklist and extended it for very-distant (~20 mile) views. Key point: D5 has no origin limit, so the Revit route's midpoint-rebasing, elevation normalization, and point-budget thinning are all dropped; a single deep clip suffices. Flagged viewshed culling as unsafe on the heightmap route (nulls become pits). Carried the 16-bit-mandatory and square-footprint uncertainty flags from Rev 3.
