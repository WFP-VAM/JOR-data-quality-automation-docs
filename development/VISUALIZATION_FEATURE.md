# Monitoring Visualizations Feature

## Overview

Added optional visualization capabilities to exercises, allowing custom monitoring dashboards to appear below validation reports.

## What Was Added

### 1. New File: `R/visualization_helpers.r`

Reusable visualization components:

- **`metric_card()`** - Percentage metric with color coding:
  - 🟢 Green (≥80%) - Good
  - 🟡 Yellow (50-79%) - Needs attention  
  - 🔴 Red (<50%) - Critical
  
- **`breakdown_card()`** - Categorical data breakdown with horizontal bar charts

- **`osm_monitoring_dashboard()`** - Complete OSM monitoring overview
- **`bcm_monitoring_dashboard()`** - BCM helpdesk validation monitoring
- **`time_metric_card()`** - Time-based metrics with average, median, range
- **`satisfaction_metric_card()`** - Binary satisfaction metrics

### 2. Updated: `exercises/osm_survey.r`

Added `visualizations()` function that displays:

#### 📍 Accessibility & Infrastructure

- Public transportation access
- Physical disability access
- Sun/rain protection

#### 🚿 WASH Facilities

- Breakdown of available facilities (toilet, soap, water)

#### 🪑 Site Amenities

- Seating arrangements (waiting area, counseling space)

#### 👁️ WFP Visibility & Communication

- WFP banners/posters visibility
- WFP Hotline cards availability
- CP staff joint visibility materials

### 3. Updated: `exercises/bcm_survey.r`

Added `visualizations()` function that displays:

#### ⏱️ Service Time Metrics

- Average waiting time (with range and median)
- Average process time (with range and median)
- Color-coded: 🟢 ≤10min, 🟡 10-20min, 🔴 >20min

#### 😊 Beneficiary Satisfaction

- Treated respectfully
- Satisfied with validation process
- Know validation result

#### 👤 User Experience

- Comfortable using equipment
- Difficulty locating station

#### 📊 Demographics & Communication

- Gender distribution
- How beneficiaries learned about validation

### 4. Updated: `app/app.r`

- Sources `visualization_helpers.r`
- Renders visualizations below validation report
- Both OSM and BCM now have custom dashboards

## How It Works

```r
# In exercise definition
exercise <- list(
  # ... metadata, fetch_data, prepare_data, validations ...
  
  # Optional visualizations
  visualizations = function(data) {
    osm_monitoring_dashboard(data)
  }
)
```

When a user selects an exercise:

1. ✅ Data is fetched and validated
2. ✅ Validation report is displayed
3. ✅ If exercise has `visualizations()`, it's rendered below the report

## Visual Design

- **Metric Cards**: Clean percentage display with color coding
- **Breakdown Cards**: Horizontal bar charts with percentages
- **Responsive Layout**: Uses Bootstrap grid (3-column for metrics)
- **Modern Styling**: Box shadows, border radius, gradient bars

## Benefits

### ✅ Self-Contained
Each exercise can define its own visualizations specific to its data structure

### ✅ Flexible Design
Each exercise has its own dashboard tailored to its data (OSM for site monitoring, BCM for service quality)

### ✅ Reusable Components
`metric_card()` and `breakdown_card()` can be used by any exercise

### ✅ Flexible
Exercises can use helpers or create completely custom visualizations

## Example Outputs

### OSM Survey (91 records)

```
Site Monitoring Overview

Accessibility & Infrastructure
┌────────────────────────┬────────────────────────┬────────────────────────┐
│ Public Transportation  │ Physical Disability    │ Sun/Rain Protection    │
│ 87.9%                 │ 72.5%                 │ 98.9%                 │
│ 80/91 sites           │ 66/91 sites           │ 90/91 sites           │
└────────────────────────┴────────────────────────┴────────────────────────┘

WASH Facilities
┌──────────────────────────────────────────────────────────────┐
│ toilet soap water     ████████████████░░░░  42 (46.2%)      │
│ toilet water soap     ██████████░░░░░░░░░  23 (25.3%)      │
│ toilet               ████░░░░░░░░░░░░░  9 (9.9%)          │
│ ...                                                           │
└──────────────────────────────────────────────────────────────┘

[And more sections...]
```

### BCM Survey (69 records)

```
Helpdesk Validation Monitoring

Service Time Metrics
┌────────────────────────┬────────────────────────┐
│ Average Waiting Time   │ Average Process Time   │
│ 5.2min                │ 5.4min                │
│ Range: 0-10 min       │ Range: 2-10 min       │
│ median: 5min          │ median: 5min          │
└────────────────────────┴────────────────────────┘

Beneficiary Satisfaction
┌────────────────────┬────────────────────┬────────────────────┐
│ Treated            │ Satisfied with     │ Know Validation    │
│ Respectfully       │ Process            │ Result             │
│ 100%              │ 100%              │ 100%              │
└────────────────────┴────────────────────┴────────────────────┘

[And more sections...]
```

## Data-to-Question Mapping

### OSM Exercise

| Question | Column | Visualization |
|----------|--------|---------------|
| Is site accessible by public transportation? | `site_access` | Metric card |
| Is infrastructure accessible for disabilities? | `site_infra` | Metric card |
| Is site shaded? | `site_shade` | Metric card |
| Which WASH facilities available? | `site_wash` | Breakdown card |
| Seating arrangements? | `site_seating` | Breakdown card |
| WFP banners visible? | `wfp_banners` | Metric card |
| WFP Hotline cards available? | `wfp_cards` | Metric card |
| CP staff joint visibility? | `joint_visibility` | Metric card |

### BCM Exercise

| Question/Metric | Column | Visualization |
|-----------------|--------|---------------|
| Average waiting time | `Q4_2_waitingtime` | Time metric card |
| Average process time | `Q4_3_processtime` | Time metric card |
| Treated respectfully | `Q4_1_were_trated_respectfully` | Satisfaction metric |
| Satisfied with validation process | `Q4_2_satisfied_w_validation_process` | Satisfaction metric |
| Know validation result | `Q4_4_know_validation_result` | Satisfaction metric |
| Comfortable using equipment | `Q4_3_comfortable_using_validation_equip` | Satisfaction metric |
| Difficulty locating station | `Q2_3_difficulty_locating_validationstation` | Metric card (inverted) |
| Gender distribution | `Q2_1_gender` | Breakdown card |
| How learned about validation | `Q2_1_how_did_you_know_about_validation` | Breakdown card |

## Adding to New Exercises

1. **Use existing helpers**:
   ```r
   visualizations = function(data) {
     tagList(
       h3("My Custom Dashboard"),
       fluidRow(
         column(4, metric_card(data, "my_column", "My Metric")),
         column(8, breakdown_card(data, "categories", "Categories"))
       )
     )
   }
   ```

2. **Or create custom viz**:
   ```r
   visualizations = function(data) {
     # Use any R viz library: ggplot2, plotly, etc.
     # Return Shiny HTML elements
   }
   ```

3. **Or skip it**:
   ```r
   # Simply don't define visualizations - exercise works fine without!
   ```

## Testing

```bash
# Verify it loads
cd app
Rscript -e "source('R/visualization_helpers.r'); cat('✓ Loaded\n')"

# Check exercises
Rscript -e "
  source('R/visualization_helpers.r')
  source('R/exercise_registry.r')
  exercises <- load_exercises()
  cat('OSM has viz:', !is.null(exercises[['5373']]$visualizations), '\n')
  cat('BCM has viz:', !is.null(exercises[['5387']]$visualizations), '\n')
"
```

## Future Enhancements

- Interactive charts with plotly
- Time-series visualizations for monitoring trends
- Export visualizations as PNG/PDF
- Drill-down capabilities
- Comparative views across exercises

## References

- OSM monitoring questions mapped to visualizations
- Bootstrap grid system for responsive layout
- Shiny tagList for dynamic UI composition

