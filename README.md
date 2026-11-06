# Chiba Prefecture Real Estate Transactions: Data Cleaning

Cleaning and wrangling of Japanese real estate transaction data for **Chiba Prefecture**, prepared for downstream analysis. The workflow is deliberately general-purpose: it is not tied to a particular prediction target or model, so the cleaned dataset can support different analytical projects.

This repository will grow over time. The cleaning notebook comes first, and further notebooks will build on its output.

## Repository contents

| File | Description |
|------|-------------|
| `01_mlit_cleaning.ipynb` | Cleaning and feature-engineering notebook |
| `12.csv` | Raw transaction data for Chiba Prefecture (MLIT prefecture code `12`) |
| `real_estate_cleaned.csv` | Cleaned dataset, produced by running the notebook |

## Data

- **Original source:** Real Estate Transaction Price Information (不動産取引価格情報), published by Japan's Ministry of Land, Infrastructure, Transport and Tourism (MLIT) through its [Real Estate Information Library](https://www.reinfolib.mlit.go.jp/) (不動産情報ライブラリ). MLIT lists this content as available through the library's search, CSV download, and API, with commercial use permitted. MLIT also describes the program on its [English information page](https://www.mlit.go.jp/en/totikensangyo/totikensangyo_fr5_000014.html).
- **Version used here:** the "Japan Real Estate Transaction Prices" dataset by nishiodens on [Kaggle](https://www.kaggle.com/datasets/nishiodens/japan-real-estate-transaction-prices). It is the MLIT download, processed by the author's [cleaning script](https://gist.github.com/nishio-dens/d66a52cb26a4fe0851ceb353c75ecdf7) into one CSV per prefecture with English column names.
- **Columns added by that script:** `MinTimeToNearestStation`, `MaxTimeToNearestStation`, `AreaIsGreaterFlag`, `FrontageIsGreaterFlag`, `TotalFloorAreaIsGreaterFlag`, `PrewarBuilding`, `Year`, and `Quarter`. The script also parses area, frontage, and building year into numbers. The cleaning in this repository starts from that output.
- **Period covered:** transactions start in 2005 (MLIT's published data begins in Q3 2005). This is an older snapshot from Kaggle, so it ends well before MLIT's current data, and analyses that need recent market conditions will require newer data from MLIT. The 1945 to 2020 range seen in the notebook is the `BuildingYear` field (when buildings were constructed, with 1945 used as a placeholder for prewar buildings), not the transaction dates.
- **Geography:** Chiba Prefecture only. The data is split into one CSV per prefecture (`01.csv` to `47.csv`), and Chiba is `12.csv`. Loading a single file avoids reading the roughly 3.9 million records of the nationwide dataset.

To use another prefecture, put its CSV in the same folder as the notebook and change the file name in the loading cell.

**License and attribution:** the Kaggle dataset is licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/), so reuse requires credit. Please credit the Kaggle author (nishiodens) for the processed dataset and MLIT as the original source, for example: "Created by processing data from the Real Estate Information Library (Ministry of Land, Infrastructure, Transport and Tourism)." MLIT's own terms for the library apply to the original data.

## How to run

1. Clone the repository and move into the folder:
   ```bash
   git clone https://github.com/a-disorbo/japan-real-estate-analysis.git
   cd japan-real-estate-analysis
   ```
2. Install the dependencies:
   ```bash
   pip install pandas numpy jupyter
   ```
3. Launch Jupyter from the project folder and run `01_mlit_cleaning.ipynb` from top to bottom:
   ```bash
   jupyter notebook
   ```

The notebook loads `12.csv` with a relative path, so run it from the folder that contains the CSV. It writes `real_estate_cleaned.csv` to the same folder.

## What the notebook does

Missing values are assessed in context rather than blindly replaced. The notebook checks whether a value is missing because it does not apply (for example, floor area for a land-only sale) or because it was not recorded, and handles each case accordingly.

1. **Inspection:** shape, data types, and null counts and percentages for every column.
2. **Station variables:** derives `AvgTimeToNearestStation` and `TimeRangeToNearestStation` from the minimum and maximum travel times, and adds a `HasStation` flag. Missing station data appears to reflect properties with no nearby station, such as areas outside city planning zones.
3. **Property area:** creates `AreaSizeCategory`, since `Area` is capped at 2,000 and 5,000 m² for large properties.
4. **Unit price:** confirms `UnitPrice` and `PricePerTsubo` are recorded only for residential land (land only).
5. **Land shape and frontage:** fills missing `LandShape` with `Unknown`, merges similar shape categories into `LandShapeReplaced`, and creates `FrontageReplaced` with missing values set to `-1`.
6. **Floor area:** creates a `HasBuilding` indicator from `Type` and sets `TotalFloorArea` to `0` where there is no building. Pre-owned condominiums have no floor area recorded in this dataset.
7. **Building year and age:** calculates `BuildingAge` from `Year` and `BuildingYear`. The few negative ages (pre-construction sales) are clipped to `0`, and properties without a building are set to `-1`. Prewar buildings carry a placeholder year of 1945.
8. **Structure:** keeps only the first listed structure category in `StructureCleaned`, and labels non-building properties `No Building` in `StructureCleanedReplaced`.
9. **Use and purpose:** simplifies `Use` to `UseSimple`. Missing `Purpose` values are filled by random sampling from the observed `Purpose` distribution within each `UseSimple` category (fixed seed, 42, so results are reproducible). The original `Purpose` column is preserved.
10. **Road, zoning, and other fields:** reviews `Direction`, `Classification`, `Breadth`, `CityPlanning`, `CoverageRatio`, `FloorAreaRatio`, `Renovation`, and `Remarks`. Missing `CityPlanning` for agricultural and forest land is filled in `CityPlanningCleaned`.
11. **Export:** saves the cleaned dataset as `real_estate_cleaned.csv`.

### Engineered and cleaned columns

| Column | Description |
|--------|-------------|
| `AvgTimeToNearestStation` | Average of the minimum and maximum travel time to the nearest station, in minutes |
| `TimeRangeToNearestStation` | Maximum minus minimum travel time, in minutes |
| `HasStation` | `1` if station travel-time data is recorded, `0` otherwise |
| `AreaSizeCategory` | Land-area size category derived from `Area` |
| `LandShapeReplaced` | Land shape with similar categories merged |
| `FrontageReplaced` | Frontage with missing values replaced by `-1` |
| `HasBuilding` | `1` if the property type implies a building, based on `Type` |
| `BuildingAge` | Transaction year minus building year, clipped at 0; `-1` when there is no building |
| `StructureCleaned` | First listed building-structure category |
| `StructureCleanedReplaced` | `StructureCleaned` with `No Building` for non-building properties |
| `UseSimple` | First listed property use |
| `PurposeImputed` | `Purpose` with missing values filled by sampling within `UseSimple` |
| `CityPlanningCleaned` | `CityPlanning` with agricultural and forest land labelled |

Original columns are kept alongside the engineered ones. Model-specific imputation, feature selection, and encoding are left to later notebooks.

## Notes

- Fields reported as thresholds (area, frontage, and floor area) are capped in the source data. The original `...IsGreaterFlag` columns are kept so the cap remains visible.
- `UnitPrice` and `PricePerTsubo` are only available for residential land (land only), so they are missing for most transactions.

## Author

**Background:** Anthony DiSorbo, Google Data Analytics Certificate (Advanced)  
**Location:** Chiba Prefecture, Japan
**LinkedIn** https://www.linkedin.com/in/adisorbo/