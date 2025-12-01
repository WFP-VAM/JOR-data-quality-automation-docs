# Metadata Integration Summary

## Overview

This document describes the metadata integration system for the JOR Data Quality Automation app. The system loads XLSForm metadata (data dictionaries) and uses them to enhance the user interface with human-readable labels.

## What Was Implemented

### 1. Metadata Processing Pipeline

**Files Created:**

- `app/R/metadata_helpers.r` - Core metadata loading and processing functions
- `tools/process_bcm_helpdesk_metadata.r` - Script to process BCM Helpdesk metadata
- `tools/test_metadata_integration.r` - Integration test script
- `metadata/processed/5387_metadata.rds` - Processed metadata for BCM Helpdesk (4.4 KB)
- `metadata/processed/all_metadata.rds` - Combined metadata for all surveys

**Functions:**

- `load_xlsform()` - Load XLSForm Excel files
- `load_all_xlsforms()` - Load all metadata from a directory
- `process_xlsform()` - Extract variables and choice lists
- `extract_variable_labels()` - Get variable name → label mappings
- `extract_choice_lists()` - Get value → label mappings for categorical variables
- `map_metadata_to_survey_ids()` - Map metadata files to survey IDs
- `save_processed_metadata()` - Save to compressed RDS files
- `load_survey_metadata()` - Load metadata for one survey
- `load_all_metadata()` - Load all processed metadata
- `get_variable_label()` - Lookup variable label
- `get_choice_label()` - Lookup choice label
- `label_variables()` - Apply labels to data frames

### 2. UI Enhancements

**Column Details Section:**

- Updated `column_info_table()` in `app/R/ui_components.r`
- Now displays a third column with variable labels when metadata is available
- Labels show human-readable descriptions instead of just column names
- Example: `gender` → "Gender of Respondent"

**Gender Distribution Visualization:**

- Updated `breakdown_card()` in `app/R/visualization_helpers.r`
- Added `metadata` and `choice_list_name` parameters
- Now displays "Male" and "Female" instead of raw codes
- Works for any categorical variable with choice lists

### 3. App Integration

**Main App (`app/app.r`):**

- Loads `metadata_helpers.r` at startup
- Loads all metadata into `all_metadata` variable
- Passes metadata to `column_info_table()` for display
- Handles missing metadata gracefully with try/catch

**BCM Helpdesk Exercise (`app/exercises/bcm_helpdesk_validation.r`):**

- Updated to load metadata in `visualizations` function
- Passes metadata to `bcm_monitoring_dashboard()`
- Gender breakdown now shows labeled values

## Data Structure

### XLSForm Format

Raw metadata files contain three sheets:

1. **survey** - Variable definitions
   ```
   type              | name   | label
   select_one gender | gender | Gender of Respondent
   ```

2. **choices** - Value labels
   ```
   list_name | name   | label
   gender    | Male   | Male
   gender    | Female | Female
   ```

3. **settings** - Form metadata
   ```
   form_title                  | version
   BCM - Helpdesk Validation   | 2025-05-26
   ```

### Processed Format

After processing, metadata is stored as:

```r
list(
  variables = data.frame(
    name = c("gender", "date", ...),
    label = c("Gender of Respondent", "Select the Date of the Visit", ...)
  ),
  choices = list(
    gender = list(
      "Male" = "Male",
      "Female" = "Female"
    ),
    yesno = list(
      "Yes" = "Yes",
      "No" = "No"
    )
  ),
  settings = data.frame(...),
  source_file = "BCM_Helpdesk_mon_260525.xlsx"
)
```

## Survey ID Mappings

Current mappings (defined in `map_metadata_to_survey_ids()`):

| Filename | Survey ID | Survey Name |
|----------|-----------|-------------|
| `Welcome Meals.xlsx` | 5404 | BCM - Welcome Meals |
| `post_offices_osm_july_2025.xlsx` | 5406 | OSM - Post Office Validation |
| `BCM_Helpdesk_mon_260525.xlsx` | 5387 | BCM - Helpdesk Validation |
| `helpdesk_validation_OSM_2025.xlsx` | 5405 | OSM - Helpdesk |

## File Size Comparison

**Before Processing:**

- `BCM_Helpdesk_mon_260525.xlsx`: ~50 KB (Excel format with all sheets)

**After Processing:**

- `5387_metadata.rds`: 4.4 KB (binary R format, only relevant data)
- **88% size reduction**, near-instant loading in app

## Usage Examples

### In Exercise Definitions

```r
visualizations = function(data) {
  # Load metadata for this exercise
  metadata <- load_survey_metadata("5387", "../metadata/processed")
  
  # Pass to dashboard
  bcm_monitoring_dashboard(data, metadata = metadata)
}
```

### In Visualization Functions

```r
# Use variable label
var_label <- get_variable_label(metadata, "gender")

# Use choice labels in breakdown
breakdown_card(
  data, 
  "gender", 
  "Gender Distribution",
  metadata = metadata,
  choice_list_name = "gender"
)
```

### Manual Lookup

```r
# Load metadata
metadata <- load_survey_metadata("5387")

# Get variable label
get_variable_label(metadata, "gender")
# Returns: "Gender of Respondent"

# Get choice label
get_choice_label(metadata, "gender", "Male")
# Returns: "Male"
```

## Testing

Run the integration test:

```r
source("tools/test_metadata_integration.r")
```

Expected output:

- ✓ Metadata loading works
- ✓ Variable label retrieval works
- ✓ Choice label retrieval works
- ✓ Gender choice list is properly structured
- ✓ All metadata loading works

## Benefits

1. **Better UX**: Users see "Gender of Respondent" instead of cryptic column names
2. **Labeled Values**: "Male" and "Female" instead of codes or raw values
3. **Fast Loading**: 4.4 KB RDS files load instantly vs. parsing Excel files
4. **Maintainability**: Metadata updates don't require code changes
5. **Consistency**: Same labels in app as in original survey definitions
6. **Scalability**: Easy to add metadata for new surveys

## Next Steps

To add metadata for other surveys:

1. Download XLSForm from SharePoint
2. Place in `metadata/raw/YYYYMMDD/`
3. Add mapping in `map_metadata_to_survey_ids()`
4. Run `source("tools/process_metadata.r")` (or create a survey-specific script)
5. Update exercise definition to use metadata
6. Update visualizations to use choice labels

## References

- [readxl package](https://readxl.tidyverse.org/)
- [labelled package](https://larmarange.github.io/labelled/articles/labelled.html)
- [XLSForm Standard](https://xlsform.org/)
- Metadata README: `/metadata/README.md`

