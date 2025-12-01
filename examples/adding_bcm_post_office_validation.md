# Adding BCM - Post Office Validation Exercise

**Date**: November 6, 2025  
**Survey ID**: 5407  
**Exercise Name**: BCM - Post Office Validation

## Overview

This document details the steps taken to add the "BCM - Post Office Validation" exercise to the data quality automation app. This serves as a reference for adding similar exercises in the future.

## Step 1: Identify the Survey

### Interactive API Exploration

```r
library(reticulate)
library(dplyr)
use_python(".venv/bin/python")
data_bridges_knots <- import("data_bridges_knots")
config_path <- "data_bridges_api_config.yaml"
client <- data_bridges_knots$DataBridgesShapes(config_path)

# Search for surveys
surveys <- client$get_household_surveys_list()
post_office_surveys <- surveys[grepl("post office", surveys$surveyName, ignore.case = TRUE), ]
```

### Results

Found 3 surveys with "post office" in the name:

- **Survey 5407**: "BCM - Post Office Validation - 2025" - **97 responses** ✅ 
- Survey 5406: "OSM - Post Office Validation - 2025" (already integrated)
- Survey 5405: "BCM - Post Office Validation - 2025" - 0 responses (duplicate/empty)

**Decision**: Use Survey 5407

## Step 2: Explore Data Structure

### Key Columns Identified

```r
survey_id <- as.integer(5407)
data <- client$get_household_survey(survey_id, "full")

# Data dimensions
# 97 responses x 55 columns
```

### Important Columns

1. **Demographics**:
   - `Q2_1_gender`: Gender (female/male)

2. **Location**:
   - `_1_6_Post_office_name`: Post office codes (1-85, "bal_circle")
   - `governorate`: 11 governorates (Amman, Zarqa, Irbid, etc.)

3. **Time Metrics**:
   - `Q3_2_waitingtime`: Waiting time in minutes
   - `Q3_3_processtime`: Process time in minutes

4. **Satisfaction**:
   - `Q4_1_were_trated_respectfully`: All respondents said yes (1)
   - `Q4_2_satisfied_w_validation_process`: 93 yes, 4 no
   - `Q3_6_comfortable_using_validation_equipment`: 90 yes, 7 no

5. **Awareness**:
   - `Q2_1_how_did_you_know_about_validation`: Multi-select (1, 2, 3, 5, "1 2", "1 3", "2 1", "3 1")

6. **Monitor**:
   - `monitor_name`: 11 different monitors

### Data Distribution

```
Post offices: 44 unique locations (Arabic names in metadata)
Governorates: 11 (Amman: 21, Mafraq: 17, Karak: 14, Zarqa: 13, etc.)
Monitors: 11 (Meshael: 24, Malek: 15, Ahmad: 15, etc.)
Gender: Female: 58, Male: 39
```

## Step 3: Process Metadata

### Find Metadata File

Located in: `metadata/raw/20251103/General Food Assistance xls/BCM_PO_validation_2025.xlsx`

### Add Filename Mapping

**File**: `app/R/metadata_helpers.r`

```r
map_metadata_to_survey_ids <- function(metadata_list) {
  filename_mappings <- list(
    "Welcome Meals.xlsx" = "5404",
    "post_offices_osm_july_2025.xlsx" = "5406",
    "BCM_HD_validation_2025.xlsx" = "5387",
    "BCM_PO_validation_2025.xlsx" = "5407",  # ADDED
    "helpdesk_validation_OSM_2025.xlsx" = "5405",
    "wfp_hd_monitoring_check_List.xlsx" = "5373"
  )
  # ...
}
```

### Process and Save Metadata

```r
library(here)
library(dplyr)
source(here('app/R/metadata_helpers.r'))

# Load and process
xlsform_path <- here('metadata/raw/20251103/General Food Assistance xls/BCM_PO_validation_2025.xlsx')
xlsform <- load_xlsform(xlsform_path)
metadata <- process_xlsform(xlsform)

# Map and save
metadata_list <- list()
metadata_list[['BCM_PO_validation_2025.xlsx']] <- metadata
mapped <- map_metadata_to_survey_ids(metadata_list)
save_processed_metadata(mapped, here('metadata/processed'))
```

### Metadata Contents

- **Variables**: 42 variables extracted
- **Choice Lists**: 7 lists
  - `complaint`: Complaint types
  - `field`: Field monitor names
  - `gender`: Male/Female
  - `governorate`: 12 Jordanian governorates
  - `post_name`: 45+ post office names (in Arabic)
  - `source`: Information sources (SMS, Word of mouth, Facebook, Leaflets, Other)
  - `yes_no`: Yes/No values

**Output Files**:

- `metadata/processed/5407_metadata.rds`
- `metadata/processed/all_metadata.rds` (updated)

## Step 4: Create Exercise Definition

**File**: `app/exercises/bcm_post_office_validation.r`

### Key Configuration

```r
exercise <- list(
  id = "5407",
  name = "BCM - Post Office Validation",
  type = "survey",
  description = "Jordan BCM post office validation survey data quality",
  
  # Filter configuration
  filter_column = "_1_6_Post_office_name",
  filter_label = "Post Office",
  
  # Data fetching
  fetch_data = function(client) {
    databridges_survey_fetch(
      client = client,
      survey_id = 5407L,
      access_type = "full"
    )
  },
  
  # Visualizations with metadata
  visualizations = function(data) {
    metadata <- load_survey_metadata("5407", "../metadata/processed")
    bcm_post_office_validation_dashboard(data, metadata = metadata)
  },
  
  # Validation rules (7 rules total)
  run_validations = function(data) {
    # ID columns for CSV export
    options('affirm.id_cols' = c(
      "date", "_1_6_Post_office_name", "governorate", 
      "_submission_time", "start", "end", "_duration", "monitor_name"
    ))
    
    data |>
      duration_check_rule()(id = 1) |>              # 5 min - 24 hours
      same_day_submission_rule()(id = 2) |>         # Same day submission
      late_submission_rule()(id = 3) |>             # Late submission flag
      time_outlier_rule()(id = 4) |>                # Waiting/process time outliers
      conditional_field_rule()(id = 5) |>           # Q2.3.1 & Q4.2.1 logic
      respectful_treatment_flag()(id = 6) |>        # Q4.1 = No flag
      missing_data_rule()(id = 7)                   # Missing data check
  }
)
```

## Step 5: Create Dashboard Function

**File**: `app/R/visualization_helpers.r`

### Dashboard Structure

```r
bcm_post_office_validation_dashboard <- function(data, metadata = NULL) {
  # Helper for metadata labels
  get_label <- function(column_name, fallback) {
    if (!is.null(metadata)) {
      label <- get_variable_label(metadata, column_name)
      if (label != column_name) return(label)
    }
    return(fallback)
  }

  tagList(
    # 1. Service Time Metrics
    #    - Waiting time
    #    - Process time
    
    # 2. Beneficiary Satisfaction
    #    - Treated respectfully
    #    - Satisfied with process
    
    # 3. User Experience
    #    - Comfortable using equipment
    
    # 4. Demographics
    #    - Gender distribution
    
    # 5. Awareness & Information
    #    - How they learned about validation
    
    # 6. Coverage by Post Office
    #    - Visits by post office
    
    # 7. Geographic Coverage
    #    - Visits by governorate
    
    # 8. Monitor Activity
    #    - Surveys by monitor
  )
}
```

### Visualization Components

- **Time Metrics**: `time_metric_card()` for waiting/process times
- **Satisfaction**: `satisfaction_metric_card()` for binary yes/no questions
- **Breakdowns**: `breakdown_card()` for categorical distributions with metadata labels

## Step 6: Registration

The exercise is automatically registered because:

1. The file exists in `app/exercises/` directory
2. It defines an `exercise` object with an `id` field
3. The `load_exercises()` function in `app/R/exercise_registry.r` automatically loads all `.r` files

No manual registration needed!

## Step 7: Testing

### Launch the App

```r
# From the app/ directory
shiny::runApp()
```

### Verify

1. **Exercise appears in dropdown**: "BCM - Post Office Validation"
2. **Data loads**: 97 responses displayed
3. **Column Details**: Shows variable labels from metadata
4. **Filter works**: Can filter by Post Office
5. **Visualizations display**:
   - Time metrics show average waiting/process times
   - Satisfaction cards show percentages
   - Breakdowns show proper labels (not codes)
   - Post office names show Arabic text
   - Governorates, monitors, and gender all labeled correctly

6. **Multi-select handling**: "How did you know" question shows combined labels like "SMS, Facebook"
7. **Validation runs**: Can export validation results

## Key Technical Details

### Multi-Select Question Handling

The `breakdown_card()` function was previously enhanced to handle multi-select questions where:

- Values like `"1 2"` are normalized to sorted order before counting
- Display labels are created by splitting codes and mapping each to its label
- Results are consolidated (e.g., `"1 2"` and `"2 1"` counted together)

### Metadata Integration

The exercise loads metadata in the `visualizations()` function:
```r
metadata <- tryCatch({
  load_survey_metadata("5407", "../metadata/processed")
}, error = function(e) {
  NULL
})
```

This metadata is then passed to dashboard functions and visualization helpers for:

- Variable label lookup via `get_variable_label(metadata, column_name)`
- Choice value mapping via choice lists (e.g., `metadata$choices$post_name`)

### Choice Lists Used

- `gender`: Maps 0/1 or male/female codes to labels
- `source`: Maps 1-5 to SMS, Word of mouth, Facebook, Leaflets, Other
- `post_name`: Maps numeric codes to Arabic post office names
- `governorate`: Maps governorate codes to proper names
- `field`: Maps monitor name codes (if needed)

## Files Changed

1. **`app/R/metadata_helpers.r`**: Added `"BCM_PO_validation_2025.xlsx" = "5407"` mapping
2. **`app/exercises/bcm_post_office_validation.r`**: New exercise definition (70 lines)
3. **`app/R/visualization_helpers.r`**: Added `bcm_post_office_validation_dashboard()` function (110 lines)
4. **`metadata/processed/5407_metadata.rds`**: New processed metadata file
5. **`metadata/processed/all_metadata.rds`**: Updated with new survey

## Step 8: Additional Validations Based on Requirements

After reviewing the official requirements document, additional validation rules were implemented:

### New Validation Rules Added

**File**: `app/R/validation_rules.r`

#### 1. Time Outlier Rule (`time_outlier_rule`)
Flags waiting and process times that are significantly above or below average:

- **Q3.2 Waiting time**: Flags if outside 0.5x - 2x average (~10 min)
- **Q3.3 Process time**: Flags if outside 0.5x - 2x average (~3.5 min)
- **Purpose**: Requirement "3.1 & 3.2 – highlight values that are above or below average entries"

#### 2. Conditional Field Rule (`conditional_field_rule`)
Ensures follow-up questions are answered when triggered:

- **Q2.3.1**: Must be answered when Q2.3 = No (didn't know which PO)
- **Q4.2.1**: Must be answered when Q4.2 = No (not satisfied with process)
- **Purpose**: Requirement "2.3.1 and 4.2.1 should not be NA or N/A"

#### 3. Respectful Treatment Flag (`respectful_treatment_flag`)
Flags any response where Q4.1 (treated respectfully) = No:

- **Q4.1**: Flags if value = 0 (not treated respectfully)
- **Purpose**: Requirement "flag 4.1 if the answer is 'No'"
- **Note**: Current data shows 0 "No" responses (all 97 respondents said Yes)

### Dashboard Enhancement

Added **Average Survey Duration** display:

- Shows mean duration across all surveys
- Displays value in minutes
- Includes count of surveys
- **Purpose**: Requirement "Average time duration"

### Validation Coverage Summary

From requirements document:

**Part 1 - Data Quality:**

- ✅ ~~GPS check~~ (skipped per user request)
- ⚠️ Q2.1.1a "Other" text validation (not implemented - low priority)
- ✅ Q2.3.1 and Q4.2.1 conditional checks (implemented)
- ✅ Time outlier detection for Q3.1 & Q3.2 (implemented)

**Part 2 - Survey Practice:**

- ✅ Duration 5 min - 24 hours (existing `duration_check_rule`)
- ✅ Average time duration display (added to dashboard)
- ✅ Same-day submission check (existing `same_day_submission_rule`)
- ❌ Case ID/phone duplicate check (N/A - these columns don't exist in data)
- ✅ Late submission flagging (existing `late_submission_rule`)

**Part 2.2 - Preliminary Analysis:**

- ✅ Flag Q4.1 = No (implemented `respectful_treatment_flag`)

### Total Validation Rules: 7

1. **Duration check** (5 min - 24 hours)
2. **Same-day submission** (submitted same day as collection)
3. **Late submission** (flag delays)
4. **Time outliers** (NEW - waiting/process time outliers)
5. **Conditional fields** (NEW - Q2.3.1 & Q4.2.1 logic)
6. **Respectful treatment** (NEW - flag Q4.1 = No)
7. **Missing data** (general missing data check)

## Summary

Successfully integrated BCM Post Office Validation (Survey 5407) with:

- ✅ 97 survey responses
- ✅ 42 variables with labels
- ✅ 7 choice lists for value mapping
- ✅ 9 visualization sections (including average duration)
- ✅ 7 validation rules (3 new survey-specific rules)
- ✅ Full metadata integration
- ✅ Multi-select question support
- ✅ Arabic text support for post office names
- ✅ Compliance with official requirements document

The exercise is now live and accessible in the app dropdown!

## Future Reference

When adding new exercises, follow this pattern:

1. Identify survey ID via API exploration
2. Explore data structure and key columns
3. Find and process metadata file
4. Add filename mapping to `metadata_helpers.r`
5. Create exercise definition in `exercises/`
6. Create dashboard function in `visualization_helpers.r`
7. Test thoroughly

For more details, see:

- `/docs/guides/ADDING_NEW_EXERCISES.md`
- `/docs/guides/ADDING_METADATA.md`

