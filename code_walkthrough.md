# Code Walkthrough
## quickrun.qmd — ODOT Safety Analysis Pipeline + Shiny App
**Author: Levi Sayles | Xavier University | 2025–2026**

---

## Overview

`quickrun.qmd` is a single Quarto document that contains the complete analysis pipeline and the full R Shiny application in one file. It is organized into the following major sections:

1. Data loading
2. Spatial join (crashes → road segments)
3. Crash aggregation and rate computation
4. PCR pavement data preparation
5. Master dataset assembly
6. DiD analysis and visualizations
7. Investment gap analysis
8. Random Forest model
9. Shiny app (UI + server)

The file is designed to be run top-to-bottom in RStudio. The Shiny app at the bottom depends on objects created earlier in the script, so all preceding sections must be run first before launching the app.

---

## Requirements

### Working Directory
The script sets the working directory at the top:
```r
setwd("~/2025 - 2026 XU/project")
```
All file paths are relative to this directory. Update this path if running on a different machine.

### Required Files
The following raw data files must be present in the working directory:

| File | Description |
|---|---|
| `Road_Inventory.gdb` | ODOT road inventory geodatabase |
| `crashes/2020Crash.csv` | 2020 crash records |
| `crashes/2021Crash.csv` | 2021 crash records |
| `crashes/2022Crash.csv` | 2022 crash records |
| `crashes/2023Crash.csv` | 2023 crash records |
| `crashes/2024Crash.csv` | 2024 crash records |
| `All_Projects.csv` | ODOT safety-funded construction projects |
| `PCR_Local.gdb` | Local road pavement condition geodatabase |
| `PCR_State.gdb` | State road pavement condition geodatabase |

### Required R Packages
```r
install.packages(c("sf", "leaflet", "tidyverse", "fixest",
                   "randomForest", "shiny", "shinydashboard",
                   "plotly", "scales", "DT"))
```

---

## Section-by-Section Walkthrough

### Section 1 — Data Loading (Lines ~15–82)

Loads all four raw data sources into R:

- **Road inventory** is read as a spatial geodatabase layer using `st_read()`. It contains 328,709 road segments with geometry, milepost ranges, traffic counts, and road class.
- **Crash records** are read as five separate CSVs (one per year) and combined into a single dataframe using `rbind()`.
- **Construction projects** are read from `All_Projects.csv`.
- **PCR pavement data** is read from two separate geodatabases (local and state roads) and combined. A helper function `make_numeric_safe()` converts character columns to numeric while preserving identifier columns that should stay as text.

---

### Section 2 — Spatial Join: Crashes to Road Segments (Lines ~100–137)

This is the most technically demanding step. Crash records contain GPS coordinates but no road segment identifier. The join works in three steps:

**Step 1 — Convert crashes to spatial points**
```r
crashes_sf <- st_as_sf(crashes,
  coords = c("ODOT_LONGITUDE_NBR", "ODOT_LATITUDE_NBR"),
  crs = 4326)
```

**Step 2 — Align coordinate reference systems**
Both datasets must use the same projection. Crashes are transformed from WGS84 (EPSG:4326) to Web Mercator (EPSG:3857) to match the road inventory. Z and M dimensions are also dropped since they cause errors in GEOS spatial operations.

**Step 3 — Nearest-feature join**
```r
crash_segment <- st_join(crashes_sf, road_inventory,
                          join = st_nearest_feature)
```
Each of the ~910,000 crash points is matched to the single nearest road segment geometry. The result is a crash-level dataset with each crash assigned a `ROADWAY_INVENTORY_ID`.

---

### Section 3 — Crash Aggregation and Rate Computation (Lines ~140–234)

The crash-level data is collapsed to segment level and crash rates are computed.

**Aggregation:**
```r
segment_crash_summary <- crash_segment %>%
  st_drop_geometry() %>%
  group_by(ROADWAY_INVENTORY_ID) %>%
  summarise(
    total_crashes          = n(),
    fatal_crashes          = sum(CRASH_SEVERITY_CD == 1, na.rm = TRUE),
    serious_injury_crashes = sum(CRASH_SEVERITY_CD == 2, na.rm = TRUE),
    injury_crashes         = sum(CRASH_SEVERITY_CD %in% 1:4, na.rm = TRUE)
  )
```

**Crash rate computation:**
Two rate metrics are computed for each segment:
- `crashes_per_mile` = total crashes / segment length
- `crashes_per_MVMT` = total crashes / (ADT × 365 × segment length / 1,000,000)

The MVMT-based rate is the primary metric used throughout the analysis because it accounts for traffic exposure — a busy road is expected to have more crashes simply because more vehicles use it.

Sanity checks at the end of this section verify there are no infinite values (which would occur if a segment has zero VMT).

---

### Section 4 — PCR Data Preparation (Lines ~180–203)

The pavement condition data requires special handling because it comes from two separate sources (local and state roads) with slightly different column structures. Key steps:

- Column name alignment: PCR data uses `NLFID` as the route identifier while the road inventory uses `NLF_ID`. These are renamed to match.
- Deduplication: Some segments have multiple PCR inspections across years. Only the most recent inspection is kept per segment using `slice_max(order_by = PCR_YEAR)`.
- The PCR data is joined to the master road dataset on `NLF_ID` (route identifier), not on `ROADWAY_INVENTORY_ID` (segment identifier), because PCR records are recorded at the route level rather than the individual segment level.

---

### Section 5 — Master Dataset Assembly

The final segment-level dataset (`final_sf`) is the result of three sequential left joins:

```
road_inventory
  → LEFT JOIN crash counts      (on ROADWAY_INVENTORY_ID)
  → LEFT JOIN PCR data          (on NLF_ID)
```

Segments with no crashes receive a count of zero (via `replace_na()`). Segments with no PCR data retain NA pavement condition values.

---

### Section 6 — Panel Construction and DiD Analysis

The segment-level cross-sectional data is reshaped into a panel by expanding to one row per segment per year. Treatment flags are assigned:

- `treated_flag` = 1 if a segment's route (NLF_ID) overlaps with a safety-funded project using the CTL milepost range overlap filter
- `post_flag` = 1 if the observation year is after the project's completion year
- `did_term` = `treated_flag × post_flag` — the DiD interaction term

**Year 2022 is excluded** from the main analysis as a transition year (see report for rationale).

The DiD model is estimated using `fixest::feols()`:
```r
did_model <- feols(crash_rate_MVMT ~ did_term | ROADWAY_INVENTORY_ID + CRASH_YR,
                   data = panel_data,
                   cluster = ~ROADWAY_INVENTORY_ID)
```

Subgroup models are run separately for each `PRIMARY_WORK_CATEGORY` to produce the project-type breakdown in Figure 2.

---

### Section 7 — Investment Gap Analysis

Using the master panel, crash rates and funding allocation are summarized by functional road class (`FUNCTION_CLASS_CD`). The key comparison is:
- Share of total safety funding received by each road class
- Average crash rate and share of statewide fatalities for each road class

This produces the bubble chart (`fig6_updated`) that forms the core of the investment gap finding.

---

### Section 8 — Random Forest

A Random Forest classifier is trained to predict whether a project will produce a meaningful crash rate improvement (change < −0.5 per MVMT). Features used:

- `FUNCTION_CLASS_CD` — road class
- `ADT_TOTAL_NBR` — average daily traffic
- `PCR_SCORE` — pavement condition
- `SPEED_LIMIT_NBR` — speed limit
- `SEG_LENGTH_MI` — segment length
- `PRIMARY_WORK_CATEGORY` — project type

The model achieved an OOB error of 41.6%, indicating that road characteristics alone are insufficient to predict project success.

---

### Section 9 — Shiny App (Lines ~[app start]–4135)

The Shiny app is defined at the bottom of the file and has two components:

**UI (`ui`)** — defines the layout using `shinydashboard`. The sidebar contains navigation menu items for each tab. The body contains the tab content panels.

**Server (`server`)** — contains all reactive logic. Each tab has its own `renderPlot()`, `renderTable()`, or `renderLeaflet()` block that responds to user input changes.

#### Tab Descriptions

**County Dashboard** — Four `renderPlot()` outputs showing crash rates, fatality share, and funding distribution for the selected county. The county selector uses a formatted dropdown (`HAM — Hamilton`) populated from the `county_name_lookup` table. A `renderText()` output dynamically updates the box title to reflect the selected county.

**Project Evaluator** — Renders a line chart showing crash rate by year for the selected project or project type. Uses `project_segment_lookup.rds` for fast before/after rate retrieval without re-running the full join.

**Risk Explorer** — Renders a `DT::datatable()` of high-risk untreated segments filtered by county, road class, and minimum segment length. An `observeEvent()` block listens for row selection and updates a `leafletProxy()` map to highlight the selected segment's geometry.

**Funding Gap** — Renders the investment gap bubble chart, reactive to county/statewide toggle and metric selection (funding share vs segment share).

**Cost Effectiveness** — Renders a scatter plot of project cost vs crash rate change. Points are colored green (improved), red (worsened), or gray (no change). Labels are added for the most extreme projects using `slice_min()` and `slice_max()`.

---

### Running the App

After running all preceding sections, launch the app with the final line:
```r
shinyApp(ui = ui, server = server)
```

The app will open in the RStudio viewer pane or a browser window. All data is already loaded in memory from the earlier sections — no additional file reads occur at app launch.

To run the app standalone (without re-running the full pipeline), use the pre-computed `app_data/` RDS files and run `app/app.R` directly:
```r
shiny::runApp("app/app.R")
```

---

## Key Objects Reference

| Object | Type | Description |
|---|---|---|
| `road_inventory` | sf data frame | 328,709 road segments with geometry |
| `crashes_all` | data frame | All crash records 2020–2024 combined |
| `crash_segment` | sf data frame | Crash records with segment ID assigned |
| `final_sf` | sf data frame | Master segment dataset with crashes + PCR |
| `panel_data` | data frame | Segment × year panel for DiD analysis |
| `project_segment_lookup` | data frame | 3,740 project-segment pairs with before/after rates |
| `county_summary` | data frame | Aggregated crash/funding stats by county × road class |
| `segments_geo` | sf data frame | Simplified segment geometries for the Leaflet map |
| `did_model` | fixest object | Main DiD model result |
| `category_data` | data frame | DiD effects by project type (feeds fig2) |
| `investment_gap_funding` | data frame | Funding vs risk by road class (feeds fig6) |
