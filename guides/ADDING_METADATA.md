# Adding Metadata to Exercises

This guide shows you how to integrate metadata (variable labels and choice lists) into your exercises.

## Quick Start

Follow these 4 steps to add metadata to an exercise:

### 1. Download and Place Raw Metadata

Download the XLSForm file from SharePoint and place it in:
```
metadata/raw/YYYYMMDD/[category]/
```

Example: `metadata/raw/20251201/General Food Assistance xls/WFP - FSOM Q3 2025 - Script.xlsx`

### 2. Add Survey ID Mapping

Edit `app/R/metadata_helpers.r` and add your mapping in `map_metadata_to_survey_ids()`:

```r
filename_mappings <- list(
  "Welcome Meals.xlsx" = "5404",
  "BCM_Helpdesk_mon_260525.xlsx" = "5387",
  "hm_schools_hc_20250320.xlsx" = "5422",
  "hm_schools_hc_20250819_camps.xlsx" = "5423",
  "WFP - FSOM Q3 2025 - Script.xlsx" = "5485",
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

## GPS Coordinates Metadata

GPS coordinates are stored separately from survey metadata and are used for location validation. This allows you to validate that GPS coordinates entered during data collection match the location selected from a dropdown menu.

### GPS Coordinates File Structure

GPS coordinates are stored in `metadata/raw/GPS coordinates.xlsx` with multiple sheets:

- **HelpDesk** sheet: Contains helpdesk locations with columns:
  - `list_name`: Filter identifier (e.g., "helpdesk", "Bread", "Shops")
  - `name`: Location identifier used in survey (e.g., "future_amman", "red_crescent")
  - `label`: Human-readable location name (e.g., "Future Club Huassin", "Jordan Red Crescent")
  - `Gov`: Governorate
  - `GPS coordinate`: Coordinate string in format "lat, lon" (e.g., "32.0, 35.9")
  - `Comment`: Optional notes

- **POs** sheet: Contains post office locations (similar structure)

### Processing GPS Coordinates

To process GPS coordinates from the Excel file:

```r
# Run the GPS processing script
source("tools/process_gps_coordinates.r")
```

This script will:
1. Load GPS coordinates from `metadata/raw/GPS coordinates.xlsx`
2. Parse GPS coordinate strings (format: "lat, lon") into separate `lat` and `lon` columns
3. Remove rows with invalid coordinates
4. Save processed .rds files to `metadata/processed/`:
   - `gps_HelpDesk.rds` - Helpdesk locations (17 locations)
   - `gps_POs.rds` - Post office locations (92 locations)
   - `all_gps_coordinates.rds` - Combined file with all sheets

**Example output**:
```
GPS COORDINATES PROCESSING PIPELINE
================================================================================

Loading GPS coordinates from: .../metadata/raw/GPS coordinates.xlsx

=== Loading Sheets ===
Found sheets: POs, HelpDesk

Processing sheet: HelpDesk
  Rows: 30
  Columns: list_name, name, label, Gov, GPS coordinate, Comment
  Valid coordinates: 30
  ✓ Processed successfully

=== Saving Processed GPS Coordinates ===
Saved: .../metadata/processed/gps_HelpDesk.rds
  Rows: 30
```

### Loading GPS Coordinates

```r
# Load GPS coordinates for a specific sheet
gps_data <- load_gps_coordinates("HelpDesk", "../metadata/processed")

# Check what's loaded
print(head(gps_data[, c("name", "label", "lat", "lon")]))

# Load all GPS coordinates
all_gps <- load_all_gps_coordinates("../metadata/processed")
```

### Using GPS Coordinates in Validation

GPS coordinates are used to validate that entered GPS coordinates match the selected location. See the complete example in [GPS Validation Guide](https://wfp-vam.github.io/JOR-data-quality-automation-docs/guides/ADDING_NEW_EXERCISES.html#gps_location_match_rule).

**Quick reference**:

```r
# In your exercise's run_validations function
# For HelpDesk locations:
gps_data <- tryCatch({
  load_gps_coordinates("HelpDesk", "../metadata/processed")
}, error = function(e) {
  NULL
})

# For Post Office locations:
# gps_data <- tryCatch({
#   load_gps_coordinates("POs", "../metadata/processed")
# }, error = function(e) {
#   NULL
# })

if (!is.null(gps_data)) {
  result <- result |>
    gps_location_match_rule(
      location_column = "_1_6_Helpdesk_name",  # Your location dropdown column
      gps_data = gps_data,
      list_name_filter = "helpdesk",           # Must match list_name in GPS file
      max_distance_meters = 100                # Distance threshold
    )(id = 7, data_name = exercise_name)
}

# The validation automatically adds a 'gps_distance_to_target_m' column
# showing the distance in meters for each record
```

**Important Notes**:

1. **`list_name_filter` must match**: The `list_name_filter` parameter must match the `list_name` values in your GPS coordinates file. For example, if your GPS file has `list_name = "helpdesk"`, use `list_name_filter = "helpdesk"`.

2. **Location matching**: The validation uses a robust multi-strategy matching approach (in order of preference):
   - **Exact match by `name`** (most common - numeric codes like "1", "2", "3" for post offices)
   - **Exact match by `label`** (Arabic or English names)
   - **Case-insensitive exact match** by `name` or `label`
   - **Partial string matching** (fallback) - checks if location name appears in GPS data or vice versa
   - All matching includes whitespace trimming for reliability
   - Matching is optimized to handle common data variations and formatting differences

3. **GPS column formats**: The validation automatically handles:
   - `_geolocation` column: List/vector format `c(lat, lon)` or string `"lat lon"`
   - `gps` column: String format `"lat lon alt accuracy"`
   - Separate columns: `latitude`/`longitude` or `lat`/`lon`

4. **Keep GPS columns**: Don't remove `_geolocation` in `prepare_data` - it's needed for validation!

5. **Distance column**: The validation automatically adds a `gps_distance_to_target_m` column showing the distance in meters between entered GPS coordinates and the expected location. This column:
   - Appears in the affirm validation report
   - Is included in CSV exports (add `"gps_distance_to_target_m"` to `affirm.id_cols`)
   - Shows `NA` for records without GPS data or when location is not found
   - Helps identify how far off GPS coordinates are from expected locations

### Updating GPS Coordinates

When GPS coordinates need to be updated or corrected:

1. **Edit the Excel file**: Update `metadata/raw/GPS coordinates.xlsx` with new or corrected coordinates
2. **Reprocess**: Run `source("tools/process_gps_coordinates.r")` to regenerate processed files
3. **Restart app**: Restart the Shiny app to load new GPS coordinates

**Adding new locations**:
- Add rows to the appropriate sheet (HelpDesk, POs, etc.)
- Ensure `list_name` matches what you'll use in `list_name_filter`
- Format GPS coordinates as "lat, lon" (e.g., "32.0, 35.9")
- Run the processing script to update .rds files

### Example: Complete GPS Validation Setup

See these complete working examples:

- **BCM Helpdesk Validation** (`app/exercises/bcm_helpdesk_validation.r`): Uses HelpDesk GPS coordinates with `list_name_filter = "helpdesk"`
- **BCM Post Office Validation** (`app/exercises/bcm_post_office_validation.r`): Uses Post Office GPS coordinates with `list_name_filter = "post_name"`
- **OSM Post Office Validation** (`app/exercises/osm_post_office.r`): Uses Post Office GPS coordinates with `list_name_filter = "post_name"`
- **FSOM 2025 Q3** (`app/exercises/fsom.r`): Non-GPS example that derives `visit_date` from `starttime` and uses standard metadata

The GPS examples demonstrate:
- Processing GPS coordinates
- Keeping `_geolocation` column in `prepare_data`
- Loading GPS data in `run_validations`
- Applying GPS validation with proper error handling
- Including `gps_distance_to_target_m` in CSV exports

## References

- Full integration details: `docs/development/METADATA_INTEGRATION.md`
- Metadata helpers API: `app/R/metadata_helpers.r`
- Example usage: `tools/examples/explore_metadata.r`
- Test script: `tools/test_metadata_integration.r`

