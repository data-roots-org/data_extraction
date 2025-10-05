# Tropical Tree Cover — Processing Pipeline

This repo contains a small, reproducible pipeline to fetch and convert tropical tree cover data into vector polygons for interactive analysis and mapping.

## What we do (in short):

 - "Reads two inputs placed under ./outputs/ (downloaded manually):"
    - Tropical_Tree_Cover.csv (resource catalog / links)
    - Tropical_Tree_Cover.geojson (tile index with metadata and download URLs)
 - Use 01_tropical_tree_cover_tif_download.ipynb to download GeoTIFFs for a list of tiles.

 - Use 02_geotiff_to_geojson.ipynb to process those GeoTIFFs and generate GeoJSON polygons.

## 1) Repository Structure
.
├─ outputs/
│  ├─ Tropical_Tree_Cover.csv         # Manually downloaded CSV (resources or derived table)
│  ├─ Tropical_Tree_Cover.geojson     # Manually downloaded tile index (vector polygons)
│  └─ ...                             # Notebook outputs (downloaded TIFs, generated GeoJSONs)
├─ 01_tropical_tree_cover_tif_download.ipynb   # Downloader notebook (TIFs by tile)
├─ 02_geotiff_to_geojson.ipynb                  # Raster → vector notebook (TIF → GeoJSON)
└─ README.md


The CSV and GeoJSON are considered inputs and are placed in ./outputs/ so the notebooks can reference them directly. Feel free to relocate them, but update the paths in the notebooks accordingly.

## 2) Data Overview

Tropical_Tree_Cover.geojson
A tile index (vector layer) covering tropical regions. Each feature typically includes:

tile_id (e.g., 10S_050W)

F10m_download (URL for 10 m probability raster per tile)

other metadata fields (e.g., half_hectare_download, ObjectId, etc.)

Tropical_Tree_Cover.csv
A tabular file listing available resources or a curated subset. The downloader notebook uses it to locate TIF download links and to match tile IDs.

The actual probability/gridded data are in GeoTIFF files referenced by the above resources. The GeoJSON is an index, not the raster data themselves.

## 3) Software Requirements

Python 3.10+ recommended (3.11 works too)

Packages:

pandas, requests, tqdm (downloading & tables)

rasterio, numpy, matplotlib (raster processing & previews)

shapely, pyproj (vectorization & geodesic calculations)

Quick setup (Windows PowerShell)
cd <your-repo-path>
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install pandas requests tqdm rasterio numpy matplotlib shapely pyproj


In VS Code, select the .venv interpreter (Command Palette → Python: Select Interpreter).

## 4) Workflow
Step A — Download GeoTIFFs by tile

Notebook: 01_tropical_tree_cover_tif_download.ipynb

Inputs:

outputs/Tropical_Tree_Cover.csv and/or outputs/Tropical_Tree_Cover.geojson

Configure:

CSV_PATH: points to your CSV (resource table or tiles_subset_properties.csv)

OUTPUT_DIR: where TIFs will be saved

TILES_WANTED: e.g. ["10S_050W", "10S_060W", "20S_050W", "20S_060W"]

(Optional) USE_BEARER=True + BEARER_TOKEN="..." if the server requires auth

What it does:

Detects CSV format and extracts TIF URLs.

Streams downloads with progress bars (tqdm).

Writes a download_log.csv to track success, errors, or existing files.

Output: A folder with one or more *.tif files per requested tile.

Step B — Convert GeoTIFFs → GeoJSON polygons

Notebook: 02_geotiff_to_geojson.ipynb

Inputs: GeoTIFFs downloaded in Step A

Configure:

TIF_PATH: path to a single TIF to process (you can run in a loop for many)

Output path for OUT_GEOJSON

Choose one mode:

Continuous (e.g., probabilities): set USE_THRESHOLD=True, THRESHOLD=0.4

Optional AUTOSCALE=True to auto-detect 0–1 vs 0–100 vs 0–10000

Binary/Categorical (e.g., already-thresholded masks): set USE_THRESHOLD=False, CLASS_VALUE=1

Optional controls for performance/quality:

OVERVIEW: downsample factor (e.g., 8 or 16 for very large rasters)

SIMPLIFY: geometry simplification tolerance (Douglas–Peucker)

MIN_AREA_M2: drop tiny polygons by geodesic area (WGS84)

DISSOLVE: merge all polygons into one MultiPolygon (heavier; consider as a separate step)

BBOX: clip by bounding box in the raster CRS (process in tiles/patches if needed)

What it does:

Reads the raster (optionally downsampled).

Builds a boolean mask (threshold or class value).

Vectorizes the mask into polygons (raster → vector).

Applies simplification, area filter, optional dissolve.

Saves GeoJSON for visualization and GIS workflows.

Output: *.geojson polygon layer(s) corresponding to the chosen tiles and settings.

## 5) Memory & Performance Guidance (8 GB machines)

Processing large rasters can be memory-intensive. Recommended settings:

Downsample (OVERVIEW): start with 8 (or even 16 for huge rasters).
This reduces pixel count by 64× (or 256×) before polygonization.

Simplify (SIMPLIFY): e.g., 0.001 (if CRS is WGS84, ~100 m around the equator).
Reduces vertices dramatically.

Minimum area (MIN_AREA_M2): e.g., 20000 (20,000 m²) to drop small fragments.

Avoid dissolve initially. If you need a single geometry, dissolve later as a separate step (e.g., in QGIS or with GDAL/OGR).

Clip in chunks using BBOX and run multiple passes (N×M grid). Merging the resulting GeoJSONs is much cheaper than dissolving everything at once.

## 6) Outputs

Raster downloads: *.tif per tile (naming depends on the source URL).

Vectorized results: *.geojson per tile (or per chunk), ready for:

Web maps (Leaflet/MapLibre)

Desktop GIS (QGIS/ArcGIS)

Further analysis (area, overlays, joins)

Optional downstreams (notebooks/scripts not included by default):

Convert to Cloud-Optimized GeoTIFF (COG) or XYZ tiles for fast web delivery.

Upload to Google Earth Engine as image assets with properties (via CLI manifests).

Vector cleaning and topology operations (QGIS/OGR).

## 7) Reproducibility Notes

Keep the versions of rasterio, shapely, and pyproj pinned in a requirements.txt if you need strict reproducibility.

The AUTOSCALE logic relies on sampling to infer the native data range (0–1 vs 0–100 vs 0–10000). For formal runs, document the threshold policy per dataset.

If resource URLs require tokens, do not commit secrets; use environment variables or local .env files ignored by git.

## 8) Acknowledgments

Data sources and original licensing belong to their respective owners/providers. Please review and respect the dataset licenses and terms of use.

Thanks to the NASA Hackathon community for guidance and review.

## 9) Contact

For questions or contributions, please open an issue or PR in this repo. If you’re on the project team, feel free to tag the maintainers on our internal channel.

## TL;DR for NASA reviewers

We ingest a tile index + resource table, download 10 m tree cover rasters for selected tiles, and convert them into vector polygons under controlled memory/precision settings. The outputs (GeoJSON) are ready for visualization and overlay analyses.