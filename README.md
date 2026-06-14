# Shared Mobility Torino

**Data-driven analysis of free-floating e-scooter sharing services (Bird, Lime and VOI) operating in the city of Turin (Torino), Italy.**

This repository contains the data-cleaning pipelines, exploratory analyses, geospatial mapping, demand modelling, and economic (revenue/cost) modelling produced for a Shared Mobility / Transport Planning university project. It compares how people use e-scooters across Turin, where they travel to and from, how operators might price and run their fleets profitably, and how e-scooters stack up against other ways of getting around the city (car, bike, public transport, taxi, etc.).

The repository is organised as a sequence of Jupyter notebooks ("EX1", "EX2"... style exercises), each building on the outputs of the previous ones.

---

## Table of Contents

1. [Who this is for](#who-this-is-for)
2. [Project context & goals](#project-context--goals)
3. [Repository structure](#repository-structure)
4. [The data](#the-data)
5. [Analysis workflow — what each notebook does](#analysis-workflow--what-each-notebook-does)
6. [Results folder — interactive maps & outputs](#results-folder--interactive-maps--outputs)
7. [Key results so far](#key-results-so-far)
8. [Glossary — key concepts explained simply](#glossary--key-concepts-explained-simply)
9. [Setup & requirements](#setup--requirements)
10. [How to reproduce the analysis](#how-to-reproduce-the-analysis)
11. [Known limitations & notes](#known-limitations--notes)

---

## Who this is for

This README is written so that it can be useful to **everyone** who opens this repository, regardless of background:

- **Students** — a guided tour of a full real-world data analysis pipeline (cleaning → exploration → spatial analysis → demand modelling → economics), with a glossary of technical terms.
- **Professors / reviewers** — a map of the notebooks, what each one demonstrates, and the methodology behind each step.
- **Data/transport specialists** — details on data cleaning rules, coordinate reference systems, spatial joins, OD (Origin–Destination) matrices, gravity/decay models, and generalised cost formulas.
- **Managers / non-technical readers** — a plain-language summary of what was found (usage patterns, revenue/cost estimates, profitability comparison between operators) without needing to read any code.

---

## Project context & goals

Three micromobility operators run dockless e-scooter fleets in Turin:

- **Bird**
- **Lime**
- **VOI**

Each operator provided trip-level records (anonymised) covering roughly **2024–2025**. The project uses this data to answer questions such as:

1. **How is each fleet actually used?** When do people ride (time of day, day of week, season), how long/far are trips, how fast do scooters travel, and how many vehicles are active?
2. **Where do people go?** Building **Origin–Destination (OD) matrices** between Turin's official statistical zones, identifying the busiest routes and zones, separately for morning peak, evening peak, and off-peak periods.
3. **Where do scooters sit idle?** Estimating average **parking duration** per zone (the time a scooter sits unused between two trips) and relating it to demand.
4. **Is the business viable?** Estimating **revenue, costs, taxes and profit** for each operator under simplified pricing/cost assumptions, then comparing the three companies.
5. **How does e-scooter sharing compare to other transport modes?** Calculating the **generalised cost** (money + value of time) of a typical "Home → Politecnico di Torino" commute by private car, car sharing, bike, bike sharing, e-scooter (shared/private), taxi, ride-hailing, and public transport.

---

## Repository structure

```
Shared Mobility torino/
│
├── Data/                          # Raw / merged input datasets (see "Data availability" below)
│   ├── df_all_bird.csv            # Cleaned, merged Bird trip data (2024–2025)
│   ├── df_all_lime.csv            # Cleaned, merged Lime trip data (2024–2025)
│   ├── df_all_voi.csv             # Cleaned, merged VOI trip data (2024–2025)
│   └── zone_statistiche_geo/      # Turin's official "Zone Statistiche" — 94 zone polygons
│       ├── zone_statistiche_geo.shp / .shx / .dbf / .prj / .qml
│
├── Notebooks/                     # All analysis notebooks (see section below for details)
│   ├── EX3.ipynb
│   ├── BirdAnalysis.ipynb
│   ├── LimeAnalysis.ipynb
│   ├── VoiAnalysis.ipynb
│   ├── Bird_OD_Matrix.ipynb
│   ├── Lime_OD_Matrix.ipynb
│   ├── Voi_OD_Matrix.ipynb
│   ├── EX4_1_BIRD.ipynb / EX4_1_LIME.ipynb / EX4_1_VOI.ipynb
│   ├── EX4_2_overlay_bird.ipynb / EX4_2_overlay_lime.ipynb / EX4_2_overlay_voi.ipynb
│   ├── EX4-3_BIRD_Revenue.ipynb / EX4_3_LIME_Revenue.ipynb / EX4_3_VOI_Revenue.ipynb
│   ├── EX4_Comparison.ipynb
│   ├── EX5_1.ipynb
│   ├── trip_base_data.csv         # Small reference table used by EX5_1 (mode-by-mode trip times)
│   └── newplot.png                # Exported chart image
│
├── Results/                        # Generated outputs: interactive maps (HTML), GeoPackages, GeoJSON
│   ├── index.html                  # Simple landing page linking to OD flow maps
│   ├── *heatmap*.html              # Folium heatmaps of trip starts/ends
│   ├── *overlay*.html              # Parking-duration choropleths overlaid with OD flow lines
│   ├── od_*.html / od_*.geojson    # Origin–Destination flow maps and data
│   ├── *parking*.geojson           # Average parking duration per zone, per operator
│   └── *.gpkg                      # GeoPackage exports of cleaned point data
│
├── requirements.txt                # Python dependencies for the notebooks
└── .gitignore
```

---

## The data

### 1. Trip-level data (one row = one rental/trip)

| Operator | Source file(s) | Approx. size | Notes |
|---|---|---|---|
| **Bird** | `Data/df_all_bird.csv` | ~147 MB, ~840k trips | Merged from separate 2024 and 2025 export files |
| **Lime** | `Data/df_all_lime.csv` | ~2 GB | Merged from monthly CSV exports (`Torino_Corse24-25_MENSILI...`) |
| **VOI** | `Data/df_all_voi.csv` | ~80 MB, ~291k trips | Merged from monthly Excel exports (`DATINOLEGGI_*.xlsx`) |

Each file contains, at minimum:
- A **vehicle ID** (`ID_VEICOLO`, `Targa veicolo`, etc.)
- **Start/end timestamps** (`DATAORA_INIZIO` / `DATAORA_FINE`, or `Data inizio/fine corsa` for VOI)
- **Start/end coordinates** (latitude/longitude, in `EPSG:4326`)
- **Trip distance** (km) and **duration** (minutes)
- Operator-specific extras (e.g. battery level for VOI/Lime, GPS route polyline `PERCORSO` for Bird/Lime)

Column names differ between operators (Italian field names from each operator's export format) — the notebooks rename them to a common schema (`start_lat`, `start_lon`, `end_lat`, `end_lon`, `start_time`, `end_time`, etc.) before combining or comparing them.

### 2. Geographic zoning data

`Data/zone_statistiche_geo/` is a shapefile of **Turin's official "Zone Statistiche"** (statistical zones used by the City of Turin for planning), containing **94 zones**, each with:
- `ZONASTAT` — numeric zone code
- `DENOM` — zone name (e.g. *Piazza Vittorio Veneto*, *Borgata Vittoria*, *Mongreno*…)
- `geometry` — polygon boundary (original CRS: `EPSG:3003`, reprojected to `EPSG:4326` for use with web maps)

All Origin–Destination and parking-duration analyses assign every trip's start/end point to one of these 94 zones via a **spatial join**.

### Data availability

The `Data/` folder is excluded from version control (`.gitignore`) because the raw/merged files are very large (the Lime file alone is ~2 GB) and may contain data the operators consider sensitive. **If you clone this repository, the `Data/` folder and most `*.geojson` intermediate files will not be present** — they need to be regenerated or supplied separately. See [Known limitations & notes](#known-limitations--notes).

---

## Analysis workflow — what each notebook does

The notebooks were built incrementally as a series of exercises ("EX..."). The table below groups them by stage. Each stage typically reads the output of a previous one.

### Stage 0 — Data assembly (`EX3.ipynb`)

Loads the three operators' merged datasets, standardises column names to a common schema (`start_lat`, `start_lon`, `end_lat`, `end_lon`, `start_time`, `end_time`, `operator`), concatenates them into one combined dataframe, and exports point geometries (start/end locations for an example month) as GeoPackage files for use in GIS software.

### Stage 1 — Per-operator data cleaning & exploratory analysis

| Notebook | What it does |
|---|---|
| **BirdAnalysis.ipynb** | Cleans Bird's 2024 and 2025 raw exports: removes trips with zero/negative distance, unrealistic speeds (>30 km/h), GPS "jumps" (start/end points implausibly far apart vs. the reported distance), fixes Daylight-Saving-Time timestamp errors, and removes trips where start ≈ end coordinates but a non-zero distance is reported. Then explores the cleaned data: daily/weekly/monthly trip volumes, active-scooter counts, hour-of-day and weekday×hour usage heatmaps, trip duration/distance distributions, and Folium spatial heatmaps (overall, morning peak, evening peak, weekends, marker clusters). |
| **LimeAnalysis.ipynb** | Merges Lime's monthly CSV exports for 2024–2025, applies similar cleaning rules (duration > 0, distance > 0, 0 < speed ≤ 30 km/h, end time ≥ start time, start ≠ end location), then runs the same family of temporal analyses (monthly/weekly/hourly trends, weekday vs weekend, per-vehicle usage intensity) plus Folium heatmaps for day/night and weekday/weekend, a sample of actual GPS routes ("route intensity" map), and a "top origins" map of the busiest pickup points. |
| **VoiAnalysis.ipynb** | Merges VOI's monthly Excel exports, parses VOI's custom date format, removes duplicate rental IDs, filters invalid durations/distances/speeds and trips longer than 180 minutes, then produces the equivalent temporal trends (daily/monthly/weekly/hourly, weekday vs weekend), hour×weekday heatmaps for 2024 vs 2025, a city-wide spatial heatmap, route-intensity maps (including a side-by-side 2024-vs-2025 dual map), and a "top 20 origins" map. |

### Stage 2 — Origin–Destination (OD) matrices

| Notebook | What it does |
|---|---|
| **Bird_OD_Matrix.ipynb** | Spatially joins every Bird trip's start and end point to one of the 94 statistical zones, builds the full OD matrix (trips between every zone pair), splits it by **morning peak (7–9h)** and **evening peak (16–19h)**, identifies the top production/attraction zones, computes trip distances between zone centroids, and fits two simple **gravity-type demand models**: trips vs. distance and trips vs. **generalised cost** (a combined money + time cost per trip), including an OLS regression and elasticity estimate. Exports the top-200 OD flows (overall, morning, evening) as GeoJSON line layers for mapping. |
| **Lime_OD_Matrix.ipynb** | A more structured OD pipeline for Lime: cleans the data, classifies every trip into **Morning Peak / Evening Peak / Off-Peak**, spatially joins origins and destinations to zones, builds OD matrices per time period, and runs **structural analyses** — intra-zonal vs. inter-zonal trip shares, and "concentration" analysis (what share of all trips is covered by the single busiest OD pair, and by the top 10). Produces flow maps of the top-200 OD pairs per period. |
| **Voi_OD_Matrix.ipynb** | Equivalent OD pipeline for VOI: cleans the data, assigns origin/destination zones, classifies trips by Morning Peak / Evening Peak / Off-Peak, builds OD matrices per period, and exports interactive flow maps (top-200 flows) for the morning and evening peaks. |

### Stage 3 — Parking duration by zone (`EX4_1_*`)

| Notebook | What it does |
|---|---|
| **EX4_1_BIRD / EX4_1_LIME / EX4_1_VOI.ipynb** | For each vehicle, computes the **idle/parking time** between the end of one trip and the start of its next trip. Joins the end-location of each trip to a statistical zone, then computes the **average parking duration per zone**. Visualises the result as a choropleth map (quantile classes) with both `matplotlib`/`geopandas` static maps and an interactive Folium choropleth. Exports the result as `*_parking.geojson` for use in Stage 4. |

### Stage 4 — Overlay maps: parking duration + OD flows (`EX4_2_*`)

| Notebook | What it does |
|---|---|
| **EX4_2_overlay_bird / EX4_2_overlay_lime / EX4_2_overlay_voi.ipynb** | Combines the parking-duration choropleth (Stage 3) with the OD flow lines (Stage 2) on a single interactive Folium map, separately for the **morning peak** and **evening peak**. This lets you see, at a glance, *where scooters tend to sit idle* relative to *where the demand is actually flowing*. |

### Stage 5 — Revenue, cost & profit modelling per operator (`EX4-3_*` / `EX4_3_*`)

| Notebook | What it does |
|---|---|
| **EX4-3_BIRD_Revenue.ipynb** | Cleans the Bird dataset further (drops unrealistic durations/speeds/GPS noise), then estimates **revenue** using an assumed two-tier pricing model (60% "Bird+" subscribers at a discounted per-minute rate, 40% pay-as-you-go users) plus a weekly subscription fee. Estimates **fixed costs** (per-scooter capital/insurance) and **variable costs** (per-minute electricity/maintenance), then computes profit before/after a 28% tax rate. Visualises revenue by hour of day, revenue distribution per scooter, and revenue vs. trip duration (Plotly charts). |
| **EX4_3_LIME_Revenue.ipynb** | Same approach for Lime: an assumed "LimePrime" pricing tariff (flat fee for short trips, then per-minute pricing for longer trips) plus a monthly/weekly subscription estimate, fixed & variable costs, tax, and profit. Includes revenue distribution and revenue-vs-duration charts. |
| **EX4_3_VOI_Revenue.ipynb** | Same approach for VOI, using VOI's actual published unlock fee + per-minute rate. Removes duplicate rental records, computes total/unlock/time-based revenue shares, fixed & variable costs, profit before/after tax, and visualises the revenue composition (unlock vs. time-based) and revenue by hour. |

### Stage 6 — Cross-company comparison (`EX4_Comparison.ipynb`)

Brings together the revenue/cost/profit figures produced in Stage 5 for **VOI, Lime and Bird** into one comparison table, then computes and charts:
- Revenue, cost and profit **per scooter**
- **Profit margin** per operator
- Revenue vs. profit (grouped bar chart)
- **Cost-to-revenue ratio**
- Fleet size vs. profit ("scaling effect" scatter plot)
- Revenue vs. cost with a break-even reference line

### Stage 7 — Generalised cost of the "Home → Politecnico" trip (`EX5_1.ipynb`)

Defines a reference **2.5 km commute** ("Home to Politecnico di Torino") and, for nine transport modes — private car, car sharing, private bike, bike sharing, e-scooter sharing, private e-scooter, taxi, ride-hailing, and public transport — builds a table of travel time, access time, waiting time and out-of-pocket cost. It then:

- Computes the **generalised cost** = monetary cost + (total time × value of time, VOT)
- Performs a **sensitivity analysis** on VOT (€5, €10, €20 per hour) to see how mode rankings change for cost-sensitive vs. time-sensitive travellers
- Decomposes the time cost into travel/access/waiting components (stacked bar chart)
- Tests a **weighted-time** variant where waiting and access time are penalised more heavily than in-vehicle travel time (a common practice in transport modelling, since waiting "feels" longer)
- Runs a small **private-car ownership cost model** (annualised depreciation, insurance, tax, fuel, maintenance) to derive a realistic per-km cost for private car use
- Produces a **Pareto (cost vs. time) scatter plot** to visualise trade-offs between modes

---

## Results folder — interactive maps & outputs

The `Results/` folder contains the **outputs** of the notebooks above, ready to view without re-running anything:

- **`index.html`** — a minimal landing page linking to one of the OD flow maps.
- **Heatmaps** (`lime_*_heatmap.html`, `voi_spatial_heatmap.html`, `marker_cluster*.html`) — interactive Folium maps showing where trips start/end, split by day/night, weekday/weekend, etc.
- **OD flow maps** (`od_*flows*.html`, `od_*.geojson`, `*OD Flows*.html`) — the busiest origin→destination "desire lines" between statistical zones, for all-day, morning-peak, evening-peak and off-peak periods, per operator.
- **Parking duration layers** (`*_parking.geojson`) — average idle time per zone, per operator, used as choropleth fill.
- **Overlay maps** (`overlay_*.html`) — parking-duration choropleth + OD flow lines combined, by operator and time period.
- **Route intensity maps** (`*route_intensity*.html`, including a 2024-vs-2025 dual map for VOI) — visualisations of actual/sample GPS trajectories.
- **GeoPackages** (`*.gpkg`) — cleaned start/end point layers exported for use in desktop GIS tools (e.g. QGIS).

> **How to view:** open any `.html` file in this folder directly in a web browser — no server required, they are self-contained interactive Leaflet/Folium maps.

---

## Key results so far

> ⚠️ The figures below come from **simplified, assumption-based economic models** built for this exercise (assumed pricing tiers, flat per-scooter fixed costs, etc.) — they are **illustrative estimates for learning purposes**, not the operators' real financial results.

### Operator comparison (from `EX4_Comparison.ipynb`)

| Operator | Estimated Fleet Size | Total Revenue (€) | Total Cost (€) | Profit Before Tax (€) | Profit After Tax (€, 28% rate) |
|---|---:|---:|---:|---:|---:|
| **Bird** | 2,816 | 2,811,181.77 | 1,811,865.18 | 999,316.59 | 719,507.95 |
| **Lime** | 2,399 | 2,624,617.57 | 1,849,117.94 | 775,499.63 | 558,359.73 |
| **VOI**  | 1,445 |   729,176.50 |   611,965.40 | 117,211.10 |  84,391.99 |

Headline takeaways from the comparison notebook:
- **Bird and Lime** operate larger fleets and generate substantially higher absolute revenue and profit than VOI in this model.
- **Profit margins and per-scooter profitability** vary — the notebook normalises revenue/cost/profit *per scooter* to check whether bigger fleets are actually more efficient, not just bigger.
- A **break-even (revenue = cost) reference line** is used to visually check how far each operator sits above the break-even point.

### Generalised cost of a 2.5 km "Home → Politecnico" trip (`EX5_1.ipynb`)

Across the nine modes considered, the analysis shows how **ranking by total cost changes depending on the traveller's Value of Time (VOT)**:
- At a **low VOT** (cost-sensitive traveller), modes with little or no monetary cost (private bike, walking-equivalent, public transport with its low fare) tend to rank best.
- At a **high VOT** (time-sensitive traveller), faster door-to-door modes (private car, private e-scooter) become relatively more attractive despite higher monetary cost.
- **Waiting and access time** (e.g. waiting for public transport, walking to a shared vehicle) have an outsized effect on perceived cost once they are weighted more heavily than in-vehicle travel time — a key reason shared-mobility and PT services aim to minimise wait/access times.

---

## Glossary — key concepts explained simply

| Term | Plain-language meaning |
|---|---|
| **Trip / Rental** | One scooter ride: from unlock to lock, with a start time/location and an end time/location. |
| **OD Matrix (Origin–Destination Matrix)** | A table that counts how many trips went *from* each zone *to* each zone — basically "who travels where, and how much". |
| **Statistical Zone (ZONASTAT)** | One of 94 official neighbourhoods/areas the City of Turin divides itself into for planning and statistics. |
| **Spatial join** | Matching each trip's GPS coordinates to the zone polygon that contains that point — i.e., "which neighbourhood did this trip start/end in?" |
| **Choropleth map** | A map where areas (zones) are shaded by colour according to a value (e.g. darker = longer average parking time). |
| **Heatmap (spatial)** | A map showing where activity is concentrated, using a colour gradient (red = lots of trips, blue/transparent = few). |
| **Parking duration** | How long a scooter sits unused between the end of one trip and the start of the next — an indicator of fleet "idle time" and potential rebalancing needs. |
| **Peak period** | Times of day with the heaviest travel demand — here defined as **Morning Peak (~7–10h)** and **Evening Peak (~17–20h)**, with everything else as **Off-Peak**. |
| **Intra-zonal vs. inter-zonal trips** | Intra-zonal = trip starts and ends in the *same* zone (short, local trips). Inter-zonal = trip crosses from one zone to another. |
| **Distance-decay / gravity model** | A simple model based on the idea that "demand between two places falls off as the distance (or cost) between them increases" — used here to relate the number of trips between zones to the distance or cost of travelling between them. |
| **Generalised Cost (GC)** | The *total* cost of a trip from the traveller's perspective: money paid **plus** the monetary value of the time spent (travel + waiting + access), so that different modes can be compared on a single scale. |
| **Value of Time (VOT)** | How much one minute (or hour) of travel time is "worth" in money terms — used to convert time into an equivalent monetary cost. A higher VOT means time matters more relative to price. |
| **Haversine distance** | The straight-line ("as the crow flies") distance between two GPS points on the Earth's surface — used here to sanity-check the distances reported by the operators and catch GPS errors. |
| **GPS jump** | A data-quality issue where the recorded start/end coordinates imply an unrealistic distance (e.g. teleporting across the city) — usually a GPS error, filtered out during cleaning. |
| **CRS (Coordinate Reference System)** | The "coordinate system" a map/dataset uses (e.g. `EPSG:4326` = standard GPS latitude/longitude; `EPSG:3003` = a metric Italian projection). Data from different sources must be converted to the same CRS before combining. |
| **GeoJSON / GPKG (GeoPackage)** | Standard file formats for storing geographic data (points, lines, polygons) — GeoJSON is text-based and web-friendly, GPKG is a database-style format used by GIS software like QGIS. |
| **Folium** | A Python library used throughout this project to create interactive web maps (the `.html` files in `Results/`). |

---

## Setup & requirements

This project uses **Python 3** with Jupyter notebooks. The core dependencies are listed in [`requirements.txt`](requirements.txt) and include:

- **Data handling:** `pandas`, `numpy`
- **Geospatial:** `geopandas`, `shapely`, `pyproj`, `pyogrio`
- **Mapping:** `folium`, `branca`
- **Plotting:** `matplotlib`, `seaborn`
- **Notebook tooling:** `jupyter`, `ipykernel`, `nbconvert`, etc.

### Installing

```bash
# from the repository root
python3 -m venv venv
source venv/bin/activate          # on Windows: venv\Scripts\activate
pip install -r requirements.txt
```

> **Additional packages used by some notebooks but not listed in `requirements.txt`:** a few notebooks also use `plotly` (interactive charts in the revenue/comparison notebooks), `mapclassify` (required by GeoPandas for the `scheme="quantiles"` choropleth maps), and `statsmodels` (OLS regression in `Bird_OD_Matrix.ipynb`). Install these as needed:
>
> ```bash
> pip install plotly mapclassify statsmodels
> ```

---

## How to reproduce the analysis

The notebooks were developed iteratively and **some contain hard-coded local file paths** (e.g. `/Users/.../Desktop/...`) from the original author's machine — see [Known limitations](#known-limitations--notes). To reproduce the full pipeline from scratch, the general order is:

1. **Obtain the raw operator exports** (Bird 2024/2025 CSVs, Lime monthly CSVs, VOI monthly Excel files) and place them where the notebooks expect them (update paths as needed), or start directly from the merged files in `Data/` if you have them.
2. **`EX3.ipynb`** — merge/standardise the three operators into a common schema.
3. **`BirdAnalysis.ipynb`**, **`LimeAnalysis.ipynb`**, **`VoiAnalysis.ipynb`** — clean each operator's data and produce the merged `Data/df_all_*.csv` files used by everything downstream.
4. **`Bird_OD_Matrix.ipynb`**, **`Lime_OD_Matrix.ipynb`**, **`Voi_OD_Matrix.ipynb`** — build OD matrices and export OD GeoJSON layers (consumed by Stage 4).
5. **`EX4_1_BIRD/LIME/VOI.ipynb`** — compute per-zone parking duration and export `*_parking.geojson` (consumed by Stage 4).
6. **`EX4_2_overlay_*.ipynb`** — combine outputs of steps 4 & 5 into overlay maps.
7. **`EX4-3_*_Revenue.ipynb`** — run the per-operator economic models.
8. **`EX4_Comparison.ipynb`** — combine the Stage 7 results into the cross-operator comparison.
9. **`EX5_1.ipynb`** — standalone; only needs `trip_base_data.csv` (already included) and can be run independently of the rest.

If you only want to **view results**, you don't need to run anything — open the files in `Results/` in a browser.

---

## Known limitations & notes

- **Large/raw data not in version control.** `Data/` (raw and merged CSVs, up to ~2 GB) and all `*.geojson` files are excluded via `.gitignore`. Anyone cloning this repo will need the original data exports (or the merged `df_all_*.csv` files) to re-run the pipeline from the start.
- **Hard-coded absolute paths.** Several notebooks reference paths specific to the original author's computer (e.g. `/Users/mina/Desktop/...`, `/Users/mina/Desktop/EX4/Turin_Zone.gpkg`). These need to be updated to local/relative paths before re-running on another machine.
- **Mixed-language code comments.** Some notebooks contain comments in Persian/Farsi alongside English — this doesn't affect functionality but is worth noting for readers unfamiliar with the language.
- **Economic model is illustrative.** The revenue/cost/profit figures rely on assumed pricing tiers, a flat per-scooter capital cost, and simplified variable-cost rates. They are designed to teach the *methodology* of cost/revenue modelling for shared mobility, not to represent each company's actual finances.
- **Exploratory development history.** Because these notebooks evolved exercise-by-exercise, some cells are exploratory/iterative (e.g. repeated attempts at a working map style) and not all intermediate cells are needed for the final output — they are kept for transparency into the analysis process.
- **`venv/` / `.venv/` folders.** Local Python virtual environments may appear in the working tree but are excluded from git via `.gitignore`.
