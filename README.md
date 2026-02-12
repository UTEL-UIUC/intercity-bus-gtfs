# Intercity Bus GTFS Workflow

## Overview

This document provides a complete, step-by-step guide for cleaning, merging, validating, and processing GTFS (General Transit Feed Specification) data for intercity bus feeds. `GTFS_Data_Processing.ipynb` contains the end-to-end Data_Processing code used to process the data. The goal is to prepare the data for mapping and analysis in GIS tools like ArcGIS Pro or QGIS.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Configuration](#configuration)
3. [Step 1: Download GTFS Feeds](#step-1-download-gtfs-feeds)
4. [Step 2: Split Combined Feeds](#step-2-split-combined-feeds)
5. [Step 3: Validate GTFS Feeds](#step-3-validate-gtfs-feeds)
6. [Step 4: Filter Intercity Routes](#step-4-filter-intercity-routes)
7. [Step 5: Consolidate Feeds](#step-5-consolidate-feeds)
8. [Step 6: Tidy and Optimize with gtfstidy](#step-6-tidy-and-optimize-with-gtfstidy)
9. [Step 7: Calculate Weekly Trip Frequency](#step-7-calculate-weekly-trip-frequency)
10. [Step 8: Build Custom GeoJSON from GTFS](#step-8-build-custom-geojson-from-gtfs)
11. [Step 9: Interactive Web Map](#step-9-interactive-web-map)
12. [Notes and Best Practices](#notes-and-best-practices)
13. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Software
- [Python](https://www.python.org/) 3.x
- Required Python libraries:
  - `pandas` - for data manipulation
  - [`gtfs_kit`](https://pypi.org/project/gtfs-kit/) - for GTFS feed handling
  - [`folium`](https://pypi.org/project/folium/) - for interactive web mapping (includes `FeatureGroup`, `LayerControl`, `Map`, `PolyLine`, `CircleMarker`, `Popup`, and plugins)
  - Standard libraries: `os`, `shutil`, `tempfile`, `csv`, `pathlib`, `collections`, `re`, `datetime`, `math`, `json`, `subprocess`, `itertools`, `typing`
- [ArcGIS Pro](https://www.esri.com/en-us/arcgis/products/arcgis-pro/overview/) or [QGIS](https://qgis.org/)
- [`gtfstidy`](https://github.com/patrickbr/gtfstidy) - for GTFS optimization and validation
   - Download pre-compiled binary from [releases](https://github.com/patrickbr/gtfstidy/releases), extract to a folder in your PATH
   - Verify installation of `gtfstidy -h` (should display help menu with available flags)

### Data Requirements
- All GTFS feeds should be placed in a single root directory
- Each agency's feed should be in its own subfolder

---

## Configuration

Update these variables before running the workflow:

```python
# Configuration (edit these before running)
PROJECT_ROOT = Path.cwd()
DATA_ROOT = PROJECT_ROOT / "data"  # place GTFS feeds + Agency_List.csv here
ROOT_DIR = DATA_ROOT / "gtfs_feeds"  # each agency in its own subfolder
RUN_TAG = "YYYY-MM-DD"  # update per run (e.g., 2026-01-20)

# Output directories
UNIQUE_FEEDS_DIR = DATA_ROOT / "Unique_Feeds"
MERGED_FEED_DIR = DATA_ROOT / "Merged_Feed"
MERGED_TIDY_FEED_DIR = Path(str(MERGED_FEED_DIR) + "_tidy")
OUTPUT_DIR = PROJECT_ROOT / "outputs"

# Output files
WEEKLY_TRIPS_CSV = OUTPUT_DIR / f"weekly_trips_per_shape_{RUN_TAG}.csv"
GEOJSON_OUTPUT = OUTPUT_DIR / f"Merged_Agencies_{RUN_TAG}.geojson"
MAP_OUTPUT = OUTPUT_DIR / f"Intercity_bus_map_{RUN_TAG}.html"
AGENCY_LIST_CSV = DATA_ROOT / "Agency_List.csv"

# Interactive route filtering (Step 4)
INTERACTIVE_ROUTE_FILTERING = True  # keep True for manual input
ROUTE_IDS_TO_REMOVE = []  # used when INTERACTIVE_ROUTE_FILTERING is False

# Create the directories if they don't exist
for _dir in [UNIQUE_FEEDS_DIR, MERGED_FEED_DIR, OUTPUT_DIR]:
   _dir.mkdir(parents=True, exist_ok=True)
```

---

## Step 1: Download GTFS Feeds

### Purpose
- Collect GTFS feeds from various transit agencies or sources
- [Transitland](https://www.transit.land/) and [Mobility Database](https://mobilitydatabase.org/) are the main data sources

### Process
1. Download GTFS feeds
2. When GTFS feeds are unavailable, create the feeds manually using schedules from the respective agencies. [o2g](https://github.com/hiposfer/o2g), [osm2gtfs](https://github.com/grote/osm2gtfs) and [GTFS Builder](https://www.nationalrtap.org/Technology-Tools/GTFS-Builder) are some of the tools to create these GTFS feeds. [Transcor Data Services (TDS)](https://maps.tds.ai/) is another source for manual data creation. For smaller agencies, the `.txt` files can be built by hand.
3. Unzip each feed into its own folder
4. Verify each folder contains the required GTFS `.txt` files
5. Maintain a reference spreadsheet `Agency_List.csv` with agency names, source URLs, source dates, and download dates.
6. Place all agency folders into `ROOT_DIR` (configured in the **Configuration** section above).
   - By default, the notebook expects feeds at: `DATA_ROOT/gtfs_feeds/`
   - The notebook also expects `Agency_List.csv` at: `DATA_ROOT/Agency_List.csv`

### Expected Output
A directory structure like:
```
data/gtfs_feeds/
├── Agency_1/
│   ├── agency.txt
│   ├── routes.txt
│   ├── trips.txt
│   └── ...
├── Agency_2/
│   ├── agency.txt
│   └── ...
└── ...
```

---

## Step 2: Split Combined Feeds to Unique Feed

### Purpose
Some agencies provide one GTFS feed covering multiple bus operators. This step splits these combined feeds into separate feeds per agency for easier cleaning and processing.

**Why this matters:**
- **Data quality control**: Issues in one agency's data won't affect validation of others
- **Easier troubleshooting**: Problems can be traced to specific agencies
- **Flexible processing**: Different agencies may require different cleaning approaches
- **Clearer attribution**: Each feed maintains its own agency identity
- **Modular updates**: Individual agencies can be updated without re-processing the entire combined feed

### How It Works

The splitting process maintains referential integrity by following the relational structure of GTFS:

1. **Read Agency Information**: Loads `agency.txt` to identify all agencies in the combined feed
   - Extracts unique `agency_id` values
   - Retrieves agency names for folder creation

2. **Filter by Agency** (following GTFS relationships): For each agency_id:
   - **Step 1**: Filter `routes.txt` by `agency_id` → gets all routes for this agency
   - **Step 2**: Filter `trips.txt` by `route_id` → gets all trips using those routes
   - **Step 3**: Filter `stop_times.txt` by `trip_id` → gets all stop times for those trips
   - **Step 4**: Filter `stops.txt` by `stop_id` → gets only stops actually used by this agency
   - **Step 5**: Filter `shapes.txt` by `shape_id` → gets route geometries for this agency's trips
   - **Step 6**: Filter `calendar.txt` and `calendar_dates.txt` by `service_id` → gets schedules for this agency's trips
   - **Step 7**: Filter optional files (`fare_rules.txt`, `fare_attributes.txt`, `transfers.txt`, `frequencies.txt`) where applicable

3. **Create Separate Folders**: Saves each agency's filtered data to its own folder
   - Folder names are sanitized to remove invalid characters
   - Each folder contains a complete, self-contained GTFS feed

### Key Functions
- `clean_folder_name()`: Removes invalid characters from folder names
- `split_gtfs_by_agency()`: Main function to split feeds

### Expected Output
```python
UNIQUE_FEEDS_DIR = DATA_ROOT / "Unique_Feeds"
 ```
Individual agency folders in `UNIQUE_FEEDS_DIR`, each containing a complete GTFS feed for that agency.

### Notebook Behavior (Auto-detect, Split Outputs, and What to Do Next)
- **Auto-detect (only split when needed)**: The notebook checks each folder under `ROOT_DIR` and only runs splitting when `agency.txt` indicates a **multi-agency feed** (typically `agency.txt` has **more than 1** agency row / `agency_id`).
- **Where split outputs go**: If a *combined* feed folder (under `ROOT_DIR`) is split, outputs are written under:
   - `UNIQUE_FEEDS_DIR/<combined_feed_folder>/<agency_name>/...`
- **If nothing is split**: If no multi-agency feeds are detected, splitting is skipped and `UNIQUE_FEEDS_DIR/` may remain empty.
- **Dropped stops report (optional output)**: During splitting, the notebook may write `dropped_stops_after_split.csv` into the per-agency output folder. This records stops that existed in the combined feed but were not referenced by that agency’s `stop_times.txt` (and therefore were not included in the split feed).
- **Continue the workflow after splitting**: Splitting writes outputs to `UNIQUE_FEEDS_DIR`, but later steps (validation, filtering, merge) iterate folders under `ROOT_DIR`. To continue processing split feeds, either:
   - set `ROOT_DIR` to the split output folder you want to process, or
   - copy/move the split agency folders back under `ROOT_DIR`.

---

## Step 3: Validate GTFS Feeds

### Purpose
Ensure data quality and integrity before processing by validating each GTFS feed against [GTFS specification](https://gtfs.org/documentation/schedule/reference/). [MobilityData GTFS Validator](https://github.com/MobilityData/gtfs-validator) and [Google GTFS Validator](https://developers.google.com/transit/gtfs) are some of the other official validators. However, they may not work properly if the GTFS feeds are manually created or incomplete.

### Validation Checks

#### 3.1 Required Files Check
The `check_required_files()` function compares the list of loaded files against required files. It then verifies presence of all required GTFS files:
- `agency.txt`, `stops.txt`, `routes.txt`, `trips.txt`, `stop_times.txt`, `calendar.txt`

and optional files:
- `calendar_dates.txt`, `fare_attributes.txt`, `fare_rules.txt`, `shapes.txt`, `frequencies.txt`, `transfers.txt`, `feed_info.txt`, `attributions.txt`

→ If any required files are missing, the validation skips that agency entirely.

#### 3.2 Required Fields Check
Ensures each file contains **mandatory fields** as defined by the [GTFS specification](https://gtfs.org/documentation/schedule/reference/):
- **agency.txt**: `agency_id`, `agency_name`
- **stops.txt**: `stop_id`, `stop_name`, `stop_lat`, `stop_lon`
- **routes.txt**: `route_id`, `agency_id`, and either `route_short_name` or `route_long_name`
- **trips.txt**: `route_id`, `service_id`, `trip_id`
- **stop_times.txt**: `trip_id`, `arrival_time`, `departure_time`, `stop_id`, `stop_sequence`
- **calendar.txt**: `service_id`, day columns, `start_date`, `end_date`

→ If required fields are missing, validation stops for that agency.

#### 3.3 Unique ID Check
Verifies uniqueness (no duplicate values) of primary keys:
- `route_id` in *routes.txt*
- `trip_id` in *trips.txt*
- `stop_id` in *stops.txt*

→ Prints any duplicate values found. This validation is required because duplicate IDs break relationships between tables and cause data integrity issues.

#### 3.4 Referential Integrity Check
The `check_referential_integrity(primary_df, primary_key, foreign_df, foreign_key, primary_name, foreign_name)` function ensures foreign keys exist in primary tables (relationships are valid):
- **Routes → Agency**: Every `agency_id` in *routes.txt* must exist in *agency.txt*
- **Trips → Routes**: Every `route_id` in *trips.txt* must exist in *routes.txt*
- **Stop Times → Trips**: Every `trip_id` in *stop_times.txt* must exist in *trips.txt*
- **Stop Times → Stops**: Every `stop_id` in *stop_times.txt* must exist in *stops.txt*
- **Trips → Calendar**: Every `service_id` in *trips.txt* must exist in *calendar.txt*

→ Prints missing IDs that break referential integrity. This validation is required because broken references mean trips/stops/routes can't be properly linked.

#### 3.5 Data Format Validation

- **Time Format**: 
    - HH:MM:SS (can exceed 24:00:00 for next-day service)
    - Use the `validate_time_format(time_str)` function with the `arrival_time` and `departure_time` fields in *stop_times.txt*
- **Date Format**:
    - YYYYMMDD
    - Use the `validate_date_format(date_str)` function with the `start_date` and `end_date` fields in *calendar.txt*
- **Coordinates**: 
    - Latitude (-90 to 90), Longitude (-180 to 180)
    - Use the `validate_lat_lon(lat, lon)`function with the `stop_lat` and `stop_lon` fields in *stops.txt*

→ Counts and reports invalid entries for each format type.

#### 3.6 Stop Sequence Check
The `check_stop_sequences(stop_times_df)` function verifies stop sequences are consecutive for each trip. It:
- Groups `stop_times.txt` by `trip_id`
- For each trip, checks if `stop_sequence` values form a consecutive series. Expected: `[1, 2, 3, 4]` or `[0, 1, 2, 3]` (starting from minimum value)

→ Lists trips with incorrect sequences and shows expected vs. actual values.

#### 3.7 Additional Validations
1. **Orphaned trip_ids**:
    - `trip_id`s that appear in *stop_times.txt* but NOT in *trips.txt*
    - These are "ghost trips" that can't be traced to routes or schedules
    - Saved in `orphaned_per_agency` dictionary for later cleanup

2. **Missing service_ids**:
    - `service_id`s referenced in *trips.txt* but NOT defined in *calendar.txt*
    - Trips with undefined services have no schedule information
    - Saved in `missing_per_agency` dictionary

3. **Unused shape_ids**:
    - `shape_id`s defined in *shapes.txt* but never referenced by any trip in *trips.txt*
    - These are dead data taking up space
    - Saved in `unused_per_agency` dictionary
    
4. **Identical shapes**:
    - Different `shape_id`s that have identical lat/lon coordinate sequences
    - The validator:
        - Groups shapes by `shape_id`
        - Creates tuple of `(lat, lon)` pairs sorted by `shape_pt_sequence`
        - Identifies groups with matching coordinate tuples
    - Duplicate shapes should be consolidated to reduce data redundancy
    - Saved in `identical_per_agency` dictionary with groups of matching shape IDs

5. **Single-day services**:
    - `service_id`s in *calendar.txt* where `start_date == end_date`
    - These are typically special one-time services (holidays, events) that may need review or removal for regular service analysis
    - Saved in `single_day_per_agency` dictionary
    - **Note**: In the merged feed preparation (`Step 6.1`), these single-day services are removed (and related rows in `calendar_dates.txt`, `trips.txt`, and `stop_times.txt`) prior to weekly frequency calculation to avoid inflating weekly service estimates.

6. **Unused Stops**:
    - `stop_id`s that appear in *stops.txt* but never appear in *stop_times.txt*
    - These stops are not used by any trip and may indicate leftover or orphaned records
    - Saved in `unused_stops_per_agency` dictionary

### Validation Output
For each agency, the validator prints:
- ✓ Pass indicators for successful checks
- ✗ Fail indicators with details for failed checks
- ⚠ Warnings for optional missing files
- ❌ Overall status (pass/fail) at the end

The validation results feed into subsequent cleaning cells which use the stored dictionaries to fix the identified issues.

---
## Step 4: Filter Intercity Routes

### Purpose

Review and identify routes that provide only intercity services. This step filters out local/urban routes, keeping only routes that connect cities or regions over longer distances. This is essential for focusing the analysis on intercity bus services.

### Process

1. **Calculate Route Distances**:
   - Uses the **`haversine()`** function to compute great-circle distances between consecutive shape points
   - The Haversine formula calculates distances along Earth's surface using latitude/longitude coordinates
   - Distances are summed for all points in sequence along each route's shape
   - If a route has multiple shapes, distances are aggregated
   - Results are converted to miles for display

2. **List All Routes**:
   - Loads all GTFS feeds using **`gk.read_feed()`** to access route information
   - For each route, displays: agency name, route ID, route name, and total distance in miles
   - This helps identify which routes are intercity (typically longer distances) vs. local routes

3. **Manual Route Selection**:
   - User reviews the route list and identifies non-intercity routes to remove
   - User inputs route IDs to remove (comma-separated)
   - The script filters out these routes while maintaining referential integrity
   - The removal list is **route_id-only** and is applied across **all agencies**. If multiple agencies reuse the same `route_id`, removing that ID will remove routes in every agency where it appears.
   - Recommendation: remove in small batches and re-check counts, or adjust the notebook to accept `(agency, route_id)` pairs if ID collisions are common in your inputs.

4. **Filter Related GTFS Files**:
   - **Routes**: Filters `routes.txt` to keep only selected route IDs using **`.isin()`** method
   - **Trips**: Filters `trips.txt` by `route_id`, then extracts related `trip_id`s, `service_id`s, and `shape_id`s
   - **Stop Times**: Filters `stop_times.txt` by `trip_id`, then extracts related `stop_id`s
   - **Stops**: Filters `stops.txt` to keep only stops used by remaining trips
   - **Calendar**: Filters `calendar.txt` and `calendar_dates.txt` by `service_id`
   - **Shapes**: Filters `shapes.txt` by `shape_id` to keep only shapes used by remaining trips
   - **Optional files**: Filters `fare_rules.txt`, `fare_attributes.txt`, `transfers.txt`, and `frequencies.txt` where applicable

5. **Save Filtered Feeds**:
   - Uses **`feed.to_file()`** to write the filtered GTFS feed back to each agency folder
   - Maintains the same folder structure with cleaned data

**Notebook artifact (optional output)**:
- During route filtering, the notebook may write `dropped_stops_after_route_filter.csv` into an agency folder when stops are dropped as a result of removing routes.

> Note: This step is optional when all routes in a feed represent intercity services. Route filtering is only necessary for feeds that include both intercity and local routes, and it can be skipped to reduce processing time when not needed.

### Key Functions

- **`haversine(lat1, lon1, lat2, lon2)`**: Calculates great-circle distance between two points in kilometers using the Haversine formula
- **`gk.read_feed()`**: Loads GTFS feed from directory using gtfs_kit
- **`.isin()`**: Pandas method to filter DataFrames by matching values in a list/set
- **`.dropna()`**: Removes rows with missing values
- **`feed.to_file()`**: Saves the filtered GTFS feed back to files

### Expected Output

Each agency folder contains a filtered GTFS feed with only intercity routes and their related data (trips, stops, shapes, etc.).

---

## Step 5: Consolidate Feeds

### Purpose
Merge all validated agency feeds into one unified GTFS dataset. Most GIS tools (e.g., [ArcGIS Pro](https://www.esri.com/en-us/arcgis/products/arcgis-pro/overview/), [QGIS](https://qgis.org/)) can only import one GTFS feed at a time.

**Why consolidate:**
- **gtfstidy requirement**: `gtfstidy` (used in **Step 6**) processes only a single GTFS feed at a time. It requires a consolidated `.zip` file containing all agencies' data as input, making this step essential before optimization and cleaning
- **Single import**: GIS tools typically handle one GTFS feed per import operation
- **Network analysis**: Enables system-wide route analysis and trip planning
- **Unified visualization**: All agencies appear in one cohesive map
- **Cross-agency comparisons**: Facilitates service frequency and coverage analysis
- **Simpler workflows**: One dataset is easier to manage than dozens of separate feeds

### Process

The consolidation process uses `gtfs_kit` to safely merge feeds while preserving data integrity:

1. **Load All Feeds**:
   - Iterate through all agency folders in `ROOT_DIR`
   - Load each feed using **`gk.read_feed()`** (gtfs_kit) with distance units set to kilometers
   - Add each successfully loaded feed to a feeds list
   - Skip any folders that don't contain valid GTFS files

2. **Merge Feeds**:
   - Start with the first feed as the base dataset
   - For each subsequent feed, concatenate all dataframes row-wise:
     - **Core files**: `agency.txt`, `stops.txt`, `routes.txt`, `trips.txt`, `stop_times.txt`
     - **Schedule files**: `calendar.txt`, `calendar_dates.txt`
     - **Geographic files**: `shapes.txt`
     - **Optional files** (if present): `fare_attributes.txt`, `fare_rules.txt`, `transfers.txt`, `frequencies.txt`, `feed_info.txt`, `attributions.txt`
   - Uses **`pd.concat()`** (pandas.concat) to append rows from each feed
   - Preserves all columns; adds NaN for missing columns in some feeds

3. **Save Merged Feed**:
   - Write combined feed to `MERGED_FEED_DIR` using **`feed.to_file()`** or gtfs_kit write functions
   - Creates all standard GTFS `.txt` files with data from all agencies
   - Resulting feed contains the union of all individual agency datasets
   - File sizes will be significantly larger than individual feeds

### Key Functions

- **`gk.read_feed()`**: Loads GTFS feed from directory using gtfs_kit
- **`pd.concat()`**: Concatenates multiple DataFrames row-wise to merge feeds
- **`feed.to_file()`**: Saves the merged GTFS feed to a directory

### Expected Output
```python
MERGED_FEED_DIR = DATA_ROOT / "Merged_Feed"
 ```
A single GTFS feed in `MERGED_FEED_DIR` containing data from all agencies created via **row-wise concatenation** (`pd.concat`) of GTFS tables.

**Important note about IDs:**
- The merge step **does not prefix or remap IDs** (`route_id`, `trip_id`, `stop_id`, `shape_id`, `service_id`, etc.).
- If multiple agencies reuse the same IDs, the merged feed can contain **ID collisions** until **Step 6.2**, where `gtfstidy -i` renumbers IDs in the modification phase.

---

## Step 6: Tidy and Optimize with **gtfstidy**

### Purpose
Prepare the merged GTFS for downstream analysis and mapping by fixing common issues, normalizing data, and aggressively deduplicating and cleaning using `gtfstidy`.

### 6.1 Preparation
Before running `gtfstidy`, ensure the merged feed is consistently formatted. This prevents crashes and validation errors during the gtfstidy process.

The preparation phase uses a **`clean_and_write_csv()`** helper function that:
- Normalizes ID fields (removes `.0` suffixes from numeric IDs, converts to strings)
- Ensures integer fields are properly formatted (not floats like `0.0`)
- Fills NaN values with empty strings
- Writes CSV files in GTFS-compliant format with minimal quoting
- Reads GTFS tables as strings and preserves blanks (avoids pandas auto-coercing IDs to floats/NaN, which can break joins and downstream tooling)

**Process**:

1. **Fix `agency.txt` timezone, URL, and email**:
   - **Problem**: Missing or invalid `agency_timezone` causes gtfstidy to fail validation
      - **Solution**: Standardize all agencies to a consistent IANA timezone (e.g., "America/New_York")
   - **Problem**: Missing `agency_url` violates GTFS spec
      - **Solution**: Add placeholder URLs (e.g., "https://example.com") if real URLs unavailable
      - **Additional**: Prepend `https://` to URLs missing protocol
   - **Problem**: Invalid `agency_email` entries (missing `@` symbol) cause validation errors
      - **Solution**: Clear invalid email entries (set to empty string)

2. **Remove orphaned `parent_station` references in `stops.txt`**:
   - **Problem**: `parent_station` values pointing to non-existent `stop_id`s cause referential integrity errors and gtfstidy parsing failures
      - **Solution**: Normalize `stop_id` and `parent_station` values (handle float/int/string mismatches), then set orphaned `parent_station` values to empty string
      - **Why it happens**: Stations may be removed during cleaning but child stops still reference them
      - **Critical**: gtfstidy crashes on parsing if orphaned references exist

3. **Fix `shapes.txt` - remove `shape_dist_traveled` and reindex sequences**:
   - **Problem**: Inconsistent or incorrect `shape_dist_traveled` values cause gtfstidy calculation errors
      - **Solution**: Remove the column entirely; gtfstidy will recalculate distances accurately using `-m` flag
      - **Benefit**: Ensures consistent distance measurements across all shapes
   - **Problem**: Duplicate or non-consecutive `shape_pt_sequence` values
      - **Solution**: Sort by `shape_id` and `shape_pt_sequence`, then re-index to ensure strictly increasing consecutive integers (0, 1, 2...) per shape

4. **Fix `stop_times.txt` - remove `shape_dist_traveled` and fix sequences**:
   - **Problem**: Inconsistent `shape_dist_traveled` values cause gtfstidy errors
      - **Solution**: Remove the column; gtfstidy will recalculate
   - **Problem**: Duplicate `stop_sequence` values within the same trip
      - **Solution**: Sort by `trip_id` and `stop_sequence`, then re-index to ensure consecutive sequences starting from 1

5. **Normalize hex colors and clean `routes.txt`**:
   - **Problem**: Invalid color formats (missing `#`, wrong length, non-hex characters) break visualization tools and gtfstidy validation
      - **Solution**: 
        - Convert to string, strip whitespace and remove `#` prefix
        - Pad with leading zeros if valid hex but short (e.g., "9966" → "009966")
        - Ensure exactly 6-digit hex format (e.g., "FF0000" for red)
        - Clear invalid non-hex values (set to empty string)
   - **Problem**: Invalid/non-standard fields like `route_sort_color` cause validation errors
      - **Solution**: Drop invalid fields from the dataframe
   - **Additional**: Ensure `route_sort_order` is properly formatted as integer

6. **Remove orphaned `shape_id` references in `trips.txt`**:
   - **Problem**: `shape_id` values in `trips.txt` that don't exist in `shapes.txt` cause referential integrity errors
      - **Solution**: Load valid `shape_id`s from `shapes.txt`, normalize IDs for comparison (handle float/int/string mismatches), then set orphaned `shape_id` values to empty string

7. **Fix `calendar.txt` formatting**:
   - Ensures proper CSV formatting and column name cleaning (strip whitespace)

8. **Remove single-day (one-off) services before frequency analysis**:
   - **Problem**: `service_id`s where `start_date == end_date` represent one-off service that can inflate weekly frequency estimates
   - **Solution**: Remove these `service_id`s from `calendar.txt` and also remove related rows from `calendar_dates.txt` (if present), `trips.txt`, and `stop_times.txt` to maintain referential integrity

9. **Zip the merged feed**:
   - gtfstidy accepts only `.zip` files as input (not directories)
   - Use Python's `zipfile` module to create `Merged_Feed.zip` containing all GTFS `.txt` files
   - Remove existing ZIP if present to ensure fresh creation
   - Use standard ZIP compression with `ZIP_DEFLATED` compression level 9
   - Do not use nested folders inside the archive (files should be at root level)

The preparation phase outputs a clean `Merged_Feed.zip` file ready for gtfstidy processing.

### 6.2 Modification
This step runs `gtfstidy` with **lowercase flags** to normalize the feed and fix validation errors. It produces an intermediate `Merged_Feed_modified.zip` file.

**Goal**: Ensure the feed is valid and consistent before aggressive optimization.

**Key Flags**:
- `--fix` — auto-fix common validation errors
- `-e` — drop invalid entries on parse (skip entities with errors)
- `-m` — remeasure shapes (recalculates `shape_dist_traveled` accurately)
- `-s` — sort/simplify structure (minimizes shapes by removing redundant points)
- `-c` — compact/compress where possible (minimizes calendar services)
- `-i` — renumber IDs (simplifies IDs to sequential integers)

**Process**:
1. **Run gtfstidy command**: Uses Python's `subprocess.run()` to execute gtfstidy with the modification flags
   - Input: `Merged_Feed.zip` (from **Step 6.1**)
   - Output: `Merged_Feed_modified.zip` (intermediate file)
   - Command format: `gtfstidy --fix -e -m -s -c -i -o "{output_zip}" "{input_zip}"`

2. **Capture output**: The script captures both stdout and stderr from gtfstidy to display:
   - Parsing progress and dropped entities (trips, stop times, stops, shapes, services, routes, agencies, transfers, etc.)
   - Shape remeasurement results
   - Shape minimization statistics (reduction in total points)
   - Service minimization results (`calendar_dates.txt` and `calendar.txt` entry reductions)
   - ID renumbering completion

3. **Generate detailed modification report**: After successful execution, the script compares before/after statistics:
   - **Stops**: Count comparison (typically unchanged)
   - **Routes**: Count comparison (typically unchanged)
   - **Trips**: Count comparison (may decrease if invalid trips are dropped)
   - **Shapes**: 
     - Unique shape count comparison
     - Total shape points comparison
     - Reduction percentage (e.g., 51.6% fewer points)
   - **Services**: Active service ID count from `calendar.txt` and `calendar_dates.txt`
   - **ID Changes**: Sample `agency_id` before/after to verify renumbering (e.g., 'acb' → '1')

**Expected Output**:
- `Merged_Feed_modified.zip` — normalized intermediate feed ready for optimization phase
- Console output showing parsing results, shape remeasurement, minimization statistics, and detailed modification report

**Example Output**:
```
Remeasuring shapes... done. (1801 shapes remeasured)
Minimizing shapes... done. (-3274611 shape points [-51.65%])
Minimizing services... done. (-3036 calendar_dates.txt entries [-68.42%], -3 calendar.txt entries [-0.03%])
Minimizing ids... done.
```

### 6.3 Optimization
This step runs `gtfstidy` with **uppercase flags** to aggressively optimize the feed. It produces the final `Merged_Feed_tidy/` directory.

**Goal**: Reduce file size, remove redundancy, and improve map visualization.

**Key Flags**:
- `--delete-orphans` — remove orphaned references (unused stops, shapes, routes, services)
- `--drop-errs` (or `-D`) — drop erroneous entities (trips with missing service_ids, invalid data)
- `-O` — general optimization pass (gtfstidy optimize)
- `-S` — simplify shapes (further reduce redundant shape points)
- `-R` — optimize/remove redundant routes (consolidate duplicate routes)
- `-C` — compress calendars/services (remove duplicate service patterns)
- `-T` — optimize trips (convert to frequencies where appropriate, remove redundant trips)
- `--red-stops-fuzzy` — fuzzy stop matching (merge stops with similar names/locations)
- `--snap-stops` — snap stops to shapes (move stops within 100m to nearest point on route line)
- `-W` — show warnings (display detailed information about deletions and changes)

**Stop deduplication note (`-P`)**:
- In the notebook, `-P` is commented out because it can remove many stops that are not truly redundant in some feeds.
- If you want to enable it, add `-P` back into the command below and validate the before/after stop counts.

**Process**:
1. **Run gtfstidy command**: Uses Python's `subprocess.run()` to execute gtfstidy with the optimization flags
   - Input: `Merged_Feed_modified.zip` (from **Step 6.2**)
   - Output: `Merged_Feed_tidy/` (final optimized feed directory)
   - Command format: `gtfstidy --delete-orphans --drop-errs -O -S -R -C -T --red-stops-fuzzy --snap-stops -W -o "{output_dir}" "{input_zip}"`

2. **Capture output**: The script captures both stdout and stderr from gtfstidy to display:
   - Parsing progress and dropped entities
   - Unreferenced entry removal (orphaned stops, shapes, services, routes)
   - Redundant stop removal (deduplication results)
   - Shape remeasurement
   - Stop snapping results (stops moved to route lines)
   - Redundant shape/route removal
   - Service duplicate removal
   - Frequency conversion and trip minimization

3. **Generate detailed optimization report**: After successful execution, the script compares before/after statistics:
   - **Stops (Deduplication & Snapping)**: 
     - Count comparison (typically decreases due to deduplication)
     - Locations moved (count of stops snapped to shapes)
   - **Routes (Redundant Route Removal)**: Count comparison (may decrease if duplicates are found)
   - **Trips (Optimization & Error Dropping)**: Count comparison (decreases due to error dropping and frequency conversion)
   - **Shapes (Simplification)**: 
     - Unique shape count comparison
     - Total shape points comparison
     - Reduction percentage
   - **Services (Calendar Compression)**: Active service ID count (typically decreases significantly due to duplicate removal)
   - **Deleted IDs (Orphans & Errors)**: Lists deleted IDs from each file (routes.txt, trips.txt, stops.txt, shapes.txt) with examples

**Expected Output**:
```python
MERGED_TIDY_FEED_DIR = Path(str(MERGED_FEED_DIR) + "_tidy")
 ```
- A single GTFS feed in `MERGED_TIDY_FEED_DIR` containing optimized data
- Console output showing optimization progress, deletions, and detailed optimization report

**Example Output**:
```
Removing unreferenced entries... done. (-80 stops [-2.20%], -41 shapes [-2.28%], -7179 services [-72.99%], -1 routes [-0.10%])
Removing redundant stops... done. (-107 stops [-3.01%])
Snapping stop points to shapes... done. (+256 stop points [+7.43%])
Removing service duplicates... done. (-2077 services [-78.20%])
Minimizing frequencies / stop times... done. (+98 frequencies, -232 trips [-5.61%])
```

### 6.4 Cleanup
The **optional cleanup cell** deletes intermediate artifacts to save disk space.

**This will permanently delete**:
- `Merged_Feed/` (the merged directory)
- `Merged_Feed.zip` (input ZIP for gtfstidy)
- `Merged_Feed_modified.zip` (intermediate output ZIP)
- `Unique_Feeds/` (split feeds directory)

---

## Step 7: Calculate Weekly Trip Frequency

### Purpose
Calculate how many trips each route shape runs per week. It is essential for:

- **Service analysis**: Understanding transit coverage and frequency patterns across agencies
- **Planning decisions**: Identifying high-frequency corridors vs. infrequent routes

The calculation accounts for multiple trips sharing the same shape and different service schedules (weekday vs. weekend services).

### Methodology

The process uses two main functions:

#### **`load_gtfs(gtfs_path)` Function**
- Loads required GTFS files with encoding fallback to handle different file encodings:
   - **Input**: Path to GTFS feed directory (e.g., `Merged_Feed_tidy/`)
   - **Files loaded**: 
      - `routes.txt` - Links trips to agencies via `route_id` and `agency_id`
      - `trips.txt` - Contains `trip_id`, `route_id`, `service_id`, and `shape_id`
      - `calendar.txt` - Defines which days of the week each service operates
      - `agency.txt` - Provides agency names for final output
   - **Encoding handling**: Tries multiple encodings (`utf-8-sig`, `utf-16`, `utf-8`) to handle files with different character encodings

#### **`calculate_weekly_trips_per_shape(routes, trips, calendar, agency)` Function**

1. **Filter Trips with Shape IDs**:
   - Uses **`.dropna(subset=['shape_id'])`** to only include trips that have a `shape_id`
   - Excludes trips without shape geometry

2. **Merge Trips with Routes**:
   - Merges trips with routes using **`.merge()`** on `route_id` to get `agency_id` for each trip
   - Uses `how='left'` to preserve all trips

3. **Calculate Weekly Days**:
   - For each service_id in calendar, sums the day columns using **`.sum(axis=1)`**:
   - `weekly_days = monday + tuesday + wednesday + thursday + friday + saturday + sunday`
   - Creates a new column `weekly_days` in the calendar dataframe

4. **Merge Trips with Calendar**:
   - Merges trips with calendar using **`.merge()`** on `service_id` to get `weekly_days` for each trip
   - Each trip now has its service's weekly active days

5. **Calculate Weekly Trips per Shape**:
   - Each trip contributes its service's `weekly_days` to its corresponding `shape_id`
   - Creates a `contribution` column equal to `weekly_days`
   - Groups by `shape_id` and sums contributions using **`.groupby('shape_id')['contribution'].sum().reset_index()`** to get total weekly trips per shape
   
   **Examples:**
   
   - **Example 1 - Simple case**: 
     - Shape "A" has 3 trips: Trip1 (Mon-Fri = 5 days), Trip2 (Daily = 7 days), Trip3 (Mon-Fri = 5 days)
     - Total weekly trips = 5 + 7 + 5 = **17 weekly trips**
   
   - **Example 2 - Bidirectional route**:
     - Shape "Route_101_Outbound" has 2 trips: Trip_Morning (Mon-Fri = 5 days), Trip_Evening (Mon-Fri = 5 days)
     - Shape "Route_101_Inbound" has 2 trips: Trip_Return_Morning (Mon-Fri = 5 days), Trip_Return_Evening (Mon-Fri = 5 days)
     - Outbound total = 5 + 5 = **10 weekly trips**
     - Inbound total = 5 + 5 = **10 weekly trips**
     - *Note: Each direction is counted separately*
   
   - **Example 3 - Mixed service patterns**:
     - Shape "B" has trips with different service patterns:
       - Trip1: Weekday service (Mon-Fri = 5 days)
       - Trip2: Weekend service (Sat-Sun = 2 days)
       - Trip3: Daily service (Mon-Sun = 7 days)
     - Total weekly trips = 5 + 2 + 7 = **14 weekly trips**
   
   **Why this metric matters:**
   - High frequency (e.g., 50+ weekly trips) indicates major transit corridors
   - Low frequency (e.g., 2-5 weekly trips) may indicate limited or seasonal service
   - Useful for prioritizing infrastructure investments and service improvements

6. **Collect Associated IDs**:
   - For each shape, collects unique values using **`.groupby('shape_id').unique()`**:
     - `agency_id`(s) - converted to single value (since each shape has unique agency)
     - `service_id`(s) - list of all service IDs used by trips on this shape
     - `route_id`(s) - list of all route IDs using this shape
     - `trip_id`(s) - list of all trip IDs using this shape
   - Merges all collected IDs into the weekly_trips dataframe using **`.merge()`**

7. **Add Agency Names**:
   - Merges with agency dataframe using **`.merge()`** on `agency_id` to get `agency_name`
   - Extracts single `agency_id` from the list (since each shape belongs to one agency)

8. **Export Results**:
   - Saves to `weekly_trips_per_shape_{RUN_TAG}.csv` in `OUTPUT_DIR` (configured via `WEEKLY_TRIPS_CSV`)
   - Uses **`.to_csv()`** with `index=False` to avoid saving row indices
   - **Columns**: `shape_id`, `weekly_trips`, `agency_id`, `agency_name`, `service_ids`, `route_ids`, `trip_ids`

### Key Functions

- **`pd.read_csv()`**: Reads CSV files with encoding fallback
- **`.dropna()`**: Removes rows with missing values in specified columns
- **`.merge()`**: Joins dataframes on common keys
- **`.sum(axis=1)`**: Sums values across columns (row-wise)
- **`.groupby().sum()`**: Groups data and calculates sums
- **`.groupby().unique()`**: Groups data and collects unique values
- **`.reset_index()`**: Converts groupby result back to dataframe
- **`.to_csv()`**: Exports dataframe to CSV file

### Expected Output

- **Console output**: Shows processing status and preview of results dataframe
- **CSV file**: `weekly_trips_per_shape_{RUN_TAG}.csv` containing one row per unique shape_id with:
  - `shape_id`: Unique shape identifier
  - `weekly_trips`: Total weekly trip count (sum of all trip contributions)
  - `agency_id`: Agency identifier
  - `agency_name`: Agency name
  - `service_ids`: List of service IDs (as array/string representation)
  - `route_ids`: List of route IDs
  - `trip_ids`: List of trip IDs


---

## Step 8: Build Custom GeoJSON from GTFS

### Purpose

This step creates a standards-compliant **GeoJSON FeatureCollection** from the merged and tidied GTFS feed (from **Step 6**). This approach provides complete control over which properties are included in the GeoJSON output, making it suitable for use in GIS tools (QGIS, ArcGIS Pro), web mapping libraries (Leaflet, Mapbox GL JS), or the Folium visualization workflow in **Step 9**.

The GeoJSON output contains two types of features:
- **Route features**: LineString geometries representing route paths with comprehensive route metadata
- **Stop features**: Point geometries representing transit stops with route relationship information

### 8.1 Load GTFS Data and Supporting Files

This step loads all required GTFS files from the merged and tidied feed directory (`MERGED_TIDY_FEED_DIR`, typically named `Merged_Feed_tidy`), along with supporting data files:

- **Core GTFS files**: Reads `agency.txt`, `routes.txt`, `trips.txt`, `stops.txt`, `shapes.txt`, and `stop_times.txt` from the merged feed using **`pd.read_csv()`**
- **Weekly trips data**: Loads `WEEKLY_TRIPS_CSV` (generated in **Step 7**; filename follows `weekly_trips_per_shape_{RUN_TAG}.csv`)
- **Source metadata**: Reads the `Agency_List.csv` file using **`pd.read_csv()`** with `encoding='ISO-8859-1'` to handle special characters

The process prints summary statistics using **`len()`** to count rows and **`.columns.tolist()`** to list column names. This helps verify that all required data is present before building the GeoJSON features.

### Key Functions

- **`pd.read_csv()`**: Reads CSV files into pandas DataFrames
- **`os.path.join()`**: Constructs file paths safely across operating systems
- **`len()`**: Counts the number of rows in DataFrames
- **`.columns.tolist()`**: Extracts column names as a list

### 8.2 Build Route Features as LineString Geometry

This step creates GeoJSON LineString features for each route-shape combination in the GTFS feed:

1. **Group and sort shape coordinates**: 
   - Sorts `shapes.txt` using **`.sort_values(['shape_id', 'shape_pt_sequence'])`** to ensure proper coordinate ordering
   - Groups coordinates by `shape_id` using **`.groupby('shape_id').apply()`** to create arrays of `[longitude, latitude]` pairs (GeoJSON requires lon/lat order)
   - Converts to dictionary using **`.to_dict()`** for efficient lookup

2. **Extract route-shape relationships**:
   - Gets unique combinations using **`.drop_duplicates()`** on `route_id` and `shape_id` from `trips.txt` (handles routes with multiple shapes, e.g., different AM/PM service patterns)
   - Merges with `routes.txt` and `agency.txt` using **`.merge()`** with `on='route_id'` and `on='agency_id'` to enrich with route and agency metadata

3. **Build route feature properties**:
   - **Core fields**: Uses **`row.get()`** to safely retrieve `agency_name`, `agency_id`, `route_id`, `route_short_name`, `route_long_name`
   - **Source information**: Filters source DataFrame using boolean indexing **`source_df[source_df['agency_name'] == ...]`** and retrieves values with **`.iloc[0].get()`**
   - **Optional styling**: The notebook may read/normalize `route_color` and `route_text_color` during preprocessing, but the GeoJSON export step removes these keys by default (defensive cleanup) so they are not written to the final `.geojson`.
   - **Weekly trips**: Filters weekly trips DataFrame using **`weekly_trips_df[weekly_trips_df['shape_id'] == shape_id]`** and extracts value with **`.iloc[0]`**
   - **Stop information**: Filters trips using boolean indexing **`trips[(trips['route_id'] == route_id) & (trips['shape_id'] == shape_id)]`**, then sorts stop_times with **`.sort_values('stop_sequence')`** and converts to list with **`.tolist()`**

4. **Create GeoJSON features**: Each feature contains a LineString geometry with the shape coordinates and a properties dictionary with all route metadata.

The output is a list of route features that will be combined with stop features to create the final GeoJSON FeatureCollection.

### 8.3 Build Stop Features as Point Geometry

This step creates GeoJSON Point features for each stop in the GTFS feed, enriched with information about which routes serve each stop:

1. **Link stops to routes**:
   - Merges `stop_times.txt` with `trips.txt` using **`.merge()`** with `on='trip_id'` to connect stops to routes
   - Further merges with `routes.txt` and `agency.txt` using **`.merge()`** to get route and agency details
   - Groups by `stop_id` using **`defaultdict(list)`** to collect all unique routes serving each stop

2. **Build stop-route relationships**:
   - For each stop, creates a list of route information dictionaries using **`row.get()`** to safely retrieve:
       - `route_id`, `agency_id`, `route_short_name`, `route_long_name`
       - > Note: `route_color` / `route_text_color` are not written to the final GeoJSON (export cleanup), and `route_type` is not included in the notebook’s per-stop route dictionaries.
   - Only adds unique routes using **`if route_info not in stop_route_info[stop_id]`** to avoid duplicates

3. **Create stop feature properties**:
   - **Core fields**: Uses **`stop.get()`** to retrieve `stop_id`, `stop_name`, `agency_name` (from first route serving the stop)
   - **Route information**: Gets routes using **`stop_route_info.get(stop_id, [])`** and counts with **`len()`** for `route_count` (useful for identifying major transfer hubs)
   - **Source information**: Filters source DataFrame using **`source_df[source_df['agency_id'] == agency_id]`** and retrieves with **`.iloc[0].get()`**
   - **Optional GTFS fields**: Checks for values using **`pd.notna()`** before including `stop_code`, `stop_desc`, `stop_url`, `stop_timezone`, and `wheelchair_boarding`.
     - > Note: `zone_id` is removed from the final GeoJSON by the export cleanup.

4. **Create GeoJSON features**: Each feature contains a Point geometry with `[longitude, latitude]` coordinates and a properties dictionary with all stop metadata.

The output is a list of stop features that will be combined with route features to create the final GeoJSON FeatureCollection.

### 8.4 Combine Features and Export GeoJSON

This step combines all route and stop features into a single GeoJSON FeatureCollection and saves it to a file:

1. **Create FeatureCollection**:
   - Combines the `route_features` list (LineString features) with the `stop_features` list (Point features) using list concatenation **`route_features + stop_features`**
   - Wraps them in a GeoJSON FeatureCollection structure with `type: "FeatureCollection"` and a `features` array

2. **Export to file**:
   - Saves the combined GeoJSON using **`json.dump()`** with `indent=2` for formatting
   - Opens file with **`open(output_file, 'w', encoding='utf-8')`** to handle special characters in agency names, route names, and stop names
   - Uses **`with`** statement for proper file handling

3. **Summary output**:
   - Prints the total number of features using **`len(geojson['features'])`** and **`len(route_features)`**, **`len(stop_features)`** for breakdown

### Key Functions

- **`json.dump()`**: Writes Python dictionaries to JSON files with formatting
- **`open()`**: Opens file handles for reading/writing with encoding support
- **`len()`**: Counts elements in lists/dictionaries

### 8.5 Preview Sample Features

This step provides a quick preview of the generated GeoJSON features to verify the structure and content:

1. **Sample route feature**:
   - Accesses first feature using **`geojson['features'][0]`** and serializes with **`json.dumps()`** with `indent=2`
   - Truncates output using string slicing **`[:500]`** for readability
   - Shows the complete structure including geometry type (LineString), coordinates, and all properties
   - Useful for verifying that route metadata (agency name, route names, weekly trips, source information) is correctly included

2. **Sample stop feature**:
   - Calculates stop feature index using **`len(route_features)`** to find first stop after all routes
   - Accesses feature using **`geojson['features'][stop_feature_index]`** and displays with **`json.dumps()`**
   - Shows the Point geometry with coordinates and stop properties
   - Verifies that route relationships, agency information, and optional fields are properly structured

**Sample console output (truncated)**:

```
Sample Route Feature:
{
  "type": "Feature",
  "properties": {
    "agency_name": "Oregon POINT",
    "agency_id": "19",
    "route_id": 201,
    "route_short_name": "NorthWest",
    "route_long_name": "Portland-Astoria",
    "weekly_trips": 28,
    "stops": ["935","562","2343","1242","2344","962","3625","1835","1189","2051","329"],
    "stop_count": 11
    ...
  },
  "geometry": {
    "type": "LineString",
    "coordinates": [
      [-122.67736, 45.528996],
      [-122.677345, 45.52901],
      ...
    ]
  }
}

Sample Stop Feature:
{
  "type": "Feature",
  "properties": {
    "stop_id": "935",
    "stop_name": "Portland Amtrak Station",
    "agency_name": "Oregon POINT",
    "routes": [
      {
        "route_id": 201,
        "agency_id": 19,
        "route_short_name": "NorthWest",
        "route_long_name": "Portland-Astoria",
        ...
      }
    ],
  },
  "geometry": {
    "type": "Point",
    "coordinates": [-122.67736, 45.528996]
  }
}
```

### Key Functions

- **`json.dumps()`**: Converts Python dictionaries to JSON-formatted strings
- **`len()`**: Counts elements to calculate array indices
- **String slicing `[:500]`**: Truncates output for readability

This preview helps ensure the GeoJSON output is correctly formatted before using it in GIS tools or web mapping applications. The full structure can be inspected by opening the saved `.geojson` file in a text editor or JSON viewer.

### Expected Output

- **`Merged_Agencies_{RUN_TAG}.geojson`**: A standards-compliant GeoJSON FeatureCollection (written to `GEOJSON_OUTPUT`) containing:
  - All route features as LineString geometries with comprehensive route metadata
  - All stop features as Point geometries with route relationship information
  - Complete source attribution and weekly trip frequency data

---

## Step 9: Interactive Web Map

### Purpose

- This workflow creates a [**Python**](https://www.python.org/)-based **interactive web map** using Folium that displays all intercity bus routes, stops, and weekly trip frequencies from the GeoJSON file created in **Step 8**.  
- It provides a lightweight and open-source alternative to traditional **GIS** visualization tools for creating shareable web maps.

### Methodology

This workflow uses the GeoJSON file (`Merged_Agencies_{RUN_TAG}.geojson`) created in **Step 8** to generate an interactive web map. The GeoJSON already contains all route and stop features with their properties, so this step focuses on visualization.

#### **1. Prepare the Files**
- Define the following file paths in your script:  
   - `GEOJSON_FILE` → path to the GeoJSON file created in **Step 8** (e.g., `Merged_Agencies_{RUN_TAG}.geojson`)  
   - `OUTPUT_MAP` → name for the map output (e.g., `Intercity_bus_map_{RUN_TAG}.html`)


#### **2. Load GeoJSON Data**
- Load the GeoJSON file using Python's `json` module or `geopandas` if available
- Parse the FeatureCollection to extract route features (LineString) and stop features (Point)
- Verify that the GeoJSON contains the expected properties (agency_name, route_id, weekly_trips, etc.)

#### **3. Generate Interactive Map**

This step loads the GeoJSON file created in **Step 8** and creates an interactive Folium web map:

1. **Load GeoJSON file**:
   - Reads the `Merged_Agencies_{RUN_TAG}.geojson` file created in **Step 8**
   - Parses the FeatureCollection to extract route and stop features

2. **Initialize Folium map**:
   - Creates a map centered on the continental U.S. (lat: 39.83, lon: -98.58) with zoom level 5
   - Uses OpenStreetMap basemap tiles (you can swap tiles if desired)

3. **Group features by agency**:
   - Organizes routes and stops by agency for layer control
   - Assigns each agency a unique hex color generated via `generate_distinct_colors()` (golden-angle hues converted from HSV → hex)

4. **Add route layers**:
   - For each agency, creates a `FeatureGroup` layer (initially hidden; users toggle visibility via layer control)
   - Plots routes as colored `PolyLine` objects with pop-ups showing:
     - Route name, route ID, agency name
     - Weekly trips (from GeoJSON properties)
     - Distance information if available
   - Line **thickness encodes weekly frequency** using `get_line_weight()`:
     - `< 6 trips` → weight 2
     - `6–10 trips` → weight 4
     - `11–20 trips` → weight 6
     - `21–35 trips` → weight 8
     - `36+ trips` → weight 10
   - A fixed legend overlay explains these bands so the map viewer understands the symbology

5. **Add stop markers** (matches the notebook):
   - Adds stops as `folium.Marker` objects using a custom `DivIcon` (hollow circle with a bus icon), colored by agency
   - Pop-ups show stop name, stop ID, and routes serving the stop
   - Optional marker clustering is controlled by `STOP_USE_MARKER_CLUSTER` / `STOP_CLUSTER_DISABLE_AT_ZOOM`

6. **Add layer control**:
   - Adds a `LayerControl` (expanded by default) to toggle agencies on and off
   - Allows users to focus on specific agencies or view all routes together

7. **Save map**:
    - Exports the final interactive map as `Intercity_bus_map_{RUN_TAG}.html`
   - The HTML file is self-contained and can be opened in any web browser or shared online
   - Console output confirms:
       - `Map saved as Intercity_bus_map_{RUN_TAG}.html`
     - `Routes are colored by agency and sized by weekly trip frequency`
     - `Stops use custom hollow circle with bus icon (cluster=False)`

**Advanced interactions included in the Folium template**:
- Custom popup formatter (`format_properties_html`) that turns long arrays/dicts into readable bullet lists with truncation (“… and N more”)
- Optional per-agency stop clustering controlled by `STOP_USE_MARKER_CLUSTER` / `STOP_CLUSTER_DISABLE_AT_ZOOM`
- Hollow circle stop icons styled via `DivIcon` so every stop matches the agency color scheme
- Overlap navigator JavaScript (`overlap_handler_js`) that:
  - Detects features within 100 m of a click (routes via point-to-segment distance, stops via Haversine distance)
  - Lets users cycle through overlapping features with Prev/Next buttons inside the popup
  - Displays which agency the currently viewed feature belongs to


### Expected Output:
   - **`Intercity_bus_map_{RUN_TAG}.html`**
   - An interactive web map that:  
      - Displays routes as colored lines and stops as clickable points  
      - Groups routes by agency, each with a unique color (initially hidden; toggle via layer control)  
      - Allows toggling agencies using the layer control  
      - Shows route names, route IDs, agency names, and weekly trip counts in route pop-ups  
      - Shows stop names, stop IDs, and routes serving each stop in stop pop-ups  
     - Uses data from the GeoJSON file created in **Step 8** (`Merged_Agencies_{RUN_TAG}.geojson`)

---

## Notes and Best Practices

### Data Quality
- Always validate before and after cleaning
- Keep backups of original data
- Document any manual decisions (e.g., which routes to remove)

### Performance
- Large datasets may take time to process
- Consider processing agencies in batches if memory is limited

### Customization
- Modify distance thresholds in **Step 4** based on your definition of "intercity"
- Adjust validation rules if your data has special requirements
- Add custom validation checks as needed

### Integration with GIS
- The merged feed can be imported into ArcGIS Pro using GTFS tools
- The weekly frequency CSV can be joined to shape features for thematic mapping
- Use shape geometries from `shapes.txt` to create polyline features

---

## Troubleshooting

### Common Issues

**Missing Files**:
- Ensure all required GTFS files are present
- Some agencies may not provide optional files like `shapes.txt`

**Encoding Errors**:
- GTFS files should be UTF-8 encoded
- Use `encoding='utf-8-sig'` when reading CSVs to handle BOM

**Memory Issues**:
- Process agencies individually if dataset is too large
- Use chunking for very large files

**Validation Failures**:
- Review specific error messages
- Fix issues before proceeding to merge
- Some agencies may have non-standard GTFS implementations

---