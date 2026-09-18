# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

The ROLI Map Generator is a Streamlit web application that creates interactive choropleth maps using World Justice Project's Rule of Law Index (ROLI) scores. The app allows users to visualize ROLI data across different countries, regions, and time periods with extensive customization options.

## Development Commands

### Running the Application

**Local development:**
```bash
streamlit run app.py
```

**Dev Container (automatically runs on attach):**
```bash
streamlit run app.py --server.enableCORS false --server.enableXsrfProtection false
```

### Package Management

This project uses `uv` for dependency management with Python 3.13+.

**Install dependencies:**
```bash
uv pip install -r requirements.txt
```

**Update requirements after modifying pyproject.toml:**
```bash
uv pip compile pyproject.toml -o requirements.txt
```

## Architecture

### Application Structure

**Main Application (`app.py`):**
- Single-file Streamlit application (~820 lines)
- Password-protected using Streamlit secrets
- Four-step user workflow: extension selection → data selection → customization → visualization
- Generates three outputs: choropleth map, data table, and bar chart

**Utility Modules (`src/utils/`):**
- `passcheck.py`: Password authentication via Streamlit session state
- `data_adds.py`: Contains `variable_labels` (ROLI variable descriptions) and `bbox_coords` (regional bounding boxes)
- `boundaries_cleaning.py`: Geospatial boundary data preprocessing
- `boundary_simplification.py`: GeoJSON simplification utilities
- `data-join.py`: Data merging utilities

### Data Flow

1. **Data Loading** (`load_data()` at line 26-34):
   - Cached with `@st.cache_data`
   - Loads `data4app.geojson` (country boundaries) into GeoPandas
   - Loads `ROLI_data.xlsx` (ROLI scores) into Pandas
   - Returns dictionary: `{"boundaries": GeoDataFrame, "roli": DataFrame}`

2. **Map Extension Logic** (lines 63-236):
   - World map: Uses all boundaries as-is
   - Regional map: Filters by WJP regions or UN regions using `bbox_coords`
   - Custom map: User defines lat/lon bounding box
   - Opacity feature highlights selected countries (alpha 1.0 vs 0.2)

3. **Data Processing** (lines 546-585):
   - Filters ROLI data by selected year
   - Optional delta mode: calculates percentage change between two years using `pct_change()`
   - Merges ROLI scores with boundary geometries on ISO3 codes (`WB_A3` ↔ `code`)

4. **Visualization Pipeline** (lines 606-674):
   - For non-world maps: masks boundaries using bounding box intersection
   - Applies Miller Cylindrical Projection (ESRI:54003) for regional/custom maps
   - Creates custom colormaps using `matplotlib.colors.LinearSegmentedColormap` or `ListedColormap`
   - Renders map with GeoPandas `.plot()` method

### Key Data Files

- `Data/data4app.geojson`: Simplified world boundaries (23MB)
- `Data/ROLI_data.xlsx`: Rule of Law Index scores by country/year
- `Data/Territories_EC.xlsx`, `Data/Territories_WB.xlsx`: Territory mappings (modified, tracked in git)
- `.streamlit/config.toml`: Streamlit theme configuration

## Important Implementation Details

### Regional Classifications

Two classification systems are supported:
1. **WJP Regions** (7 regions): Used by World Justice Project in Insights Reports
2. **UN Regions** (18+ subregions): Based on UN SDG regional classifications

Regional bounding boxes are defined in `src/utils/data_adds.py` (`bbox_coords` DataFrame).

### Coordinate Reference Systems

- **World maps**: Default CRS (WGS84/EPSG:4326)
- **Regional/Custom maps**: Miller Cylindrical Projection (ESRI:54003) for better area representation

### Delta (Percentage Change) Mode

When enabled (lines 358-402):
- Calculates `pct_change()` between base year and target year
- Bins values into categorical ranges (2, 4, or 6 categories)
- Uses discrete colormap (`ListedColormap`) instead of continuous gradient
- Default breaks at ±2.05% and ±4.05%

### Color Mapping

- **Continuous mode**: Uses `LinearSegmentedColormap` with 2-7 color breaks
- **Delta mode**: Uses `ListedColormap` with fixed categories
- Colors converted to hex using `colors.rgb2hex()` for table export
- Missing values rendered in `#EBEBEB` gray

### Password Protection

App requires password authentication via `check_password()` function. Password must be stored in Streamlit secrets (`st.secrets["password"]`). All app logic is wrapped in `if check_password():` block.

## Data Schema Expectations

### ROLI Data (Excel)
Required columns:
- `country`: Country name (string)
- `code`: ISO3 country code (string)
- `year`: Year as string (e.g., "2024", "2012-2013")
- Variable columns: Numeric scores (0-1 range for ROLI data)

### Boundary Data (GeoJSON)
Required properties:
- `WB_A3`: ISO3 country code
- `REGION_WJP`: WJP region classification
- `REGION_UN`: UN region classification
- `SUBREGION`: UN subregion classification
- `geometry`: Polygon/MultiPolygon geometries

## Custom Data Upload

Users can upload custom Excel files to visualize their own data. Requirements:
- Must have columns: `COUNTRY`, `CODE`, `YEAR` (case-insensitive, auto-renamed)
- Additional columns treated as variables
- User must specify min/max expected values for colormap scaling
