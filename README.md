# 🗺️ 3D Geo-Density Topographic Surface (Google Timeline)

An artistic, interactive 3D visualization that transforms Google Timeline location history into an elevated topographic density map (a *3D Surface Heightmap*) in **Blender**.

Instead of relying on static bars or traditional 2D heatmaps, this project converts visit frequency and dwell time into **fluid three-dimensional peaks and mountains**, highlighting the cities and regions where you spent the most time and traveled the most.

---

## 🎯 Project Goal

* **Data transformation:** Process semantic JSON files exported directly from the Google Maps app.
* **Density mapping:** Compute a 2D Kernel Density Estimation (**KDE**) to generate a calibrated grayscale texture (*heightmap*).
* **3D representation:** Use **Blender** to displace the geometry of a flat base plane and apply procedural shaders that emit neon light based on elevation ($Z$).

---

## 🛠️ Requirements & Installation

### Python Environment

Make sure you have Python 3.10+ and the following libraries installed:

```bash
pip install pandas numpy matplotlib seaborn scipy geopandas reverse_geocoder
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

### Step 2: Processing & Texture Generation (Python)

Data processing runs through a Jupyter Notebook ([Create_2D_Image.ipynb](Create_2D_Image.ipynb)) structured in five main phases:

1. **Extraction & cleaning**
   * Reads the JSON file, filtering the visit structure (`visit -> topCandidate -> placeLocation`).
   * Parses the `"geo:LATITUDE,LONGITUDE"` geographic format into float coordinates.

2. **Regional filtering (Americas)**
   * Restricts the geographic window from Patagonia (Argentina, $-60^\circ$ lat) to Alaska (USA, $75^\circ$ lat), and from $-170^\circ$ to $-30^\circ$ longitude. This maximizes spatial resolution over the areas actually visited.

3. **Heightmap generation for Blender**
   * Uses `scipy.stats.gaussian_kde` to compute a smooth density matrix, free of contour-level artifacts.
   * Visit density is heavily skewed toward home/frequent locations, so the matrix goes through two power-law (gamma) compression passes — `DENSITY_GAMMA` then `SECONDARY_GAMMA`, both tunable in the notebook — before normalizing. This keeps less-visited cities from being flattened to near-invisible by the home peak, while still guaranteeing areas with no visits land at exactly `0.0` (pure black = $Z=0$); only visited areas get pulled toward `1.0` (maximum elevation).
   * Exported as a high-resolution image: `heatmap_heightmap_americas.png`.

4. **Geospatial validation**
   * Generates a verification image (`heatmap_americas_reference.png`) overlaying the density layer and GPS points on continental borders downloaded from *Natural Earth* via `geopandas`.

5. **City identification (for labels)**
   * Reverse geocodes every visit offline via `reverse_geocoder`, clusters nearby visits into metro areas, and labels each cluster with the largest known city nearby (using a GeoNames major-cities lookup) rather than the closest small locality.
   * Keeps only the most recent visit per city and exports the result to `visited_cities.csv`, used later to place labels in Blender (see [Step 4](#step-4-add-labels--pins)).

---

### Step 3: Building the 3D Scene in Blender

Once `heatmap_heightmap_americas.png` has been generated, follow these steps in Blender to build the 3D terrain:

#### 1. Create a proportional plane

* Open Blender and delete the default scene (`X` > *Delete*).
* Add a plane (`Shift + A` > **Mesh** > **Plane**).
* Adjust its dimensions in the side panel (`N`) to preserve the geographic aspect ratio of the Americas region ($140^\circ$ lon $\times$ $135^\circ$ lat):
  * **Width ($X$):** `14.0 m`
  * **Height ($Y$):** `13.5 m`

#### 2. Subdivide the geometry

* Press `Tab` to enter **Edit Mode**, right-click the plane, select **Subdivide**, and set the number of cuts to `100`.
* Return to **Object Mode** (`Tab`) and add a **Subdivision Surface** modifier (`Add Modifier` > **Generate** > **Subdivision Surface**).
* Set it to **Simple** mode with `Viewport = 3` and `Render = 4`.

#### 3. Apply 3D displacement

* With the plane selected, add a **Displace** modifier (`Add Modifier` > **Deform** > **Displace**).
* Create a new texture and, in the **Texture Properties** tab, load the `heatmap_heightmap_americas.png` image.
* Configure the modifier parameters:
  * **Coordinates:** `UV`
  * **Direction:** `Z`
  * **Midlevel:** `0.000`
  * **Strength:** between `1.0` and `2.5`, depending on the desired peak height.

#### 4. Materials & neon shading

* Go to the **Shading** tab and open the **Shader Editor**.
* Connect the plane's height coordinate ($Z$) to a **ColorRamp** node:
  * `Texture Coordinate` (Generated) $\rightarrow$ `Separate XYZ` (Z axis) $\rightarrow$ `ColorRamp` (Fac).
* Set the neon color palette on the **ColorRamp**:
  * **0.0 (base/sea):** transparent dark blue or black (`#04070f`).
  * **0.2 (travel routes):** neon cyan (`#00f0ff`).
  * **0.6 (frequent visits):** magenta (`#ff007f`).
  * **1.0 (peaks/residence):** bright warm white (`#ffffff`).
* Connect the **ColorRamp** output to `Base Color` and `Emission Color` on the **Principled BSDF** node.
* Raise `Emission Strength` to `5.0`–`10.0`.

#### 5. Lighting & post-processing (bloom)

* In **World Properties**, set the world background to pure black (`#000000`).
* Add a dim, angled area light to bring out the shadows of the 3D relief.
* Go to the **Compositing** tab, enable **Use Nodes**, and add a **Glare** node (*Type: Fog Glow*) between `Render Layers` and `Composite`.
* Position the camera at an isometric or low-angle perspective, adjust focus with **Depth of Field**, and press `F12` to render.

---

### Step 4: Add Labels & Pins

To add narrative context to the final render, you can place floating labels showing each place name and the year it was visited (e.g. `BOGOTÁ '26` or `NEW YORK '22`).

Before starting in Blender, make sure you've run the corresponding cell in the Jupyter notebook to generate `visited_cities.csv`.

---

#### Method A: Manual Creation & Positioning (Recommended for 5 to 15 key milestones)

Ideal if you only want to highlight your most significant trips or cities, keeping the render clean and minimal.

1. **Create the text object:**
   * In Blender, press `Shift + A` > **Text**.
   * Press `Tab` to enter **Edit Mode**, type the label (e.g. `CHICAGO 2026`), then press `Tab` again to exit.
   * In the **Object Data Properties** tab (the 'a' icon), set *Paragraph $\rightarrow$ Alignment* to **Center**, both horizontally and vertically.
   * Pick a clean typeface under **Font** — sans-serif fonts like *Helvetica*, *Inter*, or *Futura* in uppercase work great.

2. **Position it over the peak:**
   * Move the text (`G`) and place it in 3D space, floating just above the density peak for that city.
   * Rotate the text (`R` + `X` + `90`) so it stands upright, oriented toward the final camera angle.

3. **Create the vertical pin/line:**
   * Press `Shift + A` > **Curve** > **Path** (or **Mesh** > **Cylinder** with a thin radius).
   * Orient the curve vertically, connecting the base map ($Z=0$) or the elevated peak to the base of the floating label.
   * In the curve's properties, set *Bevel Depth* to `0.02 m` to give it some thickness.

4. **Apply a neon material:**
   * Create a new material for the text and the line.
   * Switch the surface to **Emission**, or raise `Emission Strength` on the **Principled BSDF** node to a value between `5.0` and `8.0`.
   * Pick a color that contrasts with the map (e.g. white or cyan).

---

#### Method B: Automated via Scripting (Recommended for multiple locations)

If you want to automatically import dozens of locations from `visited_cities.csv`, converting geographic coordinates $(\text{Lat}, \text{Lon})$ into $(X, Y, Z)$ positions inside the Blender scene:

1. **Open the Scripting workspace:**
   * Go to the **Scripting** tab at the top of Blender and click **New**.

2. **Run the import script:**
   * Copy and paste the script below into the editor, making sure to update `CSV_PATH` to point to your `visited_cities.csv` file:

```python
import csv
import bpy

# File path to the exported cities CSV
CSV_PATH = "/path/to/your/visited_cities.csv"

# Americas Bounding Box Limits (Must match Python processing)
MIN_LON, MAX_LON = -170.0, -30.0
MIN_LAT, MAX_LAT = -60.0, 75.0

# Blender Plane Dimensions (X = 14.0m, Y = 13.5m)
PLANE_WIDTH_X = 14.0
PLANE_HEIGHT_Y = 13.5

# Offset to center (Plane origin at 0,0,0)
HALF_X = PLANE_WIDTH_X / 2.0
HALF_Y = PLANE_HEIGHT_Y / 2.0


def geo_to_blender(lat, lon):
  """Converts Geographic Coordinates to Blender (X, Y) space."""
  x = ((lon - MIN_LON) / (MAX_LON - MIN_LON)) * PLANE_WIDTH_X - HALF_X
  y = ((lat - MIN_LAT) / (MAX_LAT - MIN_LAT)) * PLANE_HEIGHT_Y - HALF_Y
  return x, y


# Read CSV and spawn labels
with open(CSV_PATH, mode="r", encoding="utf-8") as file:
  reader = csv.DictReader(file)

  for row in reader:
    city = row["city"]
    year = row["last_visit"][:4]
    lat = float(row["latitude"])
    lon = float(row["longitude"])

    # Convert to 3D Scene Coordinates
    x_pos, y_pos = geo_to_blender(lat, lon)
    z_pos = 1.5  # Suspension height above plane

    # 1. Create Text Object
    label_text = f"{city.upper()} '{year[-2:]}"
    bpy.ops.object.text_add(location=(x_pos, y_pos, z_pos))
    text_obj = bpy.context.active_object
    text_obj.data.body = label_text
    text_obj.scale = (0.2, 0.2, 0.2)

    # Rotate upright facing Front/Camera
    text_obj.rotation_euler = (1.5708, 0, 0)  # 90 deg on X axis

    # 2. Create Vertical Pin Line
    bpy.ops.mesh.cylinder_add(
        radius=0.01, depth=z_pos, location=(x_pos, y_pos, z_pos / 2.0)
    )

print("✅ Successfully imported and positioned all landmarks in Blender!")
```

3. **Run it:** click **Run Script** (▶) in the Scripting workspace. Blender spawns a text label and a vertical pin for every visited city, positioned using the same lat/lon-to-plane mapping as the heightmap from Step 3.

---

## 📸 Results

| Type | Generated file | Description |
|---|---|---|
| 2D texture | `heatmap_heightmap_americas.png` | Grayscale displacement map (heightmap), scoped to the Americas. |
| Validation | `heatmap_americas_reference.png` | Density overlay on continental borders, scoped to the Americas. |
| City data | `visited_cities.csv` | Principal cities visited, with coordinates and most recent visit date — used for Blender labels. |
| 3D render | `render_final.png` | Stylized 3D terrain with neon emission, rendered in Blender. |

---

## ⚠️ Privacy Note

This repository tracks `location-history.json`, which contains your raw, precise location history exported from Google Timeline. Think twice before pushing this repo to a public remote — consider `.gitignore`-ing the raw JSON, `visited_cities.csv`, and any heightmap/reference images derived from it if you don't want your personal movement data to be publicly visible.

---

## 📜 License

Project developed for personal use and as a data visualization portfolio piece.
