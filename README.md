# NYC Park Visitation Analysis

Analysis of NYC park visitation patterns using Advan (SafeGraph) mobility data, Census ACS demographic data, and NYC Parks GIS data. The central research question is: which parks does each census tract visit?

---

## Directory Structure

```
project/
├── notebooks/
│   ├── 00_retrieval.ipynb            # Dewey/Advan download (non-functional — subscription expired)
│   ├── 01_acs.ipynb                  # Census ACS 2022 API pull
│   ├── 02_poi_flows.ipynb            # Placekey-level flow records
│   ├── 03_parks.ipynb                # Park audit, property centroids, placekey lookup
│   ├── 04_aggregation.ipynb          # Tract-level and placekey-level aggregations
│   ├── 05_property_aggregation.ipynb # Property-level flow aggregation
│   ├── 06_analysis.ipynb             # Plots, top 10 report, OSM-overlaid maps
│   ├── 07_model.ipynb                # PPML gravity model (placekey and property level)
│   └── 08_model_visualization.ipynb  # Model interpretation: distance decay, gradients, residuals
│
└── data/
    ├── raw/
    │   ├── advan/                    # 3 monthly CSVs (Apr–Jun 2022)
    │   ├── census/                   # tl_2023_36_tract shapefile
    │   ├── forever_wild/             # NYC_Parks_Forever_Wild_20260205.csv
    │   └── park_properties/          # NYC Parks property shapefile
    ├── intermediate/
    │   ├── park_flows_placekey.csv
    │   ├── park_property_centroids.csv
    │   ├── property_placekeys.csv
    │   ├── tract_park_aggregation.csv
    │   └── placekey_aggregation.csv
    └── output/
        ├── parks_audit.csv
        ├── park_flows_property_10km.csv
        └── census_tracts_acs_2022.csv
```

---

## Data Sources

| Source | Description |
|---|---|
| Advan / SafeGraph | Monthly mobility CSVs. POI visit counts and visitor home census block groups. Apr–Jun 2022. |
| Census ACS 2022 | Tract-level demographics for NYC counties 005, 047, 061, 081, 085. |
| NYC Parks Properties | GIS shapefile of park property polygons. Key fields: `gispropnum`, `signname`, `acres`. |
| Forever Wild | CSV of Forever Wild designated sites. Joined to audit by `gispropnum`. |
| Census Tract Shapefile | `tl_2023_36_tract.shp` — tract boundaries and centroid computation. |

---

## Notebooks

### 00 — Retrieval
Downloads Advan data via the Dewey platform. **Non-functional** — subscription has expired. Raw CSVs in `data/raw/advan/` are the only available data.

### 01 — ACS
Pulls ACS 2022 5-year estimates from the Census API for all NYC tracts. Computes racial composition as proportions of total population. Age under 15 and over 65 are summed across individual sex/age buckets. Sentinel value `-666666666` for missing income is left as-is in the raw output.

Output: `data/output/census_tracts_acs_2022.csv`

### 02 — POI Flows
Reads each Advan CSV and parses the `VISITOR_HOME_CBGS` JSON column. Each CBG is truncated to 11 characters to get a tract ID. For each park POI × tract pair, records the Euclidean distance from the POI point to the tract centroid in EPSG:2263 (feet → km). No distance filter is applied here — all flows are recorded. Handles empty strings and JSON parse errors.

Output: `data/intermediate/park_flows_placekey.csv`

### 03 — Parks
Loads all Advan POIs and filters to NYC (within NYC boundary union, or has flow records). Performs a spatial join of POI points to park property polygons (predicate: `within`). Unmatched POIs fall back to nearest-neighbor with no distance cap. Joins Forever Wild status by `gispropnum`. Outputs three files.

Outputs:
- `data/output/parks_audit.csv`
- `data/intermediate/park_property_centroids.csv`
- `data/intermediate/property_placekeys.csv`

### 04 — Aggregation
Two aggregation views of `park_flows_placekey.csv`, both sharing one load.

**Tract aggregation:** for each tract, total visits and a list of `[placekey, distance_km, visits]` triplets across all visited parks.

**Placekey aggregation:** for each park POI, total visits and a list of `[tract, distance_km, visits]` triplets across all visiting tracts. Controlled by `MAX_DIST_KM` at the top of the notebook — set to `None` to include all flows, or a number to restrict to nearby tracts only. Currently disabled (`None`).

Outputs:
- `data/intermediate/tract_park_aggregation.csv`
- `data/intermediate/placekey_aggregation.csv`

### 05 — Property Aggregation
Maps each placekey to its property via the audit. A tract-property pair qualifies if any POI in the property is within 10km of the tract centroid. Once qualified, the property's visit count for that tract is the **maximum** across all POIs in the property (not the sum). Many properties contain spatially overlapping POIs (e.g. Central Park has dozens of placekeys covering the same area), and summing would systematically double-count visitors recorded under multiple POIs. `max` is conservative — it under-counts when POIs are genuinely separate, but avoids over-counting in dense properties. This choice is under review; alternatives (hierarchical group-then-sum, visitor-overlap dedup, sum-with-cap) are documented in the notebook.

Display distance is centroid-to-centroid (tract centroid → property polygon centroid). `all_placekeys` is populated from `property_placekeys.csv` — includes all POIs in the property, not just those with flows from that tract.

Output: `data/output/park_flows_property_10km.csv`

### 06 — Analysis
Produces exploratory plots and a top-10 report. Loads the property flow file, parks audit, and the two aggregation files from notebook 04.

Plots: visit distributions by property/tract/placekey; tract diversity (unique parks per tract); POI reach (unique tracts per POI); top-10 most-visited properties; POI visit-intensity heatmap with OSM basemap; choropleth maps of visits to Prospect Park and Central Park with OSM basemaps; combined map of all NYC park properties (light green) with Forever Wild nature areas highlighted (darker green) on top of OSM tiles.

All map cells use `contextily` to fetch OpenStreetMap tiles (data is reprojected to EPSG:3857 / Web Mercator before plotting).

### 07 — Model
Fits two Poisson Pseudo-Maximum-Likelihood (PPML) gravity models — one at the placekey level, one at the property level — and compares their coefficients.

**Step-by-step:**

1. **Path setup** — declares all input file paths and the Census sentinel value constant.
2. **Load data** — reads placekey flows, property flows, parks audit, and ACS demographics.
3. **Clean ACS** — reconstructs the standard 11-digit GEOID (`geoid_full`) from the integer state/county/tract columns. Replaces Census sentinel values (`-666666666`) with `NaN`.
4. **Impute missing ACS from spatial neighbors** — identifies tracts that appear in the flow data but lack complete ACS coverage (either absent from the ACS pull entirely, or with suppressed/missing values). For each such tract, all queen-contiguous (touching-boundary) neighbors are found using the NYC tract shapefile in EPSG:2263. Missing column values are filled with the mean of those neighbors' non-null values. Tracts absent from the ACS table entirely are added as new rows.
5. **Build placekey model dataframe** — joins park attributes (acres, Forever Wild flag) from the audit and ACS demographics from the home tract. Computes `log_dist`, `log_area`, `log_pop`, and `is_nature`. Drops rows with any remaining NaN in model columns.
6. **Build property model dataframe** — same as step 5 but using the property-level flow file.
7. **PPML placekey model (#1)** — fits `visits ~ is_nature + log_dist + log_area + log_pop + is_nature × {demographic vars}` via `statsmodels` GLM with Poisson family.
8. **PPML property model — binary nature (#2)** — same specification on the property-level data.
9. **PPML property model — continuous nature (#3)** — same data and demographic predictors as #2, but replaces the binary `is_nature` flag with `nature_frac`, the share of the property's area that is Forever Wild (∈ [0, 1]). Both as a main effect and in every interaction term. This lets the "nature gradient" scale with how natural the park actually is rather than treating any FW designation as equivalent.
10. **Coefficient comparison** — plots model #1 vs #2 coefficients with 95% CIs side-by-side. Model #3 prints its own summary; coefficients are not directly comparable to #2 because a unit increase in `nature_frac` (0 → 1) is not the same as flipping `is_nature`.

**Model formulas:**

Models #1 and #2 (binary):
```
visits ~ is_nature + log_dist + log_area + log_pop
       + is_nature × median_income
       + is_nature × {white, black, asian, aian, other_race, two_or_more}
       + is_nature × {under_15, over_65}
```

Model #3 (continuous, property level only):
```
visits ~ nature_frac + log_dist + log_area + log_pop
       + nature_frac × median_income
       + nature_frac × {white, black, asian, aian, other_race, two_or_more}
       + nature_frac × {under_15, over_65}
```

The interaction coefficients capture whether the demographic gradient for natural-area parks differs from that for general parks. In model #3, the slope of each demographic effect changes linearly with `nature_frac`.

### 08 — Model Visualization
Re-fits the three PPML models from notebook 07 in a condensed prep block, then explores them with interpretation plots that don't fit the workflow of the model notebook itself:

1. **Distance-decay curves** — predicted visits vs distance for FW vs non-FW parks, log-y scale.
2. **Demographic gradient panels** — for each of the 9 demographic predictors, predicted visits over the 5th–95th percentile range at FW=0 and FW=1, holding everything else at sample mean.
3. **Continuous nature-frac gradients** — same panels for model #3, with one curve per `nature_frac` level (0, .25, .5, .75, 1.0).
4. **Coefficient heatmap** — all three models side by side with significance stars overlaid.
5. **Predicted vs observed** — calibration plot using decile-binned medians and 25th–75th percentile bands.
6. **Elasticity bar chart** — % change in expected visits per +10pp / +$10k shift, separately for standard and FW parks.
7. **Residual diagnostics** — distribution of Pearson residuals, residuals vs predicted, and top 10 over/under-predicted tract-property pairs.
8. **Nature-frac response curve** — model #3 prediction sweeping `nature_frac` from 0 to 1 at low / median / high tract income.
9. **Empirical attendance rates** — observed visits per 1k residents by demographic decile × FW status (no model — direct data).

---

## Output File Schemas

### `park_flows_placekey.csv`
One row per tract–POI pair.

| Column | Description |
|---|---|
| `tract_i` | Census tract GEOID (11-digit string) |
| `park_j` | Advan placekey |
| `visits` | Visit count from tract i to park j |
| `distance_km` | Euclidean distance from tract centroid to POI (km, EPSG:2263) |

Sort: `tract_i`, `park_j`

---

### `parks_audit.csv`
One row per Advan POI. Maps each placekey to its NYC Parks property.

| Column | Description |
|---|---|
| `placekey` | Advan placekey |
| `parent_placekey` | Parent placekey if POI is a sub-location; null otherwise |
| `name` | POI name from Advan (`LOCATION_NAME`) |
| `gis_prop_num` | NYC Parks property number |
| `property_name` | Official NYC Parks property name (`signname` from shapefile) |
| `visits` | Total `RAW_VISIT_COUNTS` across all 3 months |
| `acres` | Property acreage from NYC Parks shapefile |
| `forever_wild_id` | Forever Wild designation ID; null if not designated |
| `nature_fraction` | Share of property area that is Forever Wild, ∈ [0, 1]. Computed as `sum(fw_acres) / property_acres`, clipped to [0, 1]. NaN where property `acres` is missing or non-positive. |

Sort: `gis_prop_num`, `placekey`

---

### `park_flows_property_10km.csv`
One row per tract–property pair. Only pairs where at least one POI in the property is within 10km of the tract centroid are included.

| Column | Description |
|---|---|
| `tract_i` | Census tract GEOID |
| `gis_prop_num` | NYC Parks property number |
| `property_name` | Official NYC Parks property name |
| `visits` | Total visits from tract i to any POI in this property |
| `distance_km` | Euclidean distance from tract centroid to property polygon centroid (km) |
| `all_placekeys` | All placekeys in this property (from audit) |

Sort: `tract_i`, `distance_km`, `gis_prop_num`

---

### `census_tracts_acs_2022.csv`
ACS 2022 demographics for all NYC census tracts.

| Column | Description |
|---|---|
| `geoid` | 11-digit census tract GEOID |
| `total_population` | Total tract population |
| `median_household_income` | Median household income; -666666666 where unavailable |
| `white_pct`, `black_pct`, `aian_pct`, `asian_pct`, `nhpi_pct`, `other_race_pct`, `two_or_more_races_pct` | Racial composition as proportion of total population |
| `under_15_pct` | Proportion of population under age 15 |
| `over_65_pct` | Proportion of population over age 65 |

---

### `tract_park_aggregation.csv`
One row per tract.

| Column | Description |
|---|---|
| `tract_i` | Census tract GEOID |
| `total_visits` | Total visits from this tract to all parks |
| `park_visits` | List of `[placekey, distance_km, visits]` triplets |

Sort: `tract_i`

---

### `placekey_aggregation.csv`
One row per park POI.

| Column | Description |
|---|---|
| `placekey` | Advan placekey |
| `total_visits` | Total visits to this POI across all tracts |
| `num_tracts` | Number of unique tracts that visited this POI |
| `tract_list` | List of unique census tract GEOIDs that visited this POI |

Sort: `placekey`  
Filtered by: `MAX_DIST_KM` in `04_aggregation.ipynb` (currently disabled)

---

### Intermediate Files

| File | Description |
|---|---|
| `park_property_centroids.csv` | Centroid x/y (EPSG:2263, feet) per `gis_prop_num`. Used in 05 for tract-to-property distance. Sort: `gis_prop_num` |
| `property_placekeys.csv` | All placekeys per property from audit. Ensures complete POI lists in property flows. Sort: `gis_prop_num` |

---

## Key Concepts

**Placekey** — SafeGraph universal POI identifier. Multiple placekeys can share one `gis_prop_num`.

**gis_prop_num** — NYC Parks GIS property number. The aggregation unit for parks in the final output.

**Parent placekey** — when a POI is a sub-location of a larger place (e.g. Sheep Meadow within Central Park), it carries a `parent_placekey`. Used in 05 to deduplicate visits at the property level.

**10km filter** — applied at the property level in 05. A tract-property pair qualifies if any POI in the property is within 10km of the tract centroid. Once qualified, all of that property's visits from that tract are included.

**Forever Wild** — NYC Parks designation for natural areas. Joined by `gispropnum`, not spatially.

**Distance** — all distances are Euclidean computed in EPSG:2263 (NY State Plane, feet), then converted to km via `feet × 0.3048 / 1000`.

---

## Known Issues

### 1. Advan Visit Count Cap (709,622)
Advan caps visit counts at 709,622. Multiple POIs within a single property (e.g. Central Park) show this exact value. Visit counts should be treated as an index of relative popularity, not absolute counts.

### 2. Parent Placekey Dedup Incomplete
The deduplication logic in 05 drops child POIs only when their `parent_placekey` belongs to the same property in the current dataset. In practice 0 rows are removed for Central Park — none of its POIs have a `parent_placekey` that points to another POI in M010. The parent POI likely exists in Advan but was not included in the NYC-filtered dataset.

**Impact:** some properties may still have double-counted visits where the parent POI was filtered out.

### 3. 00_retrieval Non-Functional
The Dewey/Advan subscription has expired. Notebook 00 cannot be re-run. The raw Advan CSVs in `data/raw/advan/` cover April–June 2022 only.

### 4. ACS Neighbor Imputation Is an Approximation
Notebook 07 step 3b imputes missing ACS values from spatial neighbors. The imputed values are means of adjacent tracts and do not reflect any sampled data. Tracts with no neighbors that have valid ACS data (e.g. isolated island tracts) remain missing and are excluded from the model via `dropna`. The imputation reduces but does not eliminate data loss in the model.
