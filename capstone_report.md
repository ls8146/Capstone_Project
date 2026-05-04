# Do Safety Dollars Save Lives?
## Evaluating ODOT's Safety-Funded Road Construction Projects

**Xavier University | Data Science Senior Capstone | 2025–2026**  
**Author: Levi Sayles**  
**Partner Organization: Ohio Department of Transportation (ODOT) District 8**

---

## Abstract

This report evaluates whether Ohio Department of Transportation (ODOT) safety-funded road construction projects reduce crash rates across Ohio's road network. Using a panel dataset of 328,709 road segments observed over five years (2020–2024), a Difference-in-Differences (DiD) causal inference framework is applied to estimate the average treatment effect of safety-funded construction on crash rates. The overall effect is not statistically significant (p = 0.185). However, disaggregating by project type reveals that intersection improvements produce a significant and consistent crash rate reduction of 6.54 crashes per million vehicle miles traveled — the only project type with a clear safety return. Additionally, a significant investment gap is identified: roads carrying over half of Ohio's fatalities receive less than 30% of safety funding. These findings are packaged into an interactive R Shiny application for ongoing use by ODOT analysts.

---

## 1. Background and Motivation

### 1.1 The Problem

Ohio recorded 1,390 traffic fatalities in 2023. ODOT spent over $140 million on safety-funded construction projects that year, with the explicit goal of reducing crash rates on dangerous roads. Despite this significant investment, no systematic evaluation framework existed to assess whether these projects were achieving their intended safety outcomes.

This gap was confirmed during meetings with ODOT engineers and planners early in the project. As one engineer noted during an initial meeting: *"We look for projects based on current needs — but we never go back to evaluate them."* This project is a direct response to that observation.

### 1.2 Research Questions

1. Do safety-funded road construction projects reduce crash rates on treated road segments compared to similar untreated segments?
2. Which project types, if any, produce measurable safety improvements?
3. Is safety funding being allocated to the highest-risk roads in the network?
4. Can road characteristics predict which projects will succeed?

### 1.3 ODOT Collaboration

Two formal meetings were held with ODOT District 8 staff throughout the project. The first meeting presented initial analysis results and received feedback on data interpretation. The second meeting presented final findings, which ODOT confirmed reflected patterns they recognize in the field — including the investment gap on local and collector roads.

---

## 2. Data

### 2.1 Data Sources

Four datasets were combined from ODOT's public data systems:

| Dataset | Records | Years |
|---|---|---|
| Crash Records | ~910,000 individual crash events | 2020–2024 |
| Road Inventory | 328,709 road segments statewide | 2024 snapshot |
| Pavement Condition | Segment-level PCR scores and IRI | 2020–2024 |
| Construction Projects | 13,550 safety-funded projects | 2020–2024 |

### 2.2 Building the Analysis Dataset

Constructing the master analysis dataset required solving three distinct technical challenges.

**Challenge 1: Matching 910,000 Crashes to Road Segments**

Crash records contain GPS coordinates but no road segment identifier. Road segments exist as line geometries in a separate database. With no shared key between the two datasets, each of 910,000 crash points was matched to its closest road segment geometry using a spatial nearest-feature join (`sf::st_nearest_feature()` in R). This single operation was the most computationally intensive step in the pipeline.

**Challenge 2: The Milepost Range Overlap Problem**

Construction projects are defined by a route identifier (NLF_ID) and a milepost range — a start and end point along that route. A naive join on NLF_ID alone would match an entire road to a project, potentially pulling in miles of road that had nothing to do with the construction work.

To solve this, a range overlap filter was written that checks whether each segment's milepost range physically intersects the project's milepost boundaries:

```r
matched <- matched %>%
  filter(
    CTL_BEGIN_NBR <= CTL_END,   # segment starts before project ends
    CTL_END_NBR   >= CTL_BEGIN  # segment ends after project starts
  )
```

This produced 3,740 valid project-segment pairs from 13,550 projects.

**Challenge 3: Panel Structure and the 2022 Transition Year**

The final dataset is a panel: 328,709 segments observed across multiple years with crash counts, crash rates, and treatment indicators for each segment-year combination.

Year 2022 was excluded from the main analysis as a transition year. Many safety-funded projects were actively under construction throughout 2022, meaning that year contains a mix of under-construction roads, partially-completed projects, and newly-completed projects. Including 2022 in either the before or after period would contaminate the measurement. The final before period is 2020–2021 and the after period is 2023–2024.

A robustness check re-introducing 2022 confirmed that excluding it did not drive the main findings.

### 2.3 Crash Rate Computation

Raw crash counts are not a useful comparison metric across segments — a road carrying 50,000 vehicles per day has far more exposure to crashes than one carrying 500. All analysis uses crash rates normalized by traffic exposure:

```
Crash Rate = (Annual Crash Count / Annual VMT) × 1,000,000
```

Where Annual VMT (Vehicle Miles Traveled) = ADT × Segment Length (mi) × 365.

This produces crash rates in units of **crashes per million vehicle miles traveled (MVMT)**.

---

## 3. Methodology

### 3.1 Difference-in-Differences (DiD)

DiD is a quasi-experimental causal inference technique that compares how outcomes change over time for a treated group relative to a control group. In this context:

- **Treated group:** Road segments that received a safety-funded construction project
- **Control group:** Similar untreated road segments matched on Average Daily Traffic (ADT)
- **Before period:** 2020–2021
- **After period:** 2023–2024

The DiD estimate captures the change in crash rates on treated roads relative to the change on control roads over the same period — removing baseline differences and statewide time trends from the estimate.

**Model specification:**

```
crash_rate_MVMTᵢₜ = αᵢ + γₜ + β(treated × post)ᵢₜ + εᵢₜ
```

Where:
- `αᵢ` = segment fixed effects (remove time-invariant differences between segments)
- `γₜ` = year fixed effects (remove statewide year-to-year trends)
- `β` = the DiD estimate — average treatment effect of receiving a safety project
- Standard errors are clustered at the segment level

The model was estimated using `fixest::feols()` in R.

**Why DiD for this project?**

Roads receiving safety funding are not randomly selected — they are specifically identified as problem locations by engineers. This means treated roads are systematically different from untreated roads before construction begins. DiD controls for this by using change over time rather than levels, removing fixed baseline differences between groups.

### 3.2 Event Study

In addition to the pooled DiD estimate, an event study was conducted for each project type. Crash rates are plotted relative to project completion year (t=0), allowing visual inspection of:
- **Pre-trends:** Whether treated and control roads were trending similarly before treatment (a key DiD assumption)
- **Post-completion trajectories:** How crash rates evolve after project completion over multiple years

### 3.3 Subgroup Analysis by Project Type

The pooled DiD estimate is computed separately for each primary work category to identify which types of projects, if any, produce safety benefits. Categories with fewer than 200 matched treated segments are excluded from the subgroup analysis due to insufficient sample size.

### 3.4 Investment Gap Analysis

Statewide funding allocation is compared to crash risk by functional road class. For each class, two metrics are computed:
- Share of total safety funding received
- Average crash rate (per MVMT) and share of statewide fatalities

This identifies road classes that are over- or under-funded relative to their risk level.

### 3.5 Random Forest

A Random Forest classifier was trained to predict whether a safety-funded project would produce a meaningful crash rate improvement (defined as a reduction greater than 0.5 per MVMT). Features include road class, ADT, PCR score, IRI, speed limit, segment length, project type, and county. Out-of-bag (OOB) error was used to assess model performance.

---

## 4. Results

### 4.1 Main DiD Result

The pooled DiD estimate across all safety-funded project types is **+0.014 crashes per MVMT** (p = 0.185). This result is not statistically significant. Safety-funded projects as a whole do not produce a measurable reduction in crash rates relative to comparable untreated roads.

This finding is robust across three levels of standard error clustering:

| Clustering Level | Coefficient | p-value |
|---|---|---|
| Segment | +0.014 | 0.185 |
| NLF Route | +0.014 | 0.191 |
| County | +0.014 | 0.203 |

The estimate is identical across all three specifications, confirming stability of the result.

### 4.2 Results by Project Type

Disaggregating by project type reveals important heterogeneity in outcomes:

| Project Type | DiD Effect (per MVMT) | Direction |
|---|---|---|
| Intersection Improvement (Safety) | −6.54 | ✅ Significant reduction |
| Pedestrian Facilities | +0.14 | No effect |
| Roadway Minor Rehab | +0.34 | No effect |
| Roadside/Median Improvement | +1.15 | Slight increase |
| Pavement Treatments (Safety) | +1.65 | Slight increase |
| Traffic Control (Safety) | +2.49 | Increase |

Intersection improvements are the only project type with a clear, consistent, and directionally expected safety return. All other categories show no statistically significant improvement and several show slight increases in crash rates post-completion.

### 4.3 Hamilton County Case Studies

Three Hamilton County projects illustrate the range of outcomes hidden within the overall null result:

**Plainfield Road Roundabouts (Intersection Improvement)**
- Before: 75.3 crashes/MVMT → After: 30.1 crashes/MVMT
- Change: −45.2 per MVMT (−60%)
- Cost: $8,863,255
- Mechanism: Roundabout redesign eliminates the most dangerous crash types (angle crashes, left-turn crashes) by removing conflict points

**Eastern Corridor VAR TSG (Traffic Control)**
- Before: 19.9 crashes/MVMT → After: 34.8 crashes/MVMT
- Change: +14.9 per MVMT
- Mechanism: Signal timing changes appear to have redirected traffic in ways that increased conflict points at nearby intersections

**SR 32 / SR 125 AC Overlay (Roadway Minor Rehab)**
- Before: 0.82 crashes/MVMT → After: 0.82 crashes/MVMT
- Change: 0.0 per MVMT
- Mechanism: Pavement overlay improves ride quality and braking but has no mechanism to reduce crash rates — it does not change road geometry, speed limits, or vehicle conflict points

These three projects illustrate why the overall null result emerges: dramatically different outcomes in opposite directions average out to near-zero.

### 4.4 Investment Gap

The statewide safety investment distribution does not align with crash risk by road class:

| Road Class | % of Fatalities | % of Safety Funding |
|---|---|---|
| FC1: Interstate | 8.1% | 22.0% |
| FC2: Freeway | 2.3% | 6.9% |
| FC3: Major Arterial | 19.3% | 39.5% |
| FC4: Minor Arterial | 24.0% | 19.8% |
| FC5: Collector | 28.5% | 9.8% |
| FC6: Minor Collector | 4.9% | 0.6% |
| FC7: Local | 12.8% | 1.4% |

FC4 and FC5 roads together account for 52.5% of Ohio fatalities and receive 29.6% of safety funding. Local roads (FC7) have the highest average crash rate in the network (36.5 per MVMT) and receive 1.4% of safety dollars.

In Hamilton County, this pattern is more pronounced: FC1 interstates capture 63% of District 8 safety funding despite the lowest crash rate in the county.

### 4.5 Random Forest Results

The Random Forest model achieved an OOB error rate of **41.6%** — essentially random classification. Road characteristics alone (functional class, ADT, pavement condition, speed limit, segment length, project type) cannot predict whether a safety-funded project will produce a crash rate improvement.

This finding suggests that project success is driven by factors not captured in ODOT's road inventory — likely including site-specific geometry, intersection configuration, driver behavior patterns, and implementation quality.

---

## 5. Limitations

### 5.1 Data Does Not Capture Everything

Crash records confirm that a crash occurred but do not record the underlying cause. Driver behavior, weather conditions, sight distance limitations, and local road context are not present in the dataset. Differences between treated and untreated roads on these unmeasured dimensions cannot be fully controlled for, even with segment fixed effects.

### 5.2 Causation vs Correlation

When crash rates decline after a project, it is tempting to attribute the change to the project. However, this is an observational study — roads are not randomly assigned to receive safety funding. The DiD design comes closer to causal identification than a simple before-after comparison, but the possibility of unobserved confounders cannot be fully eliminated.

### 5.3 Regression to the Mean

Roads are often selected for safety funding following a period of elevated crash rates. Statistical regression to the mean predicts that crash rates will naturally decline after a spike, regardless of any intervention. The event study plots show some evidence of pre-completion decline in treated road crash rates, which may partially reflect this phenomenon rather than genuine project effects.

### 5.4 Partial Safety Funding Labels

ODOT noted during meetings that some projects are labeled safety-funded but only partially draw from dedicated safety dollars — a portion of the budget may come from general construction or maintenance funds. Cost effectiveness estimates in this analysis use total project cost, which likely overstates the true cost per safety dollar spent.

---

## 6. The ODOT Safety Investment Analyzer

### 6.1 Motivation

A one-time analysis answers these questions for the 2020–2024 period but does not solve the underlying problem: ODOT spends safety dollars every year, new projects complete every year, and new crash data arrives every year. The analytical findings need to be embedded in a tool that ODOT analysts can use routinely without writing code.

### 6.2 Application Overview

The ODOT Safety Investment Analyzer is an interactive R Shiny dashboard with five tabs:

**County Dashboard**
Displays crash rates by road class, fatality share, and funding distribution for any Ohio county or statewide. Allows instant comparison of crash risk and investment patterns.

**Project Evaluator**
Shows before/after crash rate trends for any safety-funded project in the database. Can display individual projects by name or aggregate all projects of a given type in a county to produce an event study chart.

**Risk Explorer**
Lists the highest-risk untreated road segments in any Ohio county, filterable by road class and minimum segment length. Clicking any row highlights the segment on an interactive Leaflet map.

**Funding Gap**
Replicates the statewide and county-level investment gap analysis interactively. Users can toggle between crash rate and fatality share as the risk metric.

**Cost Effectiveness**
Plots project cost against crash rate change for all matched projects. Each point is one project, colored by outcome (improved, worsened, no change). Allows identification of high-value and low-value projects.

### 6.3 Technical Implementation

The app uses pre-computed RDS data files to load instantly without re-running any analysis. All heavy computation (spatial joins, panel construction, DiD estimates) is done offline and saved to `app_data/`. The app then reads these files and produces interactive visualizations using `plotly`, `leaflet`, and `ggplot2`.

---

## 7. Conclusions

**Safety funding works — but only when it targets the right interventions on the right roads.**

Three conclusions follow from this analysis:

**1. Intersection improvements are the right intervention.** They are the only project type with a clear, consistent, replicable safety return. At −6.54 crashes per MVMT, the effect is large and directionally consistent across geographies and years. They are expensive, but they work by fundamentally changing how vehicles interact — eliminating the most dangerous conflict points rather than just improving the road surface.

**2. Not all safety projects prevent crashes — and that is acceptable, but the label matters.** Pavement overlays improve ride quality, extend road life, and protect tires. Traffic signal upgrades reduce delays. These are valuable investments. But they do not reduce crash rates in the way this analysis measures, and evaluating them on that basis sets up a misleading standard. The safety funding label should reflect the expected mechanism of impact.

**3. The highest-risk roads are receiving the least attention.** Collector and local roads carry over half of Ohio's fatalities and receive a fraction of safety investment. Even a modest reallocation of intersection improvement funding toward FC5 and FC7 roads — where crash rates are highest and investment is lowest — could produce measurable safety gains at the population level.

---

## 8. Future Work

- Extend the analysis through 2025 and 2026 as new crash data becomes available
- Incorporate severity-weighted crash rates (fatal and serious injury crashes only) as an alternative outcome measure
- Develop a matching algorithm that pairs treated segments with control segments on additional covariates beyond ADT (e.g., road class, speed limit)
- Explore heterogeneous treatment effects — does the effectiveness of intersection improvements vary by road class, traffic volume, or county?
- Investigate the partial funding label problem with ODOT to produce more accurate cost-per-safety-dollar estimates

---

## References

- ODOT Crash Data Repository: https://www.transportation.ohio.gov/programs/safety/crash-data
- ODOT Road Inventory: https://gis.dot.state.oh.us/tims/
- Callaway, B., & Sant'Anna, P. H. (2021). Difference-in-differences with multiple time periods. *Journal of Econometrics*, 225(2), 200–230.
- Angrist, J. D., & Pischke, J. S. (2009). *Mostly Harmless Econometrics*. Princeton University Press.

---

## Appendix: R Package Versions

| Package | Version | Use |
|---|---|---|
| `tidyverse` | 2.0.0 | Data manipulation and visualization |
| `sf` | 1.0-14 | Spatial operations and nearest-feature join |
| `fixest` | 0.11.2 | DiD model estimation with fixed effects |
| `randomForest` | 4.7-1.1 | Random Forest classifier |
| `shiny` | 1.7.5 | Interactive application framework |
| `shinydashboard` | 0.7.2 | Dashboard layout |
| `leaflet` | 2.2.0 | Interactive maps |
| `plotly` | 4.10.2 | Interactive charts |
| `scales` | 1.2.1 | Number formatting |
| `ggplot2` | 3.4.4 | Static visualizations |
