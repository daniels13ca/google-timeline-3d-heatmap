# 🗺️ 3D Geo-Density Topographic Surface (Google Timeline)

An artistic, interactive 3D visualization that transforms Google Timeline location history into an elevated topographic relief map — rendered as a self-contained, interactive HTML page with **Plotly**.

Instead of relying on static bars or traditional 2D heatmaps, this project converts visit frequency into **fluid three-dimensional hills and mountains** rising out of a realistic ocean/plains relief map, highlighting the cities and regions where you spent the most time and traveled the most.

---

## 🎯 Project Goal

* **Data transformation:** Process semantic JSON files exported directly from the Google Maps app.
* **Density mapping:** Compute a 2D Kernel Density Estimation (**KDE**) over visit locations, compressed so smaller/less-visited cities remain visible next to the dominant home peak.
* **3D representation:** Render an interactive **Plotly** 3D surface — ocean in blue, unvisited land as green plains, visited areas rising into hills/mountains colored by elevation — with country borders and labeled city markers, viewable in any browser.

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
   * Uses `scipy.stats.gaussian_kde` to compute a smooth visit-density surface, with a wider bandwidth than a "sharp peak" look so hills read as rounded terrain rather than spikes.
   * Applies a two-stage power-law (gamma) compression so smaller/less-visited cities remain visible next to the dominant home peak, then a smooth (zero-slope) transition so hills taper gently into the plain instead of ending in a hard edge.
   * Classifies every grid cell as land or ocean (via a Natural Earth boundary lookup) to apply a hypsometric ("real relief map") color ramp — ocean blue, plains green, hills/mountains in tan → brown → gray → white — and drapes the country borders at ground level.
   * Plots the principal cities from Step 2 as markers, with hover tooltips showing the city name, last visit date, and visit count.
   * Exported as a self-contained interactive page: `interactive_3d_map.html`. Open it in any browser — no external 3D software required — and rotate/zoom/pan freely.

---

## 📸 Results

| Type | Generated file | Description |
|---|---|---|
| City data | `visited_cities.csv` | Principal cities visited, with coordinates, most recent visit date, and visit count. |
| 3D map | `interactive_3d_map.html` | Interactive relief map of visited places — rotate, zoom, and hover for details, right in your browser. |

---

## 📜 License

Project developed for personal use and as a data visualization portfolio piece.
