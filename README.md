# 🗺️ 3D Geo-Density Topographic Surface (Google Timeline)

An artistic, interactive 3D visualization that transforms Google Timeline location history into an elevated topographic relief map — rendered as a self-contained, interactive HTML page with **Plotly**.

Instead of relying on static bars or traditional 2D heatmaps, this project converts visit frequency into **fluid three-dimensional hills and mountains** rising out of a realistic, textured ocean/plains relief map — irregular coastlines and hill outlines, coastal-depth shading, mottled plains — highlighting the cities and regions where you spent the most time and traveled the most.

---

## 🎯 Project Goal

* **Data transformation:** Process semantic JSON files exported directly from the Google Maps app.
* **Density mapping:** Compute a 2D Kernel Density Estimation (**KDE**) over every visit location, compressed through a two-stage gamma curve so smaller/less-visited cities remain visible next to the dominant home peak, and perturbed with procedural noise so hills read as natural, irregular terrain rather than smooth Gaussian domes.
* **3D representation:** Render an interactive **Plotly** 3D surface — ocean shaded by depth, unvisited land as mottled green plains, visited areas rising into hills/mountains colored by elevation — with country borders draped over the relief and city markers colored by how recently each place was visited, viewable in any browser.
* **UI & accessibility:** A title, two color legends, camera-preset buttons, a loading indicator, and a searchable HTML table beneath the map round out the experience — with both color scales checked computationally against simulated color blindness.

---

## 🛠️ Requirements & Installation

### Python Environment

Make sure you have Python 3.10+ and the following libraries installed:

```bash
pip install pandas numpy scipy geopandas shapely reverse_geocoder plotly
```

---

## 📌 Step-by-Step Workflow

### Step 1: Get Your Data (Google Timeline, On-Device)

> **Important note:** Due to Google's recent privacy changes, location history data is no longer stored centrally in the cloud and can no longer be fully downloaded via Google Takeout. This data now lives locally on your mobile device.

1. Open the **Google Maps** app on your mobile device.
2. Tap your profile photo $\rightarrow$ **Your Timeline**.
3. Tap the **three-dot menu (⋮)** in the top-right corner and select **Settings and privacy**.
4. Find the **Export Timeline data** section.
5. Send the generated `.json` file to your computer (via email or a cloud storage service).

---

### Step 2: Processing & 3D Map Generation (Python)

Everything runs through a single Jupyter Notebook ([Create_Interactive_3D_Map.ipynb](Create_Interactive_3D_Map.ipynb)), structured in three main phases:

1. **Extraction & cleaning**
   * Reads the JSON file, filtering the visit structure (`visit -> topCandidate -> placeLocation`).
   * Parses the `"geo:LATITUDE,LONGITUDE"` geographic format into float coordinates, restricted to the Americas bounding box (Patagonia to Alaska, Pacific coast to mid-Atlantic).

2. **City identification**
   * Reverse geocodes every visit offline via `reverse_geocoder`, clusters nearby visits into metro areas (50 km radius), and labels each cluster with the largest known city nearby (using a GeoNames major-cities lookup) rather than the closest small locality.
   * Keeps only the most recent visit per city and exports the result to `visited_cities.csv`, used to place markers on the 3D map.

3. **Interactive 3D relief map**

   Builds a 600×~580 evaluation grid over the Americas bounding box and turns it into a shaded, textured terrain in several passes:

   * **Density estimation.** `scipy.stats.gaussian_kde` fits a 2D Gaussian KDE over *every* recorded visit point (not just the 54 principal cities — that clustering is only used downstream, to decide which points get a marker and a label). The bandwidth is widened to `0.65×` the KDE's default (Scott's rule) factor, since a tight bandwidth produces thin spikes rather than rounded, mountain-like hills.
   * **Dynamic range compression.** Visit density is heavily skewed toward home/frequent locations, so a linear height scale would leave smaller cities almost invisible next to the dominant peak. The density is passed through two chained power-law (gamma) transforms (`density ** 0.4`, then `** 0.2` after renormalizing) — both map `0 → 0` and `1 → 1`, so the true background is untouched while mid-and-low density areas get pulled up disproportionately.
   * **Irregular boundaries.** A smooth (low-frequency, Gaussian-blurred) random noise field multiplies the density *before* the hill/no-hill threshold is evaluated, so the noise perturbs where each hill's edge actually falls — producing an irregular, natural-looking outline instead of a perfect Gaussian oval. The same perturbed field then feeds the height calculation, so slopes get a bit of organic texture too.
   * **Smooth tapering.** A `smoothstep` function (`3t² − 2t³`, zero slope at both ends) gates the transition from flat ground to hill, so terrain rises gently out of the plain instead of ending in a hard cliff.
   * **Land/ocean masking.** Every grid cell is classified as land or ocean via a point-in-polygon test (`shapely.contains_xy`) against 50 m-resolution Natural Earth country boundaries — coarser 110 m data smooths away bays/inlets (e.g. Tampa Bay) entirely, letting hills bleed out over real open water. The boolean land mask is Gaussian-blurred and multiplied into the height field itself (not just the color), so hills can't rise over the ocean or spill past the true coastline, without introducing another hard edge.
   * **Hypsometric coloring.** A custom Plotly colorscale mimics a real relief map: deep ocean → shallow coastal water → mottled plains → hills → mountains → snow-capped peaks. Ocean cells are always painted from the water end of the ramp — coastal water is lightened using a Euclidean distance transform from the coastline (`scipy.ndimage.distance_transform_edt`), so it fades from light (shallow) to dark (deep/far from land) with distance, plus a touch of smooth noise for texture. Unvisited land gets the same subtle mottling treatment (color only — it never touches the actual height, which still reflects visit density alone).
   * **Draped country borders + frame.** Natural Earth boundary lines are rendered as a `Scatter3d` line trace, with each point's height sampled from the terrain grid (plus a small offset) so borders hug the relief instead of floating at a fixed, flat altitude. A thin neatline rectangle around the full map extent gives a second, color-independent cue for where the map ends, since deep ocean blue alone is fairly close to the page's black background.
   * **City markers & labels.** The principal cities from Step 2 are plotted as small markers sampled to sit just above the terrain at their coordinates. Labeling all 54 at once makes dense clusters (greater Bogotá, Florida) unreadable, so only the `TOP_LABEL_COUNT` most-visited cities get a permanent on-map label — `"City - <year of last visit>"` — while the rest still get a marker and the same text on hover; every marker's *fill color* separately encodes recency (dark = long ago, bright = recent, via Plotly's built-in "Plasma" scale) through a shared `coloraxis`, independent of the terrain's own color scale.
   * **UI chrome.** A title, two color legends (terrain "Visit intensity" and marker "Last visit" recency, positioned so they don't collide), a short "how to read this map" note, and three camera-preset buttons (Isometric, Top view, and a "Most visited" button that frames whichever city has the most visits) sit on top of the 3D scene. The buttons drive `scene.camera` through Plotly's own `updatemenus`/`relayout` mechanism — `scene_camera_for()` converts a target lon/lat into the normalized coordinate space Plotly uses internally for a manual-aspect-ratio scene, so a button can frame a real place without hardcoding scene-space numbers. The toolbar's built-in camera-reset ("home") icon stays visible (`displayModeBar=True`) alongside free drag-to-rotate/scroll-to-zoom/double-click-to-reset.
   * **Colorblindness (CVD) validation.** Both color scales are checked computationally, not eyeballed: each is run through simulated protanopia/deuteranopia/tritanopia (Machado, Oliveira & Fernandes, 2009 matrices), and the worst OKLab color distance between adjacent scale stops is measured and printed. This is a read-only diagnostic — it documents where the simulated colors get harder to tell apart (the terrain's green-to-khaki transition is the weakest spot, a known characteristic of hypsometric relief maps) and notes the existing mitigations (elevation is real 3D height + shading, not color alone; recency is also anchored to lightness and repeated as exact text in the hover and table) without changing how the map looks for typical color vision.
   * **Loading overlay.** A full-screen overlay with a spinner covers the page until `plotly.js` finishes building the WebGL scene (the exported file embeds the full dataset, so it can take a moment to become interactive), then fades out automatically.
   * **Accessible table view.** A plain HTML `<table>` — not a Plotly widget — is appended below the map with every principal city (readable by screen readers or if WebGL isn't available), plus a live vanilla-JS search box that filters rows by city, state, or country as you type.
   * Exported as a single self-contained page (`fig.to_html(...)` embedded in a hand-built HTML shell with the table/search/loading-overlay markup): `interactive_3d_map.html`. Open it in any browser — no external 3D software required.

   **Key tunable parameters** (all defined at the top of the last notebook cell):

   | Parameter | Default | Effect |
   |---|---|---|
   | `PLOTLY_GRID_RESOLUTION` | `600` | Grid density; higher = smoother terrain, larger/slower HTML file. |
   | `PLOTLY_BANDWIDTH_FACTOR` | `0.65` | KDE bandwidth multiplier; higher = wider, gentler hills (too high merges nearby cities). |
   | `DENSITY_GAMMA` / `SECONDARY_GAMMA` | `0.4` / `0.2` | Two-stage compression strength; lower = smaller cities pulled up more relative to the dominant peak. |
   | `NOISE_AMPLITUDE` / `NOISE_SMOOTHNESS` | `0.5` / `0.8` | Strength and blur radius of the boundary/slope noise; lower smoothness = tighter, more numerous irregularities. |
   | `LAND_MASK_BLUR_SIGMA` | `0.7` | How gently height tapers right at the coastline. |
   | `TERRAIN_Z_ASPECT` | `0.015` | Vertical exaggeration of the whole terrain relative to the map footprint. |
   | `OCEAN_MIN` / `OCEAN_MAX` / `PLAINS_MAX` | `-0.42` / `-0.20` / `0.09` | Color bands reserved for deep/shallow water and mottled plains. |
   | `TOP_LABEL_COUNT` | `15` | How many of the 54 principal cities get an always-on map label; the rest are hover-only. |

---

## 📸 Results

| Type | Generated file | Description |
|---|---|---|
| City data | `visited_cities.csv` | Principal cities visited, with coordinates, most recent visit date, and visit count. |
| 3D map | `interactive_3d_map.html` | Interactive relief map of visited places — rotate, zoom, and hover for details, right in your browser. |

---

## 📜 License

Project developed for personal use and as a data visualization portfolio piece.
