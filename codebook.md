# Codebook
## Do Safety Dollars Save Lives? — ODOT Capstone Project
**Author: Levi Sayles | Xavier University | 2025–2026**

---

## Overview

This codebook documents all datasets, variables, and derived fields used in the analysis. All raw data was sourced from ODOT's public data systems. The final analysis dataset is a segment-year panel covering 328,709 road segments across five years (2020–2024, excluding 2022).

---

## 1. Crash Records

**Source:** ODOT Crash Data Repository  
**Files:** One CSV per year (Crash2020.csv through Crash2024.csv)  
**Records:** ~910,000 total across all years  
**Unit of observation:** Individual crash event  

| Variable | Type | Description |
|---|---|---|
| `CRASH_YR` | Integer | Year the crash occurred |
| `LATITUDE` | Double | GPS latitude of crash location |
| `LONGITUDE` | Double | GPS longitude of crash location |
| `MANNER_OF_CRASH_CD` | Character | Crash type code (angle, rear-end, sideswipe, etc.) |
| `CRASH_SEVERITY_CD` | Character | Severity: 1=Fatal, 2=Serious Injury, 3=Minor Injury, 4=PDO |
| `NUM_UNITS` | Integer | Number of vehicles involved |
| `ROADWAY_INVENTORY_ID` | Character | Road segment ID (assigned after spatial join — not in raw crash data) |

**Notes:**
- Crash records contain GPS coordinates but no segment ID. Segment assignment was performed via spatial nearest-feature join using `sf::st_nearest_feature()`.
- Each crash is assigned to the single nearest road segment geometry.

---

## 2. Road Inventory

**Source:** ODOT Road Inventory File  
**File:** Road inventory shapefile / CSV  
**Records:** 328,709 segments  
**Unit of observation:** Road segment  

| Variable | Type | Description |
|---|---|---|
| `ROADWAY_INVENTORY_ID` | Character | Unique segment identifier (primary key) |
| `NLF_ID` | Character | Route identifier used for project matching (e.g., `HAMCR00273**C`) |
| `CTL_BEGIN_NBR` | Double | Starting milepost of the segment along its route |
| `CTL_END_NBR` | Double | Ending milepost of the segment along its route |
| `FUNCTION_CLASS_CD` | Integer | Federal functional road class (1=Interstate, 2=Freeway, 3=Major Arterial, 4=Minor Arterial, 5=Collector, 6=Minor Collector, 7=Local) |
| `ADT_TOTAL_NBR` | Integer | Average Daily Traffic — total vehicles per day |
| `SPEED_LIMIT_NBR` | Integer | Posted speed limit (mph) |
| `SEG_LENGTH_MI` | Double | Segment length in miles |
| `COUNTY_CD` | Character | Two-letter Ohio county abbreviation (e.g., HAM, FRA) |
| `COUNTY_NME` | Character | Full county name |
| `DISTRICT_CD` | Integer | ODOT district number (1–12) |

**Derived variables:**

| Variable | Description |
|---|---|
| `annual_VMT` | Annual vehicle miles traveled = `ADT_TOTAL_NBR × SEG_LENGTH_MI × 365` |
| `crash_rate_MVMT` | Crash rate per million VMT = `(crash_count / annual_VMT) × 1,000,000` |

---

## 3. Pavement Condition

**Source:** ODOT PCR Database (local and state pavement records)  
**Unit of observation:** Road segment × inspection year  

| Variable | Type | Description |
|---|---|---|
| `ROADWAY_INVENTORY_ID` | Character | Segment identifier (join key) |
| `PCR_SCORE` | Double | Pavement Condition Rating (0–100; higher = better condition) |
| `IRI` | Double | International Roughness Index (lower = smoother) |
| `DISTRESS_SCORE` | Double | Composite distress indicator |
| `INSPECTION_YR` | Integer | Year of pavement inspection |

---

## 4. Construction Projects

**Source:** ODOT Construction Project Database  
**File:** Safety-funded projects extract  
**Records:** 13,550 safety-funded projects  
**Unit of observation:** Construction project  

| Variable | Type | Description |
|---|---|---|
| `PROJECT_NME` | Character | Project name as recorded by ODOT |
| `NLF_ID` | Character | Route identifier (join key to road inventory) |
| `CTL_BEGIN` | Double | Starting milepost of the project along its route |
| `CTL_END` | Double | Ending milepost of the project along its route |
| `PRIMARY_WORK_CATEGORY` | Character | Project type category (see categories below) |
| `EST_TOTAL_CONSTR_COST` | Double | Estimated total construction cost in dollars |
| `COMPLETION_DT` | Date | Project completion date |
| `completion_year` | Integer | Derived: year extracted from COMPLETION_DT |
| `COUNTY_CD` | Character | County where project is located |
| `COUNTY_NME` | Character | Full county name |
| `DISTRICT_CD` | Integer | ODOT district number |
| `SAFETY_FUNDED_FLAG` | Character | Indicates project received safety funding |

**Project type categories (`PRIMARY_WORK_CATEGORY`):**

| Category | Description |
|---|---|
| Intersection Improvement (Safety) | Roundabouts, turn lanes, signal upgrades at intersections |
| Traffic Control (Safety) | Signal timing, variable message signs, traffic control systems |
| Roadway Minor Rehab | Pavement overlays, resurfacing, minor repair |
| Roadside/Median Imp. | Guardrail, barrier, median improvements |
| Pavement Treatments (Safety) | Safety-designated pavement surface treatments |
| Pedestrian Facilities | Sidewalks, crosswalks, ADA ramps |

---

## 5. Master Analysis Dataset (Panel)

**Derived from:** All four sources above  
**Unit of observation:** Road segment × year  
**Dimensions:** 328,709 segments × 4 years (2020, 2021, 2023, 2024)  
**File:** Built by `analysis/01_data_prep.R`; saved as `analysis_app.rds`  

| Variable | Type | Description |
|---|---|---|
| `ROADWAY_INVENTORY_ID` | Character | Segment identifier (primary key) |
| `CRASH_YR` | Integer | Observation year |
| `crash_count` | Integer | Number of crashes recorded on this segment in this year |
| `crash_rate_MVMT` | Double | Crash rate per million vehicle miles traveled |
| `annual_VMT` | Double | Annual vehicle miles traveled for this segment |
| `treated_flag` | Integer | 1 if segment received a safety-funded project, 0 otherwise |
| `post_flag` | Integer | 1 if year is after project completion, 0 otherwise |
| `did_term` | Integer | Interaction term = `treated_flag × post_flag` (DiD estimator) |
| `completion_year` | Integer | Year project on this segment completed (NA if untreated) |
| `PRIMARY_WORK_CATEGORY` | Character | Project type (NA if untreated) |
| `FUNCTION_CLASS_CD` | Integer | Federal functional road class |
| `ADT_TOTAL_NBR` | Integer | Average daily traffic |
| `SEG_LENGTH_MI` | Double | Segment length in miles |
| `PCR_SCORE` | Double | Pavement condition rating |
| `COUNTY_CD` | Character | County abbreviation |
| `COUNTY_NME` | Character | Full county name |

**Note on 2022:** Year 2022 is excluded from the main analysis as a transition year — many projects were actively under construction throughout 2022, creating ambiguity between before and after periods. A robustness check re-introducing 2022 confirmed findings are unchanged.

---

## 6. Project-Segment Lookup Table

**File:** `app_data/project_segment_lookup.rds`  
**Unit of observation:** Project × segment pair  
**Records:** 3,740 matched pairs  
**Purpose:** Used by the Shiny app's Project Evaluator tab to compute before/after crash rates for individual projects  

| Variable | Type | Description |
|---|---|---|
| `PROJECT_NME` | Character | Project name |
| `ROADWAY_INVENTORY_ID` | Character | Matched segment ID |
| `PRIMARY_WORK_CATEGORY` | Character | Project type |
| `COUNTY_CD` | Character | County abbreviation |
| `COUNTY_NME` | Character | Full county name |
| `completion_year` | Integer | Year project completed |
| `EST_TOTAL_CONSTR_COST` | Double | Total project cost |
| `before_rate` | Double | Average crash rate in pre-period (2020–2021) |
| `after_rate` | Double | Average crash rate in post-period (2023–2024) |
| `change` | Double | `after_rate − before_rate` |
| `FUNCTION_CLASS_CD` | Double | Road class of matched segment |

---

## 7. Key Matching Logic

### Crash-to-Segment Matching
```r
# Spatial nearest-feature join
crashes_sf <- st_as_sf(crash_df, coords = c("LONGITUDE","LATITUDE"), crs = 4326)
segments_sf <- st_transform(segments_sf, crs = 4326)
nearest_idx <- st_nearest_feature(crashes_sf, segments_sf)
crash_df$ROADWAY_INVENTORY_ID <- segments_sf$ROADWAY_INVENTORY_ID[nearest_idx]
```

### Project-to-Segment Matching
```r
# Step 1: Join on shared route identifier
matched <- inner_join(segments, projects, by = "NLF_ID")

# Step 2: CTL milepost range overlap filter
matched <- matched %>%
  filter(
    CTL_BEGIN_NBR <= CTL_END,    # segment starts before project ends
    CTL_END_NBR   >= CTL_BEGIN   # segment ends after project starts
  )
```

---

## 8. Functional Road Class Reference

| Code | Class | Typical Roads |
|---|---|---|
| FC1 | Interstate | I-75, I-71, I-270 |
| FC2 | Freeway / Expressway | State expressways, limited-access roads |
| FC3 | Principal Arterial | Major state routes, urban arterials |
| FC4 | Minor Arterial | Secondary state routes, suburban arterials |
| FC5 | Collector | Township and county collector roads |
| FC6 | Minor Collector | Rural collectors |
| FC7 | Local | Local streets, residential roads |

---

## 9. DiD Model Specification

```
crash_rate_MVMTᵢₜ = αᵢ + γₜ + β(treated × post)ᵢₜ + εᵢₜ
```

Where:
- `αᵢ` = segment fixed effect (controls for time-invariant segment characteristics)
- `γₜ` = year fixed effect (controls for statewide time trends)
- `β` = DiD estimate — the average treatment effect of receiving a safety-funded project
- Standard errors clustered at segment level

Estimated using `fixest::feols()` in R.
