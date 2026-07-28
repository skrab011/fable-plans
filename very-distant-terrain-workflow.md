# Very-Distant Terrain (>15 km / ~10+ miles) — Revit / Enscape Workflow

**A free, step-by-step method for adding far-off mountain backdrops to a client rendering — without going on site, shooting photos, and photomontaging them in afterward.**

This is a **standalone guide**. You do not need to have read any of the other terrain docs to follow it. Every step is spelled out from the beginning. If you have never opened QGIS before, you can still do this.

- **Cost:** $0. (USGS data and QGIS are free; you already have Revit and Enscape.)
- **Time:** ~30–45 minutes the first time, ~20 minutes once you've done one.
- **What you get:** the *silhouette and mass* of a distant mountain range, correctly placed, seen through your render camera. Not survey-accurate, not photo-detailed — the **gist**, which is all that reads at this distance anyway.

> **Worked example used throughout:** a view *from Breckenridge Ski Resort* toward a mountain range roughly **18 miles (~29 km)** away. Wherever you see Breckenridge numbers, swap in your own job's numbers.

---

## 1. Why very-distant terrain needs its own procedure

If you already know the normal distant-terrain workflow, here is the **one thing that is different** and why it matters. If you don't know the normal workflow, just read this as the reason for Step 8.

Revit refuses to build geometry more than about **20 miles (~33 km)** from its internal origin (the model's 0,0,0 point), and its accuracy gets shaky well *before* that limit — you'll see "elements are too far from the origin" warnings, jittery meshes, and sluggish orbiting.

The normal workflow slides the terrain so that **your site** sits at the origin. That's fine when the mountains are ~10 km away. But at **18 miles (~29 km)**, the far peaks would land ~29 km from the origin — right at the edge of Revit's tolerance. It often imports badly.

**The fix (Step 8):** instead of putting the *site* at the origin, we put the **midpoint of the view** at the origin — roughly halfway between the camera and the mountains. Then:

- the camera end sits ~15 km on one side of the origin,
- the far peaks sit ~15 km on the other side,
- **nothing is more than ~15 km from the origin** instead of 29 km.

Fifteen km is comfortably inside Revit's working range. This single trick is what makes an 18-mile backdrop import cleanly. Everything else in this guide is standard terrain prep.

---

## 2. Tools you need (install once)

| Tool | What it's for | Where |
|---|---|---|
| **The National Map Downloader** | Free USGS elevation data | apps.nationalmap.gov/downloader (runs in a browser) |
| **QGIS** | Free GIS app; does all the data prep | qgis.org (download the "Long Term Release") |
| **Revit 2027** | Builds the terrain (toposolid) | You have it |
| **Enscape** | Renders it | You have it |
| **Notepad** (or any plain-text editor) | One possible fix at the end | Already on your PC |

---

## 3. Before you start — write down your job's numbers

Fill this in for your view. You'll read most of it off Google Maps and (later) off QGIS. The Breckenridge column shows the worked example.

| Item | What it means | How to get it | Breckenridge example |
|---|---|---|---|
| **Viewpoint** | Where the render camera stands | Lat/long from Google Maps (right-click the spot → the coordinates are at the top of the menu) | ~39.48° N, −106.07° W |
| **View direction** | Which way the camera looks | Look at the design / talk to the PM | *(fill in — e.g. "east toward the Front Range")* |
| **Target range** | The mountains the client wants to see | Google Maps / local knowledge | *(fill in the range name)* |
| **Distance** | Viewpoint → range, straight line | Google Maps "Measure distance" tool | ~18 mi / ~29 km |
| **UTM zone** | The metric map grid for the area | Colorado mountains = **Zone 13N** | Zone 13N = **EPSG:26913** |

> **What is EPSG:26913?** It's just a code name for "NAD83 UTM Zone 13N," a coordinate system that measures position in **meters** (eastings/northings) instead of degrees. All of central Colorado uses Zone 13N. If your project is somewhere else, look up its UTM zone, but for anything around Breckenridge/Summit County it's 26913.

### Decide this first: one view direction, or several?

The midpoint trick in Step 10 centers Revit's origin on the middle of **one** view corridor. If the job looks in **more than one direction** — e.g. east toward the Divide *and* southwest toward a different peak — you **cannot** center a single origin on both at once; whichever range you don't optimize for ends up far from the origin again.

**Do this instead: build one separate backdrop toposolid per view direction.** No single camera sees two directions at once, so the backdrops never need to sit correctly *relative to each other*. For each direction:

- run Steps 4–15 as its own mini-job (its own clip corridor, its own midpoint `X₀`/`Y₀`, its own CSV, its own toposolid);
- position each toposolid for the camera that looks that way;
- in each render view, **hide the backdrop(s) you're not using** (Visibility/Graphics, or select → Hide in View), so the two don't clutter each other.

This keeps every patch close to the origin and cleanly framed. Only the *far* directions need the midpoint trick — a nearer view (under ~12–15 km) can use the simpler site-centered rebase (put `X₀`/`Y₀` on the viewpoint itself).

> See the **Worked example** at the end for how this plays out on the Breckenridge east + southwest views.

---

## 4. Download the elevation data (DEM) from USGS

A **DEM** ("digital elevation model") is a grid where every cell holds a ground elevation — think of it as a giant spreadsheet of heights. USGS gives it away free.

1. Go to **apps.nationalmap.gov/downloader**.
2. Zoom the map to your area. **Zoom out enough that you can see both your viewpoint AND the distant range** — for Breckenridge, the resort *and* the target range 18 miles away must both be on screen.
3. On the left panel, open **Elevation Products (3DEP)**.
4. Check **1 arc-second DEM** (this is ~30 m ground spacing — plenty for distant mountains) and set format to **GeoTIFF**.
5. Draw a box (the "Define Area" tools) that generously covers the viewpoint *through* the range. Err on the side of too big; you'll trim it later.
6. Click **Search Products**, then download the tile or tiles that cover your box.

> **You may get more than one file.** These 1 arc-second tiles are 1°×1° squares. If your viewpoint and your range fall in different tiles, download **both** — Step 6 shows how to stitch them together.

---

## 5. Set up your QGIS project

1. Open **QGIS**.
2. Set the project's coordinate system to metric UTM so all the coordinate readouts are in meters:
   - Bottom-**right** of the window, click the little CRS button (it may say "EPSG:4326").
   - In the box, type **26913**, pick **NAD83 / UTM zone 13N**, click **OK**.

That's it — the project now "thinks" in meters, which every later step relies on.

---

## 6. Load the DEM (and stitch tiles if you have more than one)

1. **Drag the downloaded `.tif` file(s)** from Windows Explorer straight into the QGIS map area. Each appears as a grey elevation image.
2. **If you downloaded only one tile, skip to Step 7.**
3. **If you have two or more tiles, merge them into one:**
   - Top menu: **Raster → Miscellaneous → Merge**.
   - **Input layers:** click the "…" and check all your DEM tiles.
   - Leave the rest at defaults, click **Run**, then **Close**.
   - A new layer called "Merged" appears. Use *that* from now on; you can uncheck the individual tiles to hide them.

---

## 7. Reproject the DEM **and set the density** (one dialog, two jobs)

The raw DEM is in lat/long degrees. We need it in meters, and at the same time we'll thin it out so we don't drown Revit in points. Both happen in one tool.

1. Top menu: **Raster → Projections → Warp (Reproject)**.
2. **Input layer:** your DEM (the "Merged" layer if you stitched, otherwise the single tile).
3. **Source CRS:** leave as-is (it auto-detects).
4. **Target CRS:** choose **EPSG:26913** (NAD83 / UTM zone 13N).
5. **Resampling method:** **Bilinear**.
6. **Output file resolution in target georeferenced units:** type **`150`**.
   - This is the **density dial.** It sets the spacing, in meters, between elevation points. `150` means one point every 150 m.
   - **Why 150 for an 18-mile view?** At this distance you're capturing a silhouette, not boulders. 150 m keeps the point count low enough for Revit while still reading perfectly. (Closer ranges use a finer number like 90–100; very far ones like this use 150 or even 200.)
7. Click **Run**, then **Close**. You get a new "Reprojected" layer. Use that going forward.

> **Don't skip the reproject.** If you contour or export without it, the terrain comes into Revit stretched sideways.

---

## 8. Clip to just the view corridor (this controls your point budget)

You do **not** need a giant square of terrain — only the strip your camera actually sees, from the viewpoint out to just past the range. A tight clip is what keeps the whole thing light.

1. Top menu: **Raster → Extraction → Clip Raster by Extent**.
2. **Input layer:** the "Reprojected" layer.
3. Next to **Clipping extent**, click the dropdown → **Draw on Map Canvas**, then drag a rectangle over the map:
   - Start at your **viewpoint**, extend it **toward and a little past the range**.
   - Make it **wide enough** to cover everything the camera will see left-to-right, but no wider. A corridor **~15 km wide** is generous for a normal camera at this distance.
4. Click **Run**, then **Close**. A "Clipped" layer appears.

> **Rule of thumb for the corridor size:** a strip about **15 km wide × 20 km deep** at 150 m spacing works out to roughly **13,000 points** — right in the sweet spot (see Step 11). If your strip needs to be much bigger, bump the Step 7 resolution to `200` and re-run from Step 7.

---

## 8A. ⭐ Cull the hidden terrain with a viewshed (optional, big payoff)

*New in this revision. Skip it and the guide still works — everything downstream is unchanged. But for a mountain view it is the single biggest point-count saver, and it composes cleanly with the midpoint trick.*

**The idea:** most of the DEM you just clipped can't actually be *seen* from the viewpoint — it's the back-slopes of ridges, valley floors tucked behind nearer high ground, everything the front ranges occlude. A **viewshed** computes exactly which cells are visible from your camera and lets you throw the rest away *before* they ever become toposolid points. In mountainous terrain that's commonly **half or more** of the cells gone.

Two ways to spend the savings: keep the lighter point count as-is, or go back to Step 7 and drop the resolution to `90–120` so the *visible* ridgelines come in crisper for the same budget. Either is a win.

> **This does not replace the midpoint trick.** The visible far peaks are still ~28 km out. Viewshed cuts the *number* of points; the Step 10/12 midpoint rebasing keeps their *coordinates* inside Revit's origin limit. You still do both.

### Get the observer's coordinates

The viewshed observer is the **camera / viewpoint** — *not* the strip midpoint you'll read in Step 10. Hover the mouse over the viewpoint (the Vista Haus building) and read the two Coordinate numbers at the bottom of the window (they're in meters because of Step 5). Note them as `Xobs`, `Yobs`.

### Run the viewshed (GRASS `r.viewshed` — already bundled with QGIS)

1. Open the **Processing Toolbox**, search **`r.viewshed`**, double-click it.
2. **Input elevation raster:** the **`Clipped`** layer from Step 8 (use the *true-elevation* clip, before Step 9's normalize — line-of-sight math wants real heights, and normalizing first would work too but keep it simple).
3. **Coordinates of the viewing position:** type `Xobs,Yobs` (comma-separated), or click the `…` and pick the point on the map at the viewpoint.
4. **Observer elevation (height above ground):** set this to the **actual render-camera height in your Revit model**, not the 1.75 m default. If the camera sits on an upper floor, a deck, or behind tall second-storey glass, it sees over ridges a ground-level observer doesn't — get this right or you'll cull terrain that's genuinely in shot. For a ground-level exterior eye, ~1.7 is fine.
5. **Target elevation:** `0` (or a small positive value like `2` to be slightly generous at ridge crests).
6. **Maximum visibility distance:** set to just past your farthest peak — e.g. `30000` (meters) for the Breckenridge view. (`-1` = unlimited, but capping it is faster.)
7. **Check "Consider the earth curvature and refraction"** (the `-c` flag). **This is not optional at 28 km** — earth curvature drops the horizon by ~55–60 m at that range, so a flat-earth viewshed would wrongly mark low distant terrain as visible. Leave the refraction coefficient at its `0.14286` default.
8. **Check "Output format is invisible = 0, visible = 1"** (the `-b` boolean flag) — it makes the next step trivial.
9. **Run.** You get a `Viewshed` raster: 1 where visible, 0 where hidden.

### Turn the mask into a holey DEM

Now keep elevations only where visible, and set everything else to "no data" so it produces no points later.

1. Processing Toolbox → search **`r.mapcalc.simple`**, double-click.
2. Map **A** = `Viewshed`, **B** = `Clipped`.
3. **Expression:** `if(A == 1, B, null())`
4. **Output** → name it `terrain_visible`, **Run**.

`terrain_visible` is your clipped DEM with all the hidden cells punched out to null. **Use it in place of `Clipped` from here on** — feed it into Step 9's Raster Calculator, and Step 11's "Raster pixels to points" will skip the null cells automatically.

> **Optional — protect marginal ridgelines.** A summit sitting a hair below the geometric horizon can still show its tip through haze. To avoid a hard cull clipping it: before the mapcalc step, run GRASS **`r.grow`** on the `Viewshed` raster with `radius = 2` to fatten the visible mask by a cell or two, and use *that* grown mask as `A`. Not required — just insurance.

> ⚠️ **Uncertainty flag — Revit bridges the gaps (verify on the first run).** Revit's Points-File import auto-triangulates the point set and will **not** leave holes where you culled — it spans each hidden gap with a large flat triangle. By construction those bridges sit over terrain that *isn't visible from this camera*, so they normally hide behind the front ridge. **But it isn't guaranteed:** a bridge anchored on two high visible points across a low hidden valley can occasionally ride up into frame. On your first viewshed run, check the render (Step 16) for any flat web on the skyline. If one intrudes, the fix is to cull *conservatively* — use the viewshed to remove only the **large contiguous hidden blocks** (the whole back of a range, deep valleys), not fine speckle inside the visible zone, since scattered single-cell holes are what Revit bridges most visibly. Resolve this on the Vista Haus run and note the outcome here.

---

## 9. Normalize the elevations (drop the terrain down to your building's level)

The DEM stores *true* elevations above sea level — around 2,900 m at Breckenridge, ~4,000 m at the peaks. If you imported that as-is, the terrain would float kilometers above your model. So we subtract the viewpoint's elevation to bring the ground to ≈ 0.

1. Click the **Identify Features** tool (the blue ⓘ arrow in the toolbar), then click your **viewpoint** location on the map. A panel shows the elevation value there. **Write it down** — call it **`Z₀`** (for Breckenridge, ~2,900).
2. Top menu: **Raster → Raster Calculator**.
3. In the expression box, build: `"Clipped@1" - 2900` (use *your* `Z₀`, and your clipped layer's name — double-click it in the list on the left so it's spelled exactly right). **If you did Step 8A, use `"terrain_visible@1"` here instead of `"Clipped@1"`** so the culling carries through.
4. Set an output filename like `terrain_normalized`, click **OK**.

Now the viewpoint sits at elevation ≈ 0 and the peaks read as their height *above the viewpoint*. Negative numbers (ground lower than the viewpoint) are normal and fine.

---

## 10. ⭐ Read the **midpoint** coordinates (the key step for distant terrain)

This is the step that makes an 18-mile view work in Revit. We're going to find the coordinates of the **middle of the clipped strip**, and in Step 12 we'll shift everything so that midpoint becomes Revit's origin.

1. Make sure the bottom status bar reads coordinates in meters (it will, because you set the project CRS to 26913 in Step 5 — the numbers will be a 6-digit and a 7-digit figure, not small decimals).
2. **Hover your mouse over the visual center of the clipped terrain strip** — roughly halfway between the viewpoint and the range.
3. Read the two numbers in the **Coordinate** box at the bottom of the window and write them down, rounded to whole numbers:
   - **`X₀`** = the first (easting) number — 6 digits, starts around 4xx,xxx
   - **`Y₀`** = the second (northing) number — 7 digits, starts around 4,3xx,xxx

> **Why the midpoint and not the site?** Revit gets inaccurate past ~15–20 km from its origin. If we centered on the *viewpoint*, the far peaks would be ~29 km out — too far. Centering on the *midpoint* means the viewpoint and the peaks are each only ~15 km from the origin. Same terrain, half the maximum distance, clean import. **This is the whole trick.**

> **Log `X₀`, `Y₀`, and `Z₀` in your job notes.** If anyone ever re-runs this view, these three numbers reproduce it exactly.

---

## 11. Turn the elevation grid into points

Revit's terrain wants a list of individual XYZ points. This converts every grid cell into one point.

1. Open the **Processing Toolbox** (top menu: **Processing → Toolbox**, or the gear icon).
2. In its search box, type **`Raster pixels to points`** and double-click the result.
3. **Raster layer:** `terrain_normalized` (from Step 9).
4. **Field name:** leave as `VALUE` (this will hold the elevation).
5. Click **Run**, then **Close**. A dotted point layer appears.

**Now check the point budget** before going further:

- Right-click the new point layer → **Show Feature Count**. The number next to the layer name is your point total.
- **Target: 8,000–15,000 points. Hard ceiling: ~20,000.**

| If the count is… | Do this |
|---|---|
| Under ~15,000 | You're good — continue. |
| 15,000–20,000 | Acceptable, but Revit may feel heavy. Fine to continue. |
| Over ~20,000 | Go back to **Step 7**, set resolution to `200` (or clip tighter in Step 8), and re-run. Don't fight a bloated import in Revit. |

> **If you did the viewshed (Step 8A)** the count will be well under budget — often less than half. You can leave it light, or reinvest: go back to **Step 7**, set resolution to `90–120`, and re-run through 8A. Same point budget, crisper visible ridgelines.

---

## 12. Rebase X and Y so the midpoint becomes the origin

Here we subtract the midpoint coordinates from every point, and lay out the three columns (X, Y, Z) that Revit reads.

1. In the **Processing Toolbox**, search **`Refactor fields`** and double-click it.
2. **Input layer:** the point layer from Step 11.
3. In the fields table, **delete every existing row** (select each and click the red "–" / minus button) so you can build a clean set.
4. Click the green "+" to add rows, and create **exactly these three, in this exact order** (order matters — Revit reads the columns left-to-right as X, then Y, then Z):

| Field name | Type | Expression (type this in) | What it does |
|---|---|---|---|
| `x` | Decimal (double) | `$x - 4XXXXX` | East-west position, measured from the midpoint. Replace `4XXXXX` with **your `X₀`** from Step 10. |
| `y` | Decimal (double) | `$y - 43XXXXX` | North-south position, measured from the midpoint. Replace with **your `Y₀`**. |
| `z` | Decimal (double) | `"VALUE"` | Elevation — already normalized in Step 9, passes straight through. |

> **`$x` and `$y`** are QGIS shorthand for "this point's own easting/northing." You type the expression once; QGIS runs it across all points automatically. You never type coordinates point-by-point.

5. Click **Run**, then **Close**. A new layer (usually called "Refactored") appears.

---

## 13. Export the CSV file

1. Right-click the **Refactored** layer → **Export → Save Features As…**
2. **Format:** **Comma Separated Value (CSV)**.
3. **File name:** somewhere easy to find, e.g. `breckenridge_distant_terrain.csv`.
4. **Geometry:** set to **No geometry**. (Your x/y/z columns already carry the position; leaving geometry on would add duplicate coordinate columns and confuse Revit.)
5. Under **Select fields to export**, confirm only **`x`, `y`, `z`** are checked, **in that order**.
6. Click **OK**.

You now have the file Revit will import.

---

## 14. Test-import a small piece first (10 minutes that saves an hour)

Because 18 miles is near Revit's comfort edge, prove it works small before committing.

1. Quickest version: in QGIS, temporarily clip a **short section of just the target ridgeline** (repeat Steps 8–13 on that small piece) and export a second, tiny CSV.
2. Import that small CSV into Revit (procedure in Step 15).
3. **Watch for:** "elements too far from origin" warnings, a jagged/jittery mesh, or Revit choking. If it comes in **clean and orbits smoothly**, your `X₀/Y₀` midpoint is good — proceed to the full import. If it complains, see Troubleshooting.

> Skip this only if you've already run one of these views successfully and trust your numbers.

---

## 15. Import the terrain into Revit

1. In Revit, go to the **Massing & Site** tab.
2. Click the **Toposolid** dropdown → **Create from Import → Points File**.
3. Browse to and select your CSV (`breckenridge_distant_terrain.csv`).
4. **Units:** choose **Meters** (the whole pipeline is in metric meters; Revit converts to your project's display units automatically).
5. Pick a toposolid **type** when prompted (any surface type is fine for backdrop terrain).
6. Revit places one point per CSV row at its exact coordinates and auto-draws the boundary. The terrain lands **centered on the model origin** (because you rebased to the midpoint), with the viewpoint on one side and the mountains on the other.

**Positioning:** because you centered on the midpoint (not the site), the terrain won't be sitting exactly under your building — it'll be offset by ~15 km's worth in model space. That's expected. **Move the whole toposolid** so that, *looking through your render camera*, the mountains sit where they belong on the horizon. This is eyeball/camera-accurate, which is the standard for backdrop terrain — you are not tying it to survey coordinates.

**Acceptance checks:**

- [ ] Terrain imported with no (or only dismissible) origin warnings.
- [ ] The 3D view orbits without lag.
- [ ] Spot-check a summit: a peak's Z value should read roughly *(true peak elevation − `Z₀`)*. E.g. a 3,900 m peak with `Z₀` = 2,900 should sit ~+1,000 m above the viewpoint.
- [ ] Through the **actual render camera**, the range reads as the right shape in the right place. *This is the only test that ultimately matters.*
- [ ] **If you did the viewshed (Step 8A):** no flat bridging triangle rises onto the skyline where hidden terrain was culled. If one does, see the Step 8A uncertainty flag (cull large blocks only, not speckle).

---

## 16. Render check in Enscape

1. Open your Enscape view. The toposolid appears automatically — no export step.
2. Nudge the terrain's position (Step 15) if the peaks sit too high/low or off to one side in frame.
3. **Lean on atmosphere, not geometry.** Real mountains 18 miles out are pale, hazy, low-contrast. Turn up Enscape's **fog / atmosphere / haze** so the range reads as *distant*. This is what makes a coarse 150 m mesh look completely convincing — at this range, haze does the work, not polygons.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Revit warns **"elements too far from origin"** or the far mesh looks jittery | Origin is still too far from the terrain | Re-check Step 10 — make sure `X₀/Y₀` is the **midpoint** of the strip, not the viewpoint. Re-run Steps 12–13. |
| Import **errors on the first line** of the CSV | Revit's parser may not like the header row (`x,y,z`) QGIS writes | Open the CSV in **Notepad**, delete the top line, save, re-import. *(Whether Revit 2027 needs this varies — try with the header first; only strip it if it fails.)* |
| Terrain comes in **mirrored or scrambled** | Column order wrong | Confirm the fields are `x, y, z` in that exact order in Steps 12 and 13. |
| Terrain floats **kilometers above** the model | Elevations not normalized | You skipped or mis-entered Step 9. Re-do the Raster Calculator subtraction with the correct `Z₀`. |
| Terrain arrives **stretched sideways** | Reproject skipped | Re-run Step 7 (Warp to EPSG:26913). |
| Revit is **slow / heavy** | Too many points | Coarsen resolution to `200` in Step 7 or clip tighter in Step 8; stay under ~20,000 points. |
| Mountains look **too crisp / fake** in the render | Not a geometry problem | Add Enscape haze/fog (Step 16). Distance should soften them. |
| **Flat web / spanning triangle** across the skyline | Revit bridged a culled hidden gap (Step 8A) | Expected over hidden zones; only a problem if it's *visible*. Re-run 8A culling large contiguous blocks only, not fine speckle. See the Step 8A flag. |
| Viewshed keeps **too much** distant low terrain | Earth curvature not applied | Re-run `r.viewshed` (Step 8A) with the `-c` curvature flag checked. |
| Viewshed **clips a summit** that should show | Observer height too low, or hard mask edge | Set the observer elevation to the real camera height (Step 8A.4); optionally `r.grow` the mask (Step 8A optional note). |

---

## Quick reference card

1. Note viewpoint, direction, range, distance, UTM zone.
2. Download 1 arc-second DEM covering viewpoint **and** range.
3. QGIS project CRS → **EPSG:26913**.
4. Load tile(s); **Merge** if more than one.
5. **Warp/Reproject** → EPSG:26913, Bilinear, resolution **150**.
6. **Clip** a ~15 km-wide corridor toward the range.
7. *(Optional, big win)* **Viewshed cull** — `r.viewshed` from the viewpoint (curvature `-c` on, boolean `-b`, real camera height, max dist past the peaks) → `r.mapcalc.simple` `if(A==1,B,null())` → `terrain_visible`. Use it downstream.
8. **Identify** viewpoint elevation `Z₀`; **Raster Calculator** subtract it (input = `terrain_visible` if you did the viewshed).
9. Hover the **strip midpoint**; note `X₀`, `Y₀`.
10. **Raster pixels to points**; check count (target 8–15k, max 20k).
11. **Refactor fields** → `x = $x - X₀`, `y = $y - Y₀`, `z = "VALUE"`.
12. **Export CSV** (No geometry; fields x,y,z).
13. **Test-import** a small band in Revit.
14. **Toposolid → Create from Import → Points File**, units **Meters**.
15. Position by camera; add **Enscape haze**.

**The one non-obvious rule:** for anything past ~15 km, rebase to the **midpoint of the view**, not the site — otherwise Revit chokes on the distance from origin.

---

## Appendix: Worked example — Breckenridge, Vista Haus on Peak 8

**Building / viewpoint:** the **Vista Haus** restaurant near the top of Peak 8, Breckenridge — ≈ 39.478° N, −106.078° W, elevation ≈ **11,300 ft (~3,440 m)**. UTM Zone 13N / EPSG:26913. Read the exact `X₀`/`Y₀`/`Z₀` off QGIS at the building; the numbers here are just for planning.

**Both target views point east/southeast**, within roughly a 30° fan, so they **share a single eastward backdrop patch** — you do *not* need two. (The "one patch per direction" rule in Section 3: here both views *are* the same direction, so they collapse into one.)

- **Foreground — Mount Baldy** (Bald Mountain, 13,684 ft): ≈ **5 miles (~8 km)** east/ESE of Vista Haus. Close enough to import fine on its own, but it sits on the way to the Divide, so one clip captures it too.
- **Backdrop — Grays & Torreys Peaks** (14,270 / 14,267 ft): ≈ **17.5 miles (~28 km)** northeast (~52° bearing).

**How to run it:**

1. **Clip (Step 8):** draw one wide **fan east–northeast** from Vista Haus, covering both targets (bearings ~50°–85°) out to ~30 km, just past Grays/Torreys.
2. **Resolution (Step 7):** use **200 m** here — the combined fan is large, and 200 m keeps it near ~11,000 points. (150 m would push it over the 20k ceiling.)
3. **Rebase (Step 10):** center `X₀`/`Y₀` on the **center of that clipped fan** (~12 km ENE of Vista Haus), not on the building. That puts both Vista Haus and Grays/Torreys ~15 km from the origin — inside Revit's comfort zone.
4. **DEM tiles (Step 4):** download **both** and Merge (Step 6) — the fan crosses the −106° tile line:
   - `n40w107` — Vista Haus,
   - `n40w106` — Baldy **and** Grays/Torreys.

**Expectation check:** from 11,300 ft, Grays/Torreys sit only ~900 m above the camera and Baldy ~725 m — a substantial but *not* towering eastern skyline, which is correct for this vantage. Lean on Enscape haze (Step 16) to push the distant Divide back and let the nearer Baldy carry a little more contrast.

> **Optional — if Baldy is a hero element.** If the client wants Mount Baldy rendered with more crispness (it's the closest peak), give it its **own separate patch** at a finer ~90–100 m resolution, **site-centered on Vista Haus** (it's under ~12 km, so no midpoint trick needed), and keep the coarse 200 m fan just for the distant Divide behind it. Otherwise, one combined patch is simplest.

### Verify these before committing (the numbers above are planning estimates)

Every figure in this example was estimated off a map. Confirm each one in QGIS / on the actual view before you run the full clip — takes five minutes and saves a re-do.

- [ ] **Vista Haus `X₀`/`Y₀`/`Z₀`.** Hover the building in QGIS (project CRS = EPSG:26913) and read the real easting, northing, and (via Identify → Step 9) elevation. Expect `Z₀` ≈ 3,440 m; if it's far off, re-check you clicked the restaurant, not the base.
- [ ] **Which peaks are actually in frame.** Confirm with the PM/design that Grays & Torreys and Baldy are the intended subjects — from Peak 8 the eastern skyline also includes other Divide summits (Bald's neighbors, the Continental Divide ridge). Widen or narrow the clip fan to match what the camera really shows.
- [ ] **Baldy's identity and distance.** Confirm the "Mount Baldy" in view is Bald Mountain (≈ 39.489° N, −105.987° W, 13,684 ft) and re-measure Vista Haus → summit. If it comes out over ~12–15 km, it needs the midpoint treatment too, not the simpler site-centered rebase.
- [ ] **Clip-fan center for the rebase.** After clipping (Step 8), hover the actual visual center of *your* clipped fan and read `X₀`/`Y₀` there — don't reuse the "~12 km ENE" estimate.
- [ ] **Point count.** After Raster pixels to points (Step 11), check the feature count is under ~20k. If 200 m still runs heavy, go to 250 m or tighten the fan.
- [ ] **Max distance from origin.** After rebasing, sanity-check that the farthest point (a Grays/Torreys summit) is within ~15–16 km of (0,0). If a test import throws origin warnings, your rebase center is off.
- [ ] **DEM coverage.** Make sure the merged `n40w107` + `n40w106` tiles actually span the whole fan with no gap at the −106° seam before you clip.
- [ ] **Viewshed sanity (if using Step 8A).** After `terrain_visible`, eyeball the mask: the visible cells should form the ridgelines and near-facing slopes you'd expect to see, with the back-sides and hidden valleys punched out. If almost everything is still "visible," the curvature flag or observer height is likely wrong.

---

## Revision log

- **Rev 1.1 (July 28, 2026):** Added Step 8A — optional viewshed culling (`r.viewshed` + `r.mapcalc.simple`) to drop terrain that isn't visible from the camera, cutting point count (often by half or more) and freeing budget for finer resolution. Wired it into Steps 9/11, the quick-reference card, troubleshooting, and acceptance checks. Carries an unverified flag: Revit bridges culled gaps with flat triangles — normally hidden, but verify on the first Vista Haus run.
- **Rev 1 (July 2026):** Original standalone very-distant (>15 km) guide with the midpoint-rebasing trick, multi-view handling, and the Breckenridge / Vista Haus worked example.
