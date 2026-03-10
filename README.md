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
│   └── 06_analysis.ipynb             # Plots and top 10 report
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
Maps each placekey to its property via the audit. Drops child POIs whose `parent_placekey` belongs to the same property to avoid double-counting visits. A tract-property pair qualifies if any POI in the property is within 10km of the tract centroid. Once qualified, visits from all POIs in that property are summed. Display distance is centroid-to-centroid (tract centroid → property polygon centroid). `all_placekeys` is populated from `property_placekeys.csv` — includes all POIs in the property, not just those with flows from that tract.

Output: `data/output/park_flows_property_10km.csv`

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
| `tract_list` | List of `[tract, distance_km, visits]` triplets |

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
