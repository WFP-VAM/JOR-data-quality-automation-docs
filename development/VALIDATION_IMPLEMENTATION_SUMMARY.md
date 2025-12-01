# Data Quality Validation Implementation Summary

## Overview

Implemented comprehensive, requirement-driven data quality checks using the [**affirm**](https://pcctc.github.io/affirm/) package for validation and reporting.

---

## ✅ What Was Implemented

### BCM - Helpdesk Validation (Survey 5387)

| # | Check | Status | Details |
|---|-------|--------|---------|
| 1 | Duration bounds | ✅ Implemented | 5 min minimum, 24 hours maximum |
| 2 | Same-day submission | ✅ Implemented | Compares submission date with collection date |
| 3 | Case ID uniqueness | ✅ Implemented | No duplicates within same day |
| 4 | Phone number uniqueness | ✅ Implemented | No duplicates within same day |
| 5 | Late submission | ✅ Implemented | Flags submissions >24 hours after visit |
| 6 | Missing data | ✅ Implemented | Flags rows with >50% missing values |

**Total**: 6 validation rules

### BCM - Welcome Meals (Survey 5404)

| # | Check | Status | Details |
|---|-------|--------|---------|
| 1 | Duration bounds | ✅ Implemented | 5 min - 1 hour (shorter surveys) |
| 2 | Same-day submission | ✅ Implemented | Compares submission date with collection date |
| 3 | Late submission | ✅ Implemented | Flags submissions >24 hours after visit |
| 4 | Beneficiary ID uniqueness | ✅ Implemented | No duplicates within same day |
| 5 | Phone number uniqueness | ✅ Implemented | No duplicates within same day |
| 6 | Complaint rate check | ✅ Implemented | Must be ≤10% of entries |
| 7 | Missing data | ✅ Implemented | Flags rows with >30% missing values |

**Total**: 7 validation rules

### OSM - Helpdesk Monitoring (Survey 5373)

| # | Check | Status | Details |
|---|-------|--------|---------|
| 1 | Duration bounds | ✅ Implemented | 5 min minimum, 24 hours maximum |
| 2 | Same-day submission | ✅ Implemented | Compares submission date with collection date |
| 3 | Late submission | ✅ Implemented | Flags submissions >24 hours after visit |
| 4 | Visit frequency | ✅ Implemented | 1x/month (host), 2x/month (camps) |
| 5 | Overcrowding | ✅ Implemented | Must be ≤10% of entries |
| 6 | Missing data | ✅ Implemented | Flags rows with >50% missing values |

**Total**: 6 validation rules

---

## 📊 Validation Rule Details

### Duration Check

- **Columns**: `_duration` (seconds), `start`, `end` (when available)
- **Logic**: Must be between 300s (5 min) and 86400s (24 hours), configurable per exercise
- **Purpose**: Flag suspiciously quick or unreasonably long surveys
- **CSV Export**: Includes `start`, `end`, `_duration` for verification

### Same-Day Submission

- **Columns**: `date`, `_submission_time`
- **Logic**: Extract date from ISO timestamp, compare with `date` field
- **Example**: If `date = "2025-10-05"` and `_submission_time = "2025-10-05T14:11:03..."` → ✅
- **Example**: If `date = "2025-10-05"` and `_submission_time = "2025-10-06T10:00:00..."` → ❌

### Uniqueness Checks (BCM exercises)

- **Case ID**: `Case_ID` or `Beneficiary_ID` format
- **Phone Number**: `Interviewee_Phone_Number` numeric values
- **Logic**: Group by `date`, check for duplicates within each day
- **Why**: Same beneficiary shouldn't be surveyed multiple times per day

### Late Submission

- **Logic**: Calculate `difftime()` between submission timestamp and end of visit day
- **Threshold**: 24 hours (configurable)
- **Example**: Visit on Oct 5, submitted Oct 7 at 10am → ❌ (late by ~34 hours)

### Visit Frequency (OSM only)

- **Column**: `location` (Arabic text)
- **Logic**:
  - Extract year-month from `date`
  - Count visits per location per month
  - Detect camps by keywords: "camp" or "مخيم"
  - Host community: max 1 visit/month
  - Camps: max 2 visits/month

### Proportion Checks

- **Columns**: `overcrowding`, `has_complaint`, etc.
- **Logic**: Calculate proportion of "Yes" responses
- **Threshold**: ≤10% (configurable)

---

## 🔧 Technical Implementation

### Using affirm API

Each validation uses the affirm package pattern:

```r
run_validations = function(data) {
  # Initialize affirm session
  affirm::affirm_init(replace = TRUE)
  
  # Configure CSV export columns
  options('affirm.id_cols' = c("ID", "date", "enumerator", ...))
  
  # Run validations in pipeline
  result <- data |>
    duration_check_rule()(id = 1, data_name = "Exercise Name") |>
    same_day_submission_rule()(id = 2, data_name = "Exercise Name") |>
    # ... more validations
  
  # Clean up helper columns
  result |> dplyr::select(-dplyr::starts_with("."))
}
```

### Key affirm Functions

1. **`affirm_init(replace = TRUE)`**
   - Initializes a new validation session
   - `replace = TRUE` ensures a fresh start

2. **`affirm_true(data, label, condition, id, data_frames)`**
   - Core validation function
   - `label`: User-friendly description
   - `condition`: Logical expression to test
   - `id`: Unique validation identifier
   - `data_frames`: Dataset name for organizing results

3. **`affirm_report_gt()`**
   - Generates professional gt-based report
   - Shows all violations (not just samples)
   - Provides CSV download links
   - Displays in Shiny app

### Validation Rule Structure

Each rule in `validation_rules.r` returns a function:

```r
duration_check_rule <- function(duration_column = "_duration",
                                min_seconds = 300,
                                max_seconds = 86400) {
  function(data, id, data_name = "Survey Data") {
    if (duration_column %in% names(data)) {
      affirm::affirm_true(
        data,
        label = sprintf("Survey duration between %d min and %d hours when recorded", 
                       min_seconds/60, max_seconds/3600),
        condition = is.na(.data[[duration_column]]) | 
                   .data[[duration_column]] == "" | 
                   (as.numeric(.data[[duration_column]]) >= min_seconds & 
                    as.numeric(.data[[duration_column]]) <= max_seconds),
        id = id,
        data_frames = data_name
      )
    } else {
      data
    }
  }
}
```

### CSV Export Configuration

The `affirm.id_cols` option determines which columns appear in violation CSV exports:

```r
# BCM Helpdesk example
options('affirm.id_cols' = c(
  "Case_ID",                    # Record ID
  "date",                       # Collection date
  "_1_6_Helpdesk_name",        # Filter/grouping column
  "_submission_time",           # Submission timestamp
  "start", "end", "_duration",  # Timing columns
  "Interviewee_Phone_Number"    # Validation-specific column
))
```

**Best Practice**: Include ALL columns used in validations, plus ID, date, and filter columns.

---

## 📈 How Validations Appear in App

When you select an exercise in the app:

1. **affirm Report** appears showing:
   ```
   Validation Report: Exercise Name
   Summary: X passed  Y failed
   
   ✓ Validation 1 description
     All records passed this validation
   
   ✗ Validation 2 description
     N record(s) failed this validation
     [Show violated records (N)]
     [CSV download link]
   ```

2. **Violated Records** are expandable:
   - Click "Show violated records" to see table
   - All violated rows displayed (no sampling)
   - CSV download includes all violation data

3. **Monitoring Visualizations** appear below (if defined)
   - Time metrics, satisfaction scores, etc.

---

## 📁 Files Modified

| File | Changes |
|------|---------|
| `app/R/validation_rules.r` | Rewritten to use affirm instead of assertr |
| `app/exercises/*.r` | All exercises updated with `run_validations()` function |
| `app/app.r` | Updated to use `affirm_report_gt()` for rendering |
| `app/R/custom_validation_reporter.r` | Removed (no longer needed) |
| `docs/guides/ADDING_NEW_EXERCISES.md` | Updated with affirm workflow |
| `renv.lock` | Removed assertr/data.validator, added affirm and gt |

---

## 🧪 Testing

### Verification Commands

```r
# Load exercises
library(dplyr)
library(affirm)
source("app/R/validation_helpers.r")
source("app/R/exercise_registry.r")
exercises <- load_exercises()

# Check that run_validations exists
names(exercises[['5373']])  # Should include "run_validations"
names(exercises[['5387']])  # Should include "run_validations"
```

### Manual Testing

1. Run app: `shiny::runApp("app")`
2. Select an exercise
3. View validation report
4. Check for:
   - ✅ Green checkmarks for passing validations
   - ❌ Red Xs for failing validations
   - Expandable violated records section
   - CSV download links for failed validations

5. Download CSV and verify:
   - Contains violated rows
   - Includes all configured ID columns
   - Includes timing and context columns

---

## 🎯 Key Improvements with affirm

### Before (assertr/data.validator)

- ❌ Technical error messages: "violates assertion 'in_set(TRUE)'"
- ❌ Only shows 6 sample violations
- ❌ Complex error parsing required
- ❌ CSV exports were empty or broken
- ❌ Required custom reporter code

### After (affirm)

- ✅ User-friendly messages: "Survey duration between 5 min and 24 hours when recorded"
- ✅ Shows ALL violated records
- ✅ Built-in error tracking and reporting
- ✅ CSV exports work out-of-the-box with full data
- ✅ Professional gt-based reports included
- ✅ Simpler, cleaner code

---

## 🔮 Future Enhancements

### 1. Historical Data Comparison (OSM)
```r
# Compare current visit data with historical records
# Flag: improvement, degradation, or data entry error
```

**Requirements:**

- Database or file storage for historical visit data
- Location matching logic (handle Arabic text variations)
- Change classification rules

### 2. Dynamic Thresholds

- Make validation thresholds configurable per exercise
- Store in exercise metadata or config file
- Allow users to adjust thresholds via UI

### 3. Trend Reporting

- Track validation failure rates over time
- Identify problematic locations or enumerators
- Generate monthly quality reports

### 4. Real-time Alerts

- Integrate with email/Slack for critical failures
- Use RStudio Connect scheduler
- Send daily summaries to data quality team

---

## 📚 Resources

- **affirm Package**: https://pcctc.github.io/affirm/
- **affirm Getting Started**: https://pcctc.github.io/affirm/articles/getting-started.html
- **gt Package**: https://gt.rstudio.com/
- **Mastering Shiny**: https://mastering-shiny.org/
- **Project Documentation**: `docs/guides/ADDING_NEW_EXERCISES.md`

---

## ✨ Key Achievements

1. ✅ **Requirement-driven**: All validation rules map directly to user requirements
2. ✅ **User-friendly**: Clear, actionable error messages
3. ✅ **Complete data**: Shows ALL violations, not just samples
4. ✅ **CSV exports**: Full violation data downloadable for investigation
5. ✅ **Robust**: Handles missing columns and edge cases gracefully
6. ✅ **Documented**: Comprehensive documentation for maintainability
7. ✅ **Tested**: Verified all exercises load and validate correctly
8. ✅ **Production-ready**: Following affirm best practices

---

**Framework**: affirm v0.2.1  
**Status**: ✅ Complete and ready for deployment  
**Date**: 2025-10-31
