# Adding Metadata to Exercises

This guide shows you how to integrate metadata (variable labels and choice lists) into your exercises.

## Quick Start

Follow these 4 steps to add metadata to an exercise:

### 1. Download and Place Raw Metadata

Download the XLSForm file from SharePoint and place it in:
```
metadata/raw/YYYYMMDD/[category]/
```

Example: `metadata/raw/20251103/General Food Assistance xls/Welcome Meals.xlsx`

### 2. Add Survey ID Mapping

Edit `app/R/metadata_helpers.r` and add your mapping in `map_metadata_to_survey_ids()`:

```r
filename_mappings <- list(
  "Welcome Meals.xlsx" = "5404",
  "BCM_Helpdesk_mon_260525.xlsx" = "5387",
  "your_survey.xlsx" = "YOUR_SURVEY_ID"  # Add this line
)
```

### 3. Process the Metadata

Create a processing script (or modify existing):

```r
# tools/process_your_survey_metadata.r
library(here)
source(here("app/R/metadata_helpers.r"))

# Load and process
xlsform <- load_xlsform(here("metadata/raw/.../your_survey.xlsx"))
metadata <- process_xlsform(xlsform)

# Save
saveRDS(metadata, here("metadata/processed/YOUR_SURVEY_ID_metadata.rds"))

# Update combined file
all_metadata <- load_all_metadata(here("metadata/processed"))
all_metadata[["YOUR_SURVEY_ID"]] <- metadata
saveRDS(all_metadata, here("metadata/processed/all_metadata.rds"))
```

Run it:
```bash
Rscript tools/process_your_survey_metadata.r
```

### 4. Update Exercise Definition

Edit your exercise file (e.g., `app/exercises/your_exercise.r`):

```r
visualizations = function(data) {
  # Load metadata for this exercise
  metadata <- tryCatch({
    load_survey_metadata("YOUR_SURVEY_ID", "../metadata/processed")
  }, error = function(e) {
    NULL
  })
  
  # Pass to your dashboard
  your_dashboard(data, metadata = metadata)
}
```

## Using Metadata in Visualizations

### Update Dashboard Function Signature

```r
your_dashboard <- function(data, metadata = NULL) {
  # ... dashboard code ...
}
```

### Add Variable Labels to Breakdown Cards

```r
# Before - shows raw values
breakdown_card(data, "status", "Application Status")

# After - shows labeled values
breakdown_card(
  data, 
  "status", 
  "Application Status",
  metadata = metadata,
  choice_list_name = "application_status"  # Name from XLSForm choices sheet
)
```

### Get Individual Labels

```r
# Get variable label
label <- get_variable_label(metadata, "gender")
# Returns: "Gender of Respondent"

# Get choice label
label <- get_choice_label(metadata, "yesno", "Yes")
# Returns: "Yes" (or localized version if available)
```

## Finding Choice List Names

Choice list names come from the `list_name` column in the XLSForm `choices` sheet.

Example XLSForm:

**survey sheet:**
```
type                   | name   | label
select_one yesno       | consent| Do you consent?
select_one app_status  | status | Application Status
```

**choices sheet:**
```
list_name  | name      | label
yesno      | Yes       | Yes
yesno      | No        | No
app_status | pending   | Pending Review
app_status | approved  | Approved
app_status | rejected  | Rejected
```

In this case:

- Use `choice_list_name = "yesno"` for the consent field
- Use `choice_list_name = "app_status"` for the status field

### Note: `name` vs `value` Column

Some XLSForm files use `value` instead of `name` in the choices sheet. The metadata processing functions handle both automatically:

- **Standard format**: Uses `name` column for choice values
- **Alternative format**: Uses `value` column for choice values (e.g., `gfa_sbcc_endline_2025_v2.xls`)

Both formats are supported - the processing function will detect and use whichever column is present.

## Verifying Metadata

### Check What's in the Metadata

```r
source("app/R/metadata_helpers.r")
metadata <- load_survey_metadata("YOUR_SURVEY_ID")

# See all variables
View(metadata$variables)

# See all choice lists
names(metadata$choices)

# See specific choice list
metadata$choices$your_list_name
```

### Test Script

```r
# Test loading
metadata <- load_survey_metadata("YOUR_SURVEY_ID", "metadata/processed")
print(paste("Variables:", nrow(metadata$variables)))
print(paste("Choice lists:", length(metadata$choices)))

# Test specific lookups
print(get_variable_label(metadata, "your_column"))
print(get_choice_label(metadata, "your_list", "your_value"))
```

## Common Issues

### Issue: Metadata Not Loading

**Symptom:** Variable labels don't appear, gender still shows codes

**Solutions:**

1. Check file exists: `file.exists("metadata/processed/YOUR_ID_metadata.rds")`
2. Check survey ID mapping in `map_metadata_to_survey_ids()`
3. Verify `all_metadata.rds` was updated
4. Restart Shiny app

### Issue: Choice Labels Not Showing

**Symptom:** Still seeing raw values instead of labels

**Solutions:**

1. Verify choice list name matches XLSForm: `names(metadata$choices)`
2. Check data values match choice names exactly (case-sensitive!)
3. Ensure metadata is being passed to `breakdown_card()`
4. Check that dashboard function accepts `metadata` parameter

### Issue: Wrong Column Name

**Symptom:** `if (!column %in% names(data))` returns NULL

**Solutions:**

1. Check actual column names in data: `names(data)`
2. XLSForm `name` might differ from API column name
3. Some columns might be auto-prefixed (e.g., `Q2_1_gender` vs `gender`)
4. Use `grep("gender", names(data), value = TRUE)` to find it

### Issue: Choice Lists Not Extracted (0 choice lists found)

**Symptom:** Metadata processes successfully but shows 0 choice lists

**Possible causes:**

1. Choices sheet uses `value` column instead of `name` column
2. Missing `list_name` column in choices sheet
3. All rows have NA values in `list_name` or value columns

**Solutions:**

1. Check choices sheet structure: `read_excel("your_file.xlsx", sheet = "choices")`
2. Verify column names: Should have `list_name` and either `name` or `value`
3. The processing function now handles both `name` and `value` columns automatically
4. If still not working, check for empty rows or formatting issues in Excel file

## Complete Example

See `app/exercises/bcm_helpdesk_validation.r` for a complete working example:

```r
exercise <- list(
  id = "5387",
  name = "BCM - Helpdesk Validation",
  
  # ... other fields ...
  
  visualizations = function(data) {
    # Load metadata
    metadata <- tryCatch({
      load_survey_metadata("5387", "../metadata/processed")
    }, error = function(e) {
      NULL
    })
    
    # Pass to dashboard
    bcm_monitoring_dashboard(data, metadata = metadata)
  }
)
```

And in `app/R/visualization_helpers.r`:

```r
bcm_monitoring_dashboard <- function(data, metadata = NULL) {
  # ... other visualizations ...
  
  # Gender breakdown with labels
  breakdown_card(
    data, 
    "gender",                    # Column name in data
    "Gender Distribution",       # Display title
    metadata = metadata,         # Metadata object
    choice_list_name = "gender"  # Choice list from XLSForm
  )
}
```

## Best Practices

1. **Always handle missing metadata gracefully** - Use `tryCatch()` and default to NULL
2. **Keep metadata processing separate** - Don't process in app, use preprocessed RDS files
3. **Use descriptive choice list names** - Match them to XLSForm conventions
4. **Test with actual data** - Verify codes in data match XLSForm choice names
5. **Update combined file** - Always update `all_metadata.rds` after processing new surveys
6. **Document mappings** - Add comments explaining non-obvious survey ID mappings

## References

- Full integration details: `docs/development/METADATA_INTEGRATION.md`
- Metadata helpers API: `app/R/metadata_helpers.r`
- Example usage: `tools/examples/explore_metadata.r`
- Test script: `tools/test_metadata_integration.r`

