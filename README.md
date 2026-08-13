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
pip install pandas numpy matplotlib seaborn scipy geopandas
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

Data processing runs through a Jupyter Notebook ([Create_2D_Image.ipynb](Create_2D_Image.ipynb)) structured in four main phases:

1. **Extraction & cleaning**
   * Reads the JSON file, filtering the visit structure (`visit -> topCandidate -> placeLocation`).
   * Parses the `"geo:LATITUDE,LONGITUDE"` geographic format into float coordinates.

2. **Regional filtering (Americas)**
   * Restricts the geographic window from Patagonia (Argentina, $-60^\circ$ lat) to Alaska (USA, $75^\circ$ lat), and from $-170^\circ$ to $-30^\circ$ longitude. This maximizes spatial resolution over the areas actually visited.

3. **Heightmap generation for Blender**
   * Uses `scipy.stats.gaussian_kde` to compute a smooth density matrix, free of contour-level artifacts.
   * The matrix is normalized strictly between `0.0` (pure black = $Z=0$) and `1.0` (pure white = maximum elevation).
   * Exported as a high-resolution image: `heatmap_heightmap_americas.png`.

4. **Geospatial validation**
   * Generates a verification image (`heatmap_americas_reference.png`) overlaying the density layer and GPS points on continental borders downloaded from *Natural Earth* via `geopandas`.

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

## 📸 Results

| Type | Generated file | Description |
|---|---|---|
| 2D texture | `heatmap_heightmap_americas.png` | Grayscale displacement map (heightmap). |
| Validation | `heatmap_americas_reference.png` | Density overlay on continental borders. |
| 3D render | `render_final.png` | Stylized 3D terrain with neon emission, rendered in Blender. |

---

## 📜 License

Project developed for personal use and as a data visualization portfolio piece.
