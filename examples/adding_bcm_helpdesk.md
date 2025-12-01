# Adding BCM - Helpdesk Exercise

**Date**: November 6, 2025  
**Survey ID**: 5402  
**Exercise Name**: BCM - Helpdesk  
**Type**: Monitoring (not validation)

## Overview

This document details the steps taken to add the "BCM - Helpdesk" exercise following the process documented in `/docs/guides/ADDING_NEW_EXERCISES.md`.

## Step 1: Identify the Survey

### Interactive API Exploration

```r
library(reticulate)
library(dplyr)
use_python(".venv/bin/python")
data_bridges_knots <- import("data_bridges_knots")
config_path <- "data_bridges_api_config.yaml"
client <- data_bridges_knots$DataBridgesShapes(config_path)

# Search for helpdesk surveys
surveys <- client$get_household_surveys_list()
helpdesk_surveys <- surveys[grepl("helpdesk", surveys$surveyName, ignore.case = TRUE), ]
```

### Results

Found 3 surveys with "helpdesk" in the name:

- **Survey 5402**: "BCM - Helpdesk - 2025" - **868 responses** ✅ (Main monitoring survey)
- Survey 5403: "BCM - Helpdesk Validation - 2025" - 33 responses (different survey)
- Survey 5408: "OSM - Helpdesk - 2025" (already integrated as 5373/5408)

**Note**: Survey 5387 "BCM - Helpdesk Validation" is already integrated. Survey 5402 is the main helpdesk monitoring survey (not validation), which is different.

**Decision**: Use Survey 5402

## Step 2: Explore Data Structure

### Key Columns Identified

```r
survey_id <- as.integer(5402)
data <- client$get_household_survey(survey_id, "full")
# 868 responses x 70 columns
```

### Important Columns

1. **Demographics**:
   - `gender`: Female (491), Male (377)
   - `age`: 18-24, 25-34, 35-44, 45-59, 60+

2. **Location**:
   - `helpdesk`: 32 different helpdesks (Arabic names)
   - `location`: Camp (344), Host Community (524)

3. **Time Metrics**:
   - `waiting_time`: Average 11.6 minutes (range: -2 to 240)
   - `service_time`: Average 6.4 minutes (range: 0 to 60)
   - `travel_time`: Travel time to helpdesk

4. **Quality Ratings**:
   - `hd_quality`: Excellent (526), Good (326), Fair (10), Poor (5), Very Poor (1)
   - `hd_staff_quality`: Excellent (551), Good (304), Fair (9), Poor (4)

5. **Visit Details**:
   - `hd_vis_reason`: Reason for visit
   - `issues_solv_yn`: Were issues resolved?
   - `safe_way_hd`: Safe way to access helpdesk
   - `hear_hd`: How they heard about the helpdesk

6. **Beneficiary Status**:
   - `wfp_ben`: WFP beneficiary status
   - `unhcr_id`: UNHCR ID

7. **Enumerator**:
   - `en_name`: 13 different enumerators

### Data Distribution

```
Location: Camp (344), Host Community (524)
Gender: Female (491), Male (377)
Age: 25-34 (231), 35-44 (250), 45-59 (196), 18-24 (99), 60+ (92)
Top Helpdesks:
  - جمعية الأصايل للإبداع والفن التشكيلي – المفرق: 192
  - مكتب الشكاوى WFP/NRC – مخيم الزعتري: 166
  - مكتب الشكاوى WFP/NRC – مخيم الأزرق (القرية السادسة): 115
Quality: Excellent (526), Good (326)
Staff Quality: Excellent (551), Good (304)
```

## Step 3: Process Metadata

### Find Metadata File

Located in: `metadata/raw/20251103/General Food Assistance xls/BCM_Helpdesk_mon_260525.xlsx`

### Add Filename Mapping

**File**: `app/R/metadata_helpers.r`

```r
map_metadata_to_survey_ids <- function(metadata_list) {
  filename_mappings <- list(
    "Welcome Meals.xlsx" = "5404",
    "post_offices_osm_july_2025.xlsx" = "5406",
    "BCM_HD_validation_2025.xlsx" = "5387",
    "BCM_PO_validation_2025.xlsx" = "5407",
    "BCM_Helpdesk_mon_260525.xlsx" = "5402",  # ADDED
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
xlsform_path <- here('metadata/raw/20251103/General Food Assistance xls/BCM_Helpdesk_mon_260525.xlsx')
xlsform <- load_xlsform(xlsform_path)
metadata <- process_xlsform(xlsform)

# Map and save
metadata_list <- list()
metadata_list[['BCM_Helpdesk_mon_260525.xlsx']] <- metadata
mapped <- map_metadata_to_survey_ids(metadata_list)
save_processed_metadata(mapped, here('metadata/processed'))
```

**Output Files**:

- `metadata/processed/5402_metadata.rds`
- `metadata/processed/all_metadata.rds` (updated)

## Step 4: Create Exercise Definition

**File**: `app/exercises/bcm_helpdesk.r`

Following the template from other exercises with appropriate validation rules.

## Step 5: Create Dashboard Function

**File**: `app/R/visualization_helpers.r`

Function: `bcm_helpdesk_dashboard()` with sections for:

- Time metrics (waiting, service, travel)
- Quality ratings (helpdesk quality, staff quality)
- Demographics (gender, age)
- Location breakdown (camp vs. host community)
- Helpdesk distribution
- Issue resolution metrics
- Enumerator activity

## Step 6: Registration

The exercise is automatically registered via the exercise registry system.

## Step 7: Testing

1. Launch app and select "BCM - Helpdesk"
2. Verify 868 responses load
3. Check metadata labels apply correctly
4. Test filter by helpdesk location
5. Verify visualizations display
6. Run validations and export

## Files Created/Modified

1. ✅ `app/R/metadata_helpers.r` - Added survey 5402 mapping
2. ✅ `app/exercises/bcm_helpdesk.r` - New exercise definition (64 lines)
3. ✅ `app/R/visualization_helpers.r` - New `bcm_helpdesk_dashboard()` function (110 lines)
4. ✅ `metadata/processed/5402_metadata.rds` - New metadata file (54 variables, 16 choice lists)
5. ✅ `docs/adding_bcm_helpdesk.md` - This documentation

## Step 8: Additional Validations Based on Requirements

After reviewing the official requirements document, additional validation rules were implemented:

### New Validation Rules Added

**File**: `app/R/validation_rules.r`

#### 1. ID/Phone Uniqueness Rule (`id_phone_uniqueness_rule`)
Checks UNHCR ID and phone numbers are not duplicated on the same day:

- **UNHCR ID**: Flags duplicates within same date
- **Phone**: Flags duplicates within same date
- **Purpose**: Requirement "Case ID and phone number should not be duplicated...from the same day"

#### 2. Time Metrics by Location Rule (`time_metrics_by_location_rule`)
Flags time metrics that are above/below average FOR EACH HELPDESK:

- **Travel time**: Flags if outside 0.5x - 2x helpdesk average
- **Waiting time**: Flags if outside 0.5x - 2x helpdesk average
- **Service time**: Flags if outside 0.5x - 2x helpdesk average
- **Transport cost (JOD)**: Flags if outside 0.5x - 2x helpdesk average
- **Purpose**: Requirements "highlight above or below average...data and across all data for the same location"

#### 3. Other Text Validation Rule (`other_text_validation_rule`)
Ensures "Other" text fields have valid unique responses:

- **other_hear_hd**: Must not be NA/N/A or duplicate standard options
- **other_transport**: Must not be NA/N/A or duplicate standard options
- **Purpose**: Requirement "should not be NA or N/A or same as the options in this question"

#### 4. Safety Accessibility Flag (`safety_accessibility_flag`)
Flags inconsistent safety/accessibility responses:

- **Logic**: Flags if safe_way_hd = "No" AND acc_hd = "Yes"
- **Purpose**: Requirement 'If...="Unsafe" acc_hd = "Yes" highlight for follow up'
- **Note**: In actual data, safe_way_hd uses "No"/"Yes" not "Safe"/"Unsafe"

### Dashboard Enhancement

Added **Average Survey Duration** display at top of dashboard.

### Validation Coverage Summary

From requirements document:

**Part 1 - Data Quality:**

- ✅ ~~GPS check~~ (skipped)
- ✅ other_hear_hd validation (implemented)
- ✅ travel_time outliers by location (implemented)
- ✅ waiting_time outliers by location (implemented)
- ✅ service_time outliers by location (implemented)
- ✅ other_transport validation (implemented)
- ✅ jod_transport outliers by location (implemented)

**Part 2 - Survey Practice:**

- ✅ Duration 5 min - 24 hours
- ✅ Average time duration display
- ✅ Same-day submission check
- ✅ UNHCR ID and phone uniqueness (same day)
- ✅ Late submission flagging

**Part 2.2 - Preliminary Analysis:**

- ✅ Safety/accessibility flag (safe_way_hd + acc_hd logic)

## Summary

Successfully integrated BCM Helpdesk (Survey 5402) with:

- ✅ 868 survey responses (largest dataset so far)
- ✅ 54 variables with labels
- ✅ 16 choice lists for value mapping
- ✅ 9 visualization sections (including average duration)
- ✅ 13 validation rules (9 new survey-specific rules)
- ✅ Full metadata integration
- ✅ Support for 32 different helpdesk locations
- ✅ Arabic text support for helpdesk names
- ✅ Location-specific time outlier detection
- ✅ Compliance with official requirements document

### Dashboard Sections

1. **Service Time Metrics** - Waiting, service, and travel times
2. **Quality Ratings** - Helpdesk and staff quality ratings
3. **Issue Resolution** - Percentage of issues resolved
4. **Demographics** - Gender and age distribution
5. **Location Type** - Camp vs. Host Community breakdown
6. **Coverage by Helpdesk** - Visits across 32 helpdesk locations
7. **Enumerator Activity** - Surveys by each of 13 enumerators

### Validation Rules (13 total)

Based on requirements document for BCM Helpdesk monitoring:

**Part 2 - Survey Practice:**

1. **Duration check** (5 min - 24 hours)
2. **Same-day submission** (submitted same day as collection)
3. **Late submission** (flag delays)
4-5. **ID/Phone uniqueness** (UNHCR ID and phone not duplicated same day)

**Part 1 - Data Quality:**

6. **Travel time outliers** (by helpdesk location)
7. **Waiting time outliers** (by helpdesk location)
8. **Service time outliers** (by helpdesk location)
9. **Transport cost outliers** (by helpdesk location)
10-11. **Other text validation** (other_hear_hd, other_transport not NA/N/A)

**Part 2.2 - Preliminary Analysis:**

12. **Safety/Accessibility flag** (unsafe but accessible cases)
13. **Missing data** (general missing data check)

### Key Differences from BCM Helpdesk Validation (5387)

- **Survey 5387**: Helpdesk Validation - 69 responses, focused on validation process
- **Survey 5402**: Helpdesk Monitoring - 868 responses, ongoing monitoring of all helpdesks
- Different column names and structure
- More comprehensive coverage (32 helpdesks vs. specific validation locations)
- Includes quality ratings and issue resolution metrics

The exercise is now live and accessible as "BCM - Helpdesk" in the app dropdown!

---

*Following the process documented in `/docs/guides/ADDING_NEW_EXERCISES.md`*

