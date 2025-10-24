# L2-Batch-Chlorophll-Extractor-In-Situ-Data-Matcher
This Python script (run from a Jupyter Notebook) automates the extraction of satellite data from a large batch of SeaDAS L2 NetCDF (```.nc```) files. It is designed to find the closest satellite pixel to a specific in-situ site, extract chlorophyll-a (```chlor_a```) and quality flag data from various window sizes (e.g., 3x3, 5x5), and then match those extractions against a time-series of in-situ measurements.

## Overview
The main workflow is as follows:

1. **Scan L2 Files**: Recursively search a directory for all ```.nc``` files.

2. **Extract Data**: For each file, find the pixel nearest to a target latitude/longitude. Extract the ```chlor_a``` value, robustly find the acquisition timestamp, and calculate 3x3/5x5 statistics (mean, median, valid pixel count) around that pixel.

3. **Handle Masking**: If the center pixel is masked (e.g., cloud, land), search in an expanding radius (up to 50 pixels) to find the nearest valid ```chlor_a``` pixel.

4. **Save Satellite Summary**: Write all extracted satellite data to a single ```SUMMARY_CSV``` file.

5. **Match to In-Situ**: Load the ```SUMMARY_CSV``` and an ```INSITU_CSV``` (e.g., AERONET-OC data). Perform a nearest-neighbor temporal join, matching satellite files to in-situ measurements that fall within a ±HOURS_TOL window.

6. **Save Final Report**: Save the final, merged table (satellite data + matched in-situ data) to a new ```OUT_MATCHED_CSV``` file.

## Workflow
This script splits the workflow into two main steps (notebook cells) that you run separately.

1. **Scan L2 Files** → ```SUMMARY_CSV``` (**Cell 2**) Reads all ```.nc``` files from ```MAIN_DIR```. For each file, it finds the pixel nearest to ```TARGET_LAT/TARGET_LON``` and extracts ```chlor_a``` data, window statistics, and acquisition time. It saves this raw satellite-only information to ```SUMMARY_CSV```.

2. **Match In-Situ** → ```OUT_MATCHED_CSV``` (**Cell 4**) Reads the ```SUMMARY_CSV``` (from Cell 2) and the ```INSITU_CSV```. It then finds the closest in-situ measurement (e.g., ```aoc_chl_a```) for each satellite file, only if it is within the ```±HOURS_TOL``` window. It saves this final merged table to ```OUT_MATCHED_CSV```.

## Cell 1: Configuration
This cell defines all the key parameters for the entire workflow.
### Key settings
```
# ============== EDIT THESE IF NEEDED =============_
SITE        = "Lucinda"
TARGET_LAT  = -18.5198          # site latitude
TARGET_LON  = 146.386         # site longitude
HOURS_TOL   = 3                  # ± hours for the in-situ time-window match

# Base folder that contains your .nc files and CSVs
BASE_DIR    = Path("Y:/") / "Mingyue" / SITE

# Folder with many *.nc (SeaDAS L2) files
MAIN_DIR    = BASE_DIR / "l2gen_product"

# Output summary CSV from Cell 2
SUMMARY_CSV = BASE_DIR / f"{SITE}_l2_summary.csv"

# In-situ CSV (must contain columns like Date, Time, Chlorophyll-a)
INSITU_CSV  = BASE_DIR / f"{SITE}_in_situ_data.csv"

# Pattern to find NetCDF files
NC_GLOB_PATTERN = "*.nc"

# Whether to recurse into subfolders of MAIN_DIR
RECURSIVE_SEARCH = True
# ==================================================
```

## Cell 2: Scan L2 Files → Summary CSV
### What it does
- Recursively searches ```MAIN_DIR``` for files matching ```NC_GLOB_PATTERN```.

- Robustly extracts acquisition datetimes from file metadata or filename.

- Finds the (row, col) index of the pixel closest to ```TARGET_LAT/TARGET_LON```.

- Extracts the center ```chlor_a``` value and L2 flags.

- Extracts mean, median, and valid pixel counts for 3x3 and 5x5 windows.

- If the center pixel is masked, it finds the nearest valid pixel within a 50-pixel radius and records its value and distance.

- Saves all 58 rows of data to ```Lucinda_l2_summary.csv```.
- 
## Cell 4: Match Summary → Final Matched CSV
### What it does:
- Reads the ```SUMMARY_CSV ```(from Cell 2) and the ```INSITU_CSV``` (AERONET-OC data).

- Parses timestamps from both files into timezone-aware UTC objects.

- Performs a ```pandas.merge_asof``` to find the nearest in-situ record for each satellite file.

- Crucially, it only keeps matches where the time difference is within the ```tolerance``` (e.g., ```pd.Timedelta(hours=HOURS_TOL)```).

- Appends the matched in-situ data (e.g.,``` matched_aoc_chl_a```, ```matched_aoc_rrs443```) and the time difference (```match_dt_hours```) to the satellite data rows.

- Saves this new, comprehensive table to ```OUT_MATCHED_CSV```.
  ### Keysettings
```
  # OUTPUT (will NOT overwrite inputs)
OUT_MATCHED_CSV = BASE_DIR / f"{SITE}_l2_summary_2.csv"

  # Optional AERONET Rrs columns to carry through if present
  AOC_RRS_COLS = [
      "aoc_rrs412","aoc_rrs443","aoc_rrs490","aoc_rrs510",
      "aoc_rrs560","aoc_rrs665","aoc_rrs681"
]
  ```
