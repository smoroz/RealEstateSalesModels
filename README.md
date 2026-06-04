
# Connecticut Real Estate Data Pipeline

A data pipeline that consolidates 22 years of Connecticut real estate sales (2001–2022), enriches each record with a ZIP code, and outputs a clean, analysis-ready dataset visualized in Power BI.

---

## Overview

The raw data lacked consistent geographic identifiers — many records had no ZIP code. This pipeline resolves that through a three-stage enrichment strategy:

1. **Street-name fuzzy match** against OpenAddresses point data
2. **Cross-year address lookup** to fill gaps using ZIP codes found in other years
3. **Azure Maps geocoding** as a final fallback for remaining missing records

The result is ~1.1M property sale records with ZIP codes attached, enabling geographic analysis across Connecticut.

---

## Data Sources

| Source | Description |
|---|---|
| [CT Open Data Portal](https://data.ct.gov/) | Real estate sales CSVs by year (2001–2022) |
| [OpenAddresses](https://openaddresses.io/) | Point-level address data (`source.geojson.gz`) |
| [Census TIGER/Line ZCTA 2020](https://www.census.gov/geographies/mapping-files/time-series/geo/tiger-line-file.html) | ZIP Code Tabulation Area shapefiles for spatial joins |
| [Azure Maps API](https://azure.microsoft.com/en-us/products/azure-maps) | Geocoding fallback for unresolved addresses |

---

## Pipeline Steps

### 1. Load & Clean OpenAddresses
- Decompress `source.geojson.gz` into a GeoDataFrame
- Normalize street names: lowercase, strip digits, expand abbreviations (e.g. `ln` → `lane`, `pkwy` → `parkway`)
- Deduplicate on cleaned street name

### 2. Load & Clean Sales Data
- Parse each year's CSV, standardize addresses with the same normalization logic
- Merge with OpenAddresses on cleaned street name to recover ZIP codes and coordinates

### 3. Spatial Join with Census ZCTAs
- Reproject points to a shared CRS and join against Census ZCTA polygons
- Assigns `zip_code` from polygon containment where point coordinates exist

### 4. Cross-Year Lookup
- Build an address → ZIP dictionary from all records that have a ZIP code
- Backfill missing rows using matches from other years (~15K rows resolved)

### 5. Azure Maps Geocoding Fallback
- Construct full address strings for remaining ~14.9K unresolved rows
- Geocode via Azure Maps REST API to recover final ZIP codes and coordinates

### 6. Export
- Separate cleaned CSVs per list year: `Updated_Real_Estate_{year}.csv`

---

## Output Schema

| Column | Description |
|---|---|
| `Serial Number` | Unique sale identifier |
| `List Year` | Year the property was listed |
| `Date Recorded` | Sale recording date |
| `Town` | Connecticut town name |
| `Address` | Original address string |
| `Assessed Value` | Assessor's estimated value |
| `Sale Amount` | Actual sale price |
| `Sales Ratio` | Assessed / Sale ratio |
| `Property Type` | e.g. Single Family, Condo |
| `Residential Type` | Subtype classification |
| `zip_code` | Final enriched ZIP code |
| `geometry` | Point coordinates (lon, lat) |

---

## Power BI Dashboard

Three regional views built on the cleaned output, each filtered by ZIP code geography:

**All of Connecticut 2009**
<img width="1307" height="740" alt="screenshot1" src="https://github.com/user-attachments/assets/4e7b69ea-42b1-4701-a731-b9fc9fb6b522" />
- Avg Sale Amount: $326K · 32.68K records · Sales Ratio: 159.59%

**Northern/Central CT (Hartford region)**
<img width="1314" height="742" alt="screenshot2" src="https://github.com/user-attachments/assets/be850c35-f87e-4e68-aa3f-3677387019a3" />
- Avg Sale Amount: $445K · 12.05K records · Sales Ratio: 106.15%

**Coastal CT (New Haven / Shoreline)**
- Avg Sale Amount: $344K · 4.71K records · Sales Ratio: 83.92%
<img width="1315" height="739" alt="screenshot3" src="https://github.com/user-attachments/assets/c785eccb-56b7-4014-81ff-9fb8f78dbb8c" />

All views show assessed value vs. sale amount trends from 2001–2022 with a linear trend overlay.

---

## Requirements

```
pandas
geopandas
shapely
requests
```

Install with:
```bash
pip install pandas geopandas shapely requests
```

---
