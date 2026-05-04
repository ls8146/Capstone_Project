# Do Safety Dollars Save Lives?
### Evaluating ODOT's Safety-Funded Road Construction Projects

**Xavier University | Data Science Senior Capstone | 2025–2026**
**Author: Levi Sayles**

---

## Project Overview

This project evaluates whether Ohio Department of Transportation (ODOT) safety-funded road construction projects reduce crash rates across Ohio's road network. Using five years of crash records, road inventory data, pavement condition data, and construction project records, a full analysis pipeline was built to measure project effectiveness using a Difference-in-Differences (DiD) causal inference framework.

The project also includes an interactive R Shiny application — the **ODOT Safety Investment Analyzer** — that allows ODOT analysts to explore project outcomes, identify high-risk untreated road segments, and assess where safety dollars are having the most measurable impact, without writing any code.

---

## Key Findings

- **Overall effect of safety-funded projects is not statistically significant** (p = 0.185, coefficient = +0.014) — safety investment as a whole does not produce a measurable reduction in crash rates relative to comparable untreated roads
- **Intersection improvements are the exception** — the only project type with a clear, consistent safety return (DiD effect: −6.54 crashes per MVMT)
- **A significant investment gap exists** — FC4 and FC5 roads (collectors and minor arterials) account for 52.5% of Ohio fatalities but receive less than 30% of safety funding; local roads (FC7) receive just 1.4% of safety dollars despite the highest crash rate in the network
- **Road characteristics alone cannot predict project success** — a Random Forest model trained on segment-level features achieved only 41.6% OOB accuracy, essentially random

---

## Repository Structure

```
├── README.md                        ← This file
├── report/
│   └── capstone_report.md           ← Full written technical report
├── codebook/
│   └── codebook.md                  ← Data dictionary for all datasets and variables
├── analysis/
│   ├── 01_data_prep.R               ← Spatial join, panel construction, cleaning
│   ├── 02_did_analysis.R            ← DiD model, event study, robustness checks
│   ├── 03_investment_gap.R          ← Funding vs risk analysis by road class
│   ├── 04_random_forest.R           ← ML model for project success prediction
│   └── 05_visualizations.R          ← All ggplot2 figures
├── app/
│   └── app.R                        ← R Shiny application (ODOT Safety Investment Analyzer)
│   └── app_data/                    ← Pre-computed RDS files for fast app loading
│       ├── analysis_app.rds
│       ├── segment_year_app.rds
│       ├── projects_app.rds
│       ├── county_summary.rds
│       ├── segments_geo.rds
│       ├── county_geo.rds
│       └── project_segment_lookup.rds
└── presentation/
    └── odot_capstone.pptx           ← Final defense presentation
```

---

## Data Sources

All data was sourced from ODOT's public data systems. No external or proprietary data was used.

| Dataset | Source | Records | Years |
|---|---|---|---|
| Crash Records | ODOT Crash Data Repository | ~910,000 | 2020–2024 |
| Road Inventory | ODOT Road Inventory File | 328,709 segments | 2024 snapshot |
| Pavement Condition | ODOT PCR Database (local + state) | Segment-level | 2020–2024 |
| Construction Projects | ODOT Construction Project Database | 13,550 projects | 2020–2024 |

See `codebook/codebook.md` for full variable descriptions.

---

## Methodology Summary

### 1. Data Pipeline
- **Crash-to-segment matching:** 910,000 crash GPS points matched to nearest road segment geometry using `sf::st_nearest_feature()` in R
- **Project-to-segment matching:** Projects matched to segments by NLF_ID (route) with CTL milepost range overlap filter to ensure physical intersection
- **Panel construction:** 328,709 segments × 5 years; crash rates computed per million vehicle miles traveled (MVMT); 2022 dropped as transition year

### 2. Difference-in-Differences (DiD)
- Compared crash rate changes on treated roads (received safety-funded project) vs untreated control roads matched on Average Daily Traffic (ADT)
- Model includes segment fixed effects and year fixed effects
- Standard errors clustered at segment level
- Robustness checks: clustering at NLF and county level; re-introduction of 2022

### 3. Event Study
- Crash rates plotted relative to project completion year (t=0) for each project type
- Allows visual inspection of pre-trends and post-completion trajectories

### 4. Random Forest
- Trained on segment-level road characteristics to predict project success (crash rate improvement)
- Features: road class, ADT, PCR score, IRI, speed limit, segment length, project type
- OOB error: 41.6% — road characteristics alone are insufficient to predict which projects will succeed

---

## How to Run the Analysis

### Requirements
- R ≥ 4.3.0
- Key packages: `tidyverse`, `sf`, `fixest`, `randomForest`, `shiny`, `shinydashboard`, `leaflet`, `plotly`, `scales`

Install all dependencies:
```r
install.packages(c("tidyverse", "sf", "fixest", "randomForest",
                   "shiny", "shinydashboard", "leaflet", "plotly", "scales"))
```

### Run Order
```
1. analysis/01_data_prep.R        ← Build master panel dataset
2. analysis/02_did_analysis.R     ← Run DiD model and event study
3. analysis/03_investment_gap.R   ← Investment gap analysis
4. analysis/04_random_forest.R    ← Random Forest model
5. analysis/05_visualizations.R   ← Generate all figures
```

### Run the Shiny App
```r
shiny::runApp("app/app.R")
```

Or with pre-computed data already in `app/app_data/`, the app loads instantly without re-running any analysis.

---

## Updating the App with New Data

The app is designed to be refreshed annually. To update:

1. Download new crash year file from ODOT and run `01_data_prep.R` with updated year range
2. Re-run `02_did_analysis.R` through `04_random_forest.R`
3. Re-export `app_data/` RDS files
4. Relaunch the app

---

## Contact

Levi Sayles | Xavier University | Data Science
Project completed in partnership with ODOT District 8
