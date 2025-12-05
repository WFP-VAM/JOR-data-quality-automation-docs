# Adding New Exercises to the Data Quality App

This guide provides step-by-step instructions for adding a new exercise (data source) to the Data Quality Validation App. Follow these steps carefully to ensure your exercise integrates seamlessly with the existing system.

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Finding Your Survey ID (Interactive Discovery)](#finding-your-survey-id-interactive-discovery)
4. [Step-by-Step Process](#step-by-step-process)
5. [Exercise Structure Reference](#exercise-structure-reference)
6. [Validation Rules Reference](#validation-rules-reference)
7. [CSV Export Configuration](#csv-export-configuration)
8. [Visualization Reference](#visualization-reference)
9. [Testing Your Exercise](#testing-your-exercise)
10. [Troubleshooting](#troubleshooting)

---

## Overview

The app uses a **registry-based system** where each exercise is defined as a standalone R file that follows a specific structure. The exercise registry (`app/R/exercise_registry.r`) automatically discovers and loads all exercise files from the `app/exercises/` directory.

### Key Components

- **Exercise Definition File**: Defines metadata, data fetching, preparation, validations, and visualizations
- **Validation Rules**: Reusable validation functions from `app/R/validation_rules.r` using **affirm** package
- **Visualization Helpers**: Reusable UI components from `app/R/visualization_helpers.r`
- **Data Source Helpers**: Optional utilities from `app/R/data_source_helpers.r`

### Validation Framework

This app uses the [**affirm**](https://pcctc.github.io/affirm/) package for data validation and reporting. Affirm provides:

- ✅ User-friendly error messages
- ✅ Comprehensive violation tracking with full data (not just samples)
- ✅ Downloadable CSV exports of all violated records
- ✅ Professional gt-based reports
- ✅ Purpose-built for data validation workflows

---

## Prerequisites

Before adding a new exercise, ensure you have:

1. **Survey/Data Source Information**:
   - Survey ID or data source identifier
   - Access credentials configured in `data_bridges_api_config.yaml`
   - Understanding of the data structure (columns, data types)

2. **Data Quality Requirements**:
   - List of validation rules needed
   - Acceptable thresholds (e.g., max duration, missing data limits)
   - Business rules specific to this exercise

3. **Visualization Needs**:
   - Key metrics to display
   - Any custom visualizations required

---

## Finding Your Survey ID (Interactive Discovery)

Before creating an exercise, you need to know the **Survey ID** from the DataBridges API. This section walks you through discovering available surveys and their IDs interactively.

### Method 1: Interactive Discovery Script (Recommended)

We've created a helper script that makes it easy to discover surveys and explore their data structure.

#### Step 1: Run the Discovery Script

```r
# In R console (from project root)
source("tools/discover_surveys.r")
```

This script will:

1. ✓ Connect to the DataBridges API
2. ✓ List available surveys (if API supports it)
3. ✓ Provide search functions
4. ✓ Offer inspection tools

#### Step 2: Search for Your Survey

The script provides a `search_surveys()` function to find surveys by name:

```r
# Search for surveys containing "welcome"
search_surveys("welcome")

# Search for surveys containing "meals"
search_surveys("meals")

# Search for surveys containing "BCM"
search_surveys("BCM")
```

**Example output**:
```
Found 2 survey(s) matching 'welcome':

  surveyId            surveyTitle
      5400  BCM - Welcome Meals
      5401  OSM - Welcome Meals Distribution
```

#### Step 3: Inspect the Survey Data Structure

Once you have a survey ID, inspect its data to understand the structure:

```r
# Inspect survey 5400
inspect_survey(5400)
```

**This will show you**:

- Number of rows and columns
- All column names
- Potential filter columns (e.g., enumerator names, locations)
- Sample data from the first few rows
- Data types for each column

#### Step 4: Identify Key Information

From your exploration, note:

1. **Survey ID**: `5400` (use this in your exercise)
2. **Filter Column**: `Enumerator_Name` (good for filtering by enumerator)
3. **Key Columns**: Columns used in validations and visualizations
4. **Date Column**: `date` (needs conversion to Date type)
5. **ID Column**: `Beneficiary_ID` (for uniqueness validation)
6. **CSV Export Columns**: Important columns users need to see in violation exports

---

## Step-by-Step Process

### Step 0: Gather Survey Information

**Before writing any code**, complete the interactive discovery process above to gather:

- ✓ Survey ID (e.g., `5400`)
- ✓ Survey name/description
- ✓ Column names and data types
- ✓ Potential filter columns
- ✓ Key metrics columns for visualizations
- ✓ ID columns for uniqueness validation
- ✓ Timing columns (start, end, duration, submission_time)
- ✓ Columns needed in CSV exports

### Step 1: Create the Exercise File

Create a new R file in `app/exercises/` with a descriptive name (use lowercase, underscores for spaces).

**Naming convention**: `{organization}_{survey_type}.r`

**Example**: `bcm_welcome_meals.r`

```bash
touch app/exercises/bcm_welcome_meals.r
```

### Step 2: Define Basic Metadata

Start your exercise file with metadata that identifies and describes the exercise:

```r
# BCM Welcome Meals Exercise Definition
# BCM - Welcome Meals Survey - Survey ID XXXX

exercise <- list(
  # Basic metadata
  id = "XXXX",  # Survey ID or unique identifier
  name = "BCM - Welcome Meals",  # Display name
  type = "survey",  # Type: "survey", "monitoring", "assessment", etc.
  description = "Jordan BCM welcome meals data quality validation",
  
  # ... rest of configuration
)
```

**Important**: The `exercise` object name must be exactly `exercise` - the registry looks for this variable name.

### Step 3: Configure Filtering

Define how users can filter the data in the UI:

```r
  # Filter configuration
  filter_column = "Enumerator_Name",  # Column name to filter by
  filter_label = "Enumerator",  # Human-readable label for the filter
```

**Notes**:

- `filter_column` must match an actual column name in your dataset
- `filter_label` is what users see in the dropdown
- If you don't want filtering, set both to `NULL`

### Step 4: Define Data Fetching Function

Implement the `fetch_data` function that retrieves data from the source:

```r
  # Data fetching function - specific to this exercise
  fetch_data = function(client) {
    with_error_handling(
      fetch_fn = function() {
        databridges_survey_fetch(
          client = client,
          survey_id = XXXX,  # Your survey ID as integer
          access_type = "full"
        )
      },
      source_name = "BCM - Welcome Meals (Survey XXXX)"
    )
  },
```

**Tips**:

- Use `with_error_handling()` wrapper for consistent error logging
- For DataBridges surveys, use `databridges_survey_fetch()` helper
- For custom data sources, implement your own fetch logic inside `fetch_fn`
- Always convert `survey_id` to integer with `L` suffix (e.g., `5387L`)

### Step 5: Define Data Preparation Function

Implement the `prepare_data` function to clean and transform raw data:

```r
  # Data preparation steps
  prepare_data = function(raw_data) {
    # Use standard preparation helper
    data <- prepare_survey_data(raw_data)
    
    # Add any exercise-specific transformations here
    # Example: Standardize text fields
    if ("meal_type" %in% names(data)) {
      data <- data |> mutate(meal_type = trimws(tolower(meal_type)))
    }
    
    data
  },
```

**Common preparation steps**:

1. Use `prepare_survey_data()` helper for standard cleaning (removes problematic columns, converts dates)
2. Add exercise-specific transformations as needed
3. Clean or standardize text fields
4. Calculate derived columns
5. Handle missing values

#### Handling Missing Date Column

Some surveys don't include a `date` column but validation rules require it. If your survey only has `_submission_time`, derive the date column:

```r
prepare_data = function(raw_data) {
  data <- prepare_survey_data(raw_data)
  
  # Derive date column from _submission_time if date column doesn't exist
  if (!"date" %in% names(data) && "_submission_time" %in% names(data)) {
    data <- data |>
      dplyr::mutate(
        date = as.Date(substr(.data[["_submission_time"]], 1, 10))
      )
  }
  
  data
}
```

**When to use this**:

- Your survey doesn't have a `date` column
- You're using `same_day_submission_rule()` or `late_submission_rule()`
- Your survey has `_submission_time` column

**Note**: The `substr(..., 1, 10)` extracts the date portion (YYYY-MM-DD) from the ISO timestamp.

### Step 6: Define Validation Function with affirm

**Required**: Every exercise must define a `run_validations` function. The app calls this function to run data quality checks.

Specify validations using the **affirm** package:

```r
  # Validation function using affirm
  run_validations = function(data) {
    # Initialize affirm session and set ID columns for CSV export
    affirm::affirm_init(replace = TRUE)
    options('affirm.id_cols' = c("Case_ID", "date", "Enumerator_Name", 
                                   "_submission_time", "start", "end", "_duration"))
    
    exercise_name <- "BCM - Welcome Meals"
    
    # Run all validations in sequence and remove helper columns at the end
    result <- data |>
      duration_check_rule()(id = 1, data_name = exercise_name) |>
      same_day_submission_rule()(id = 2, data_name = exercise_name) |>
      late_submission_rule()(id = 3, data_name = exercise_name) |>
      case_id_uniqueness_rule(case_id_column = "Case_ID")(id = 4, data_name = exercise_name) |>
      phone_uniqueness_rule()(id = 5, data_name = exercise_name) |>
      missing_data_rule()(id = 6, data_name = exercise_name)
    
    # Remove all helper columns before returning
    result |> dplyr::select(-dplyr::starts_with("."))
  }
```

**Key points**:

- **`affirm_init(replace = TRUE)`**: Initializes a new affirm validation session
- **`options('affirm.id_cols' = c(...))`**: Specifies which columns to include in CSV exports (see [CSV Export Configuration](#csv-export-configuration))
- **`data_name`**: Exercise name for organizing violations in reports
- **Validation pipeline**: Chain validation rules using `|>`
- **Helper column cleanup**: Remove temporary columns (starting with `.`) at the end

**Available validation rules** (see [Validation Rules Reference](#validation-rules-reference)):

- `duration_check_rule()` - Survey duration bounds
- `same_day_submission_rule()` - Same-day submission verification
- `late_submission_rule()` - Submission delay check
- `missing_data_rule()` - Excessive missing data detection
- `case_id_uniqueness_rule()` - Case ID uniqueness per day
- `phone_uniqueness_rule()` - Phone number uniqueness per day
- `location_visit_frequency_rule()` - Location visit frequency limits
- `proportion_check_rule()` - Proportion-based checks (e.g., overcrowding, complaint rates)

### Step 7: Define Visualization Function

Implement the `visualizations` function to display monitoring insights:

**Option A: Use an existing dashboard function**

```r
  # Visualization function - displays monitoring insights
  visualizations = function(data) {
    bcm_monitoring_dashboard(data)  # Use existing BCM dashboard
  },
```

**Option B: Create a custom dashboard**

```r
  # Visualization function - displays custom monitoring insights
  visualizations = function(data) {
    bcm_welcome_meals_dashboard(data)  # Custom function (define in visualization_helpers.r)
  },
```

**Option C: No visualizations**

```r
  visualizations = NULL,  # No custom visualizations
```

### Step 7.5: Integrate Metadata (Variable Labels & Choice Lists)

**NEW**: The app now supports automatic loading of XLSForm metadata to display human-readable variable labels and choice labels in the UI.

#### What Metadata Provides

- **Variable Labels**: Display "1.5 Gender of Respondent?" instead of `Q2_1_gender`
- **Choice Labels**: Display "Male" and "Female" instead of codes like `0` and `1`
- **Column Details**: Shows descriptive labels for all columns
- **Better Visualizations**: Categorical breakdowns use actual response text

#### Step 7.5.1: Process Metadata

1. **Locate your XLSForm file** in the SharePoint:
   - Download from: [JOCOVAME > Data Quality Checks > Questionnaire XLS](https://wfp.sharepoint.com/sites/JOCOVAME/Shared%20Documents/Forms/AllItems.aspx?id=/sites/JOCOVAME/Shared%20Documents/General/Monitoring/Data%20Quality%20Checks%20-%20Company%202025/Questionnaire%20XLS)
   - Place in: `metadata/raw/YYYYMMDD/[category]/`
   - Example: `metadata/raw/20251103/General Food Assistance xls/Welcome Meals.xlsx`

2. **Update metadata mapping** in `app/R/metadata_helpers.r`:

```r
# In the map_metadata_to_survey_ids() function:
filename_mappings <- list(
  "Welcome Meals.xlsx" = "5404",
  "Your_Survey_File.xlsx" = "XXXX",  # Add your mapping here
  # ... existing mappings
)
```

3. **Process the metadata** (creates compressed .rds file):

```r
# Create a processing script or use the main one
source("app/R/metadata_helpers.r")

# Load and process your XLSForm
xlsform <- load_xlsform("metadata/raw/.../Your_Survey_File.xlsx")
metadata <- process_xlsform(xlsform)

# Save processed metadata
saveRDS(metadata, "metadata/processed/XXXX_metadata.rds")

# Update combined metadata file
all_metadata <- load_all_metadata("metadata/processed")
all_metadata[["XXXX"]] <- metadata
saveRDS(all_metadata, "metadata/processed/all_metadata.rds")
```

4. **Verify metadata structure**:

```r
# Load and inspect
metadata <- load_survey_metadata("XXXX", "metadata/processed")

# Check variable labels
head(metadata$variables)

# Check choice lists
names(metadata$choices)
metadata$choices$gender  # Example choice list
```

#### Step 7.5.2: Update Exercise to Load Metadata

Modify your `visualizations` function to load metadata:

```r
  visualizations = function(data) {
    # Load metadata for this exercise
    metadata <- tryCatch({
      load_survey_metadata("XXXX", "../metadata/processed")
    }, error = function(e) {
      NULL  # Graceful fallback if metadata not available
    })
    
    your_dashboard(data, metadata = metadata)
  },
```

#### Step 7.5.3: Update Dashboard to Use Metadata

Modify your dashboard function to accept and use metadata:

```r
your_dashboard <- function(data, metadata = NULL) {
  # Helper function to get labels from metadata
  get_label <- function(column_name, fallback) {
    if (!is.null(metadata)) {
      label <- get_variable_label(metadata, column_name)
      if (label != column_name) {
        return(label)
      }
    }
    return(fallback)
  }
  
  # Use metadata labels in visualizations
  breakdown_card(
    data, 
    "Q2_1_gender",  # Column name
    get_label("Q2_1_gender", "Gender Distribution"),  # Label from metadata
    metadata = metadata,  # Pass metadata
    choice_list_name = "gender"  # Choice list name from XLSForm
  )
}
```

#### Step 7.5.4: Find Choice List Names

Choice list names come from the XLSForm `choices` sheet's `list_name` column:

**XLSForm survey sheet:**
```
type                | name       | label
select_one gender   | Q2_1_gender| 1.5 Gender of Respondent?
select_one yesno    | consent    | Do you consent?
```

**XLSForm choices sheet:**
```
list_name | name   | label
gender    | Male   | Male
gender    | Female | Female
yesno     | Yes    | Yes
yesno     | No     | No
```

Use `"gender"` and `"yesno"` as the `choice_list_name` parameter.

#### Benefits of Metadata Integration

- ✅ **Automatic updates**: Labels update when metadata changes, no code changes needed
- ✅ **Consistency**: Labels match exact survey definitions
- ✅ **Multilingual support**: Can include Arabic or other language labels
- ✅ **Better UX**: Users see meaningful labels instead of technical column names
- ✅ **Fast loading**: Compressed .rds files load instantly (88% smaller than Excel)

#### Metadata Integration is Optional

If metadata isn't available or not needed, the exercise works fine without it:

- Column Details will show column names without labels (shows "—")
- Visualizations use fallback labels provided in code
- All functionality remains operational

For complete metadata integration examples, see:

- `app/exercises/bcm_helpdesk_validation.r`
- `app/exercises/bcm_welcome_meals.r`
- `app/exercises/osm_helpdesk.r`
- `app/exercises/osm_post_office.r`

For detailed metadata documentation, see `docs/guides/ADDING_METADATA.md`.

### Step 8: Complete Exercise Definition

Ensure your exercise list is complete and properly closed. Your exercise definition should end with:

```r
  # ... all your exercise components ...
)  # Close the exercise list
```

The closing parenthesis `)` completes the `exercise <- list(...)` definition that started at the beginning of your file.

---

## Exercise Structure Reference

### Complete Example

```r
# BCM Welcome Meals Exercise Definition
# BCM - Welcome Meals Survey - Survey ID 5404

exercise <- list(
  # Basic metadata
  id = "5404",
  name = "BCM - Welcome Meals",
  type = "survey",
  description = "Jordan BCM welcome meals data quality validation",
  
  # Filter configuration
  filter_column = "Q1_1_Enumerator_Name",
  filter_label = "Enumerator",
  
  # Data fetching function
  fetch_data = function(client) {
    with_error_handling(
      fetch_fn = function() {
        databridges_survey_fetch(
          client = client,
          survey_id = 5404L,
          access_type = "full"
        )
      },
      source_name = "BCM - Welcome Meals (Survey 5404)"
    )
  },
  
  # Data preparation
  prepare_data = function(raw_data) {
    data <- prepare_survey_data(raw_data)
    
    # Exercise-specific: Standardize meal type field
    if ("meal_type" %in% names(data)) {
      data <- data |> mutate(meal_type = trimws(tolower(meal_type)))
    }
    
    data
  },
  
  # Visualizations
  visualizations = function(data) {
    bcm_welcome_meals_dashboard(data)
  },
  
  # Validation function using affirm
  run_validations = function(data) {
    affirm::affirm_init(replace = TRUE)
    options('affirm.id_cols' = c("Beneficiary_ID", "date", "Q1_1_Enumerator_Name", 
                                   "_submission_time", "start", "end", "_duration",
                                   "has_complaint", "Interviewee_Phone_Number"))
    
    exercise_name <- "BCM - Welcome Meals"
    
    result <- data |>
      duration_check_rule(min_seconds = 300, max_seconds = 3600)(id = 1, data_name = exercise_name) |>
      same_day_submission_rule()(id = 2, data_name = exercise_name) |>
      late_submission_rule()(id = 3, data_name = exercise_name) |>
      case_id_uniqueness_rule(case_id_column = "Beneficiary_ID")(id = 4, data_name = exercise_name) |>
      phone_uniqueness_rule()(id = 5, data_name = exercise_name) |>
      proportion_check_rule(
        column_name = "has_complaint",
        positive_value = "Yes",
        max_proportion = 0.10,
        description_text = "Complaint rate is not more than 10%"
      )(id = 6, data_name = exercise_name) |>
      missing_data_rule(max_missing_proportion = 0.30)(id = 7, data_name = exercise_name)
    
    result |> dplyr::select(-dplyr::starts_with("."))
  }
)
```

---

## Validation Rules Reference

All validation rules in `app/R/validation_rules.r` return a function that:

1. Takes `data`, `id`, and `data_name` parameters
2. Applies `affirm_true()` validation
3. Returns the data (possibly with temporary helper columns)

### duration_check_rule()

Validates survey duration is within acceptable bounds.

```r
duration_check_rule(
  duration_column = "_duration",  # Column with duration in seconds
  min_seconds = 300,              # Minimum duration (5 minutes)
  max_seconds = 86400             # Maximum duration (24 hours)
)
```

### same_day_submission_rule()

Validates survey was submitted on the same day as data collection.

```r
same_day_submission_rule(
  date_column = "date",                    # Collection date column
  submission_column = "_submission_time"   # Submission timestamp column
)
```

### late_submission_rule()

Validates survey was submitted within acceptable time after collection.

```r
late_submission_rule(
  date_column = "date",                    # Collection date column
  submission_column = "_submission_time",  # Submission timestamp column
  max_hours = 24                          # Maximum delay in hours
)
```

### missing_data_rule()

Validates rows don't have excessive missing values.

```r
missing_data_rule(
  max_missing_proportion = 0.5  # Max 50% missing values per row
)
```

### case_id_uniqueness_rule()

Validates Case ID is unique within each collection date.

```r
case_id_uniqueness_rule(
  case_id_column = "Case_ID",  # Column with case identifier
  date_column = "date"         # Date column for grouping
)
```

### phone_uniqueness_rule()

Validates phone number is unique within each collection date.

```r
phone_uniqueness_rule(
  phone_column = "Interviewee_Phone_Number",  # Phone column
  date_column = "date"                        # Date column for grouping
)
```

### location_visit_frequency_rule()

Validates locations are not visited more than allowed per month.

```r
location_visit_frequency_rule(
  location_column = "location",        # Location name column
  date_column = "date",                # Date column
  max_visits_host = 1,                 # Max visits for host communities
  max_visits_camp = 2,                 # Max visits for camps
  camp_keywords = c("camp", "مخيم")    # Keywords to identify camps
)
```

### proportion_check_rule()

Validates proportion of positive responses doesn't exceed threshold.

```r
proportion_check_rule(
  column_name = "overcrowding",      # Column to check
  positive_value = "Yes",            # Value counting as "positive"
  max_proportion = 0.10,             # Maximum proportion (10%)
  description_text = "Custom text"   # Optional custom description
)
```

---

## Creating Custom Validation Rules

If existing validation rules don't meet your needs, you can create custom rules following the established pattern.

### Validation Rule Structure

All validation rules follow this pattern:

```r
#' Custom validation rule
#' @param param1 Description of parameter
#' @param param2 Description of parameter
#' @return Function that applies validation to data
custom_validation_rule <- function(param1 = default1, param2 = default2) {
  function(data, id, data_name = "Survey Data") {
    # 1. Check if required columns exist
    if (!"required_column" %in% names(data)) {
      return(data)
    }
    
    # 2. Create helper columns if needed (start with .)
    result_data <- data |>
      dplyr::mutate(
        .helper_check = some_condition(.data[["column_name"]])
      )
    
    # 3. Apply validation using affirm_true()
    result_data <- result_data |>
      affirm::affirm_true(
        label = "Human-readable description of what this checks",
        condition = .helper_check,  # Boolean condition
        id = id,
        data_frames = data_name
      )
    
    # 4. Clean up helper columns (if not needed in output)
    result_data |> dplyr::select(-.helper_check)
  }
}
```

### Key Patterns

#### 1. Simple Field Check

```r
simple_field_check_rule <- function(column_name, invalid_values = c("N/A", "No")) {
  function(data, id, data_name = "Survey Data") {
    if (column_name %in% names(data)) {
      affirm::affirm_true(
        data,
        label = sprintf("%s does not contain invalid values", column_name),
        condition = is.na(.data[[column_name]]) |
                   .data[[column_name]] == "" |
                   !(.data[[column_name]] %in% invalid_values),
        id = id,
        data_frames = data_name
      )
    } else {
      data
    }
  }
}
```

#### 2. Multiple Field Check

```r
multiple_field_check_rule <- function(columns, invalid_values = c("N/A", "No")) {
  function(data, id, data_name = "Survey Data") {
    result_data <- data
    
    # Check each column sequentially
    for (i in seq_along(columns)) {
      col <- columns[i]
      if (col %in% names(result_data)) {
        result_data <- result_data |>
          affirm::affirm_true(
            label = sprintf("Field %s does not contain invalid values", col),
            condition = is.na(.data[[col]]) |
                       .data[[col]] == "" |
                       !(.data[[col]] %in% invalid_values),
            id = as.integer(id + i - 1),  # Increment ID for each check
            data_frames = data_name
          )
      }
    }
    
    result_data
  }
}
```

#### 3. Conditional Logic

```r
conditional_field_rule <- function(trigger_column, trigger_value, required_column) {
  function(data, id, data_name = "Survey Data") {
    if (!all(c(trigger_column, required_column) %in% names(data))) {
      return(data)
    }
    
    result_data <- data |>
      affirm::affirm_true(
        label = sprintf("%s must be answered when %s = %s", 
                       required_column, trigger_column, trigger_value),
        condition = is.na(.data[[trigger_column]]) |
                   .data[[trigger_column]] != trigger_value |
                   (!is.na(.data[[required_column]]) & 
                    .data[[required_column]] != ""),
        id = id,
        data_frames = data_name
      )
    
    result_data
  }
}
```

#### 4. Helper Columns for Complex Logic

```r
complex_validation_rule <- function() {
  function(data, id, data_name = "Survey Data") {
    result_data <- data |>
      dplyr::mutate(
        # Create helper columns (must start with .)
        .computed_value = some_calculation(.data[["column1"]], .data[["column2"]]),
        .check_passed = .computed_value > threshold
      ) |>
      affirm::affirm_true(
        label = "Complex validation description",
        condition = .check_passed,
        id = id,
        data_frames = data_name
      ) |>
      # Remove helper columns before returning
      dplyr::select(-.computed_value, -.check_passed)
    
    result_data
  }
}
```

### Adding Helper Columns to globalVariables

If your custom rule creates **new helper columns** (starting with `.`), add them to `globalVariables()`:

```r
# In app/R/validation_rules.r
utils::globalVariables(c(
  # ... existing variables ...
  ".my_custom_helper_column"  # Add your new helper column
))
```

**Note**: You only need to add helper columns (starting with `.`), not regular data columns. Regular columns accessed via `.data[["column_name"]]` don't need to be declared.

### Using Custom Rules in Exercises

Once created, use custom rules just like standard rules:

```r
run_validations = function(data) {
  affirm::affirm_init(replace = TRUE)
  options('affirm.id_cols' = c("ID02", "date", ...))
  
  exercise_name <- "My Exercise"
  
  result <- data |>
    duration_check_rule()(id = 1, data_name = exercise_name) |>
    custom_validation_rule(param1 = value1)(id = 2, data_name = exercise_name) |>
    another_custom_rule()(id = 3, data_name = exercise_name)
  
  result |> dplyr::select(-dplyr::starts_with("."))
}
```

### Best Practices

1. **Always check column existence** before accessing columns
2. **Use descriptive labels** that explain what the validation checks
3. **Increment IDs** when checking multiple fields (id, id+1, id+2, etc.)
4. **Clean up helper columns** before returning (unless needed for CSV export)
5. **Handle missing columns gracefully** (return data unchanged if columns don't exist)
6. **Use `.data[[]]` syntax** for programmatic column access
7. **Document parameters** with roxygen comments

---

## Visualization Reference

This section provides complete guidance on creating custom dashboard visualizations.

### Dashboard Function Structure

All dashboard functions follow this pattern:

```r
#' Create monitoring dashboard for [Exercise Name]
#' @param data The survey data
#' @param metadata Optional metadata for applying labels
#' @return HTML with monitoring visualizations
your_exercise_dashboard <- function(data, metadata = NULL) {
  if (is.null(data) || nrow(data) == 0) {
    return(NULL)
  }
  
  # Helper function to get label from metadata, with fallback
  get_label <- function(column_name, fallback) {
    if (!is.null(metadata)) {
      label <- get_variable_label(metadata, column_name)
      if (label != column_name) {
        return(label)
      }
    }
    return(fallback)
  }

  tagList(
    tags$hr(),
    tags$h3("Findings Overview", style = "margin-top: 20px; margin-bottom: 15px; color: #333;"),
    
    # Your dashboard sections here
    # ...
  )
}
```

### Available UI Components

#### metric_card()

Displays a percentage metric with color coding (green ≥80%, yellow ≥50%, red <50%).

```r
metric_card(
  data,                    # Survey data frame
  column,                  # Column name to analyze
  label,                   # Display label
  positive_value = "Yes"  # Value that counts as "positive" (default "Yes")
)
```

**Example**:
```r
metric_card(data, "has_complaint", "Has Complaint", positive_value = "No")
```

#### breakdown_card()

Displays a categorical breakdown with bar chart.

```r
breakdown_card(
  data,                    # Survey data frame
  column,                  # Column name to analyze
  label,                   # Display label
  metadata = NULL,         # Optional metadata for labels
  choice_list_name = NULL  # Choice list name from XLSForm (for label mapping)
)
```

**Example**:
```r
breakdown_card(data, "Q2_1_gender", 
              get_label("Q2_1_gender", "Gender Distribution"),
              metadata = metadata,
              choice_list_name = "gender")
```

#### time_metric_card()

Displays average time in minutes with context.

```r
time_metric_card(
  data,     # Survey data frame
  column,   # Time column name (in seconds)
  label     # Display label
)
```

**Example**:
```r
time_metric_card(data, "Q4_2_waitingtime", "Average Waiting Time")
```

#### satisfaction_metric_card()

Displays satisfaction percentage (for Yes/No questions).

```r
satisfaction_metric_card(
  data,     # Survey data frame
  column,   # Column name
  label     # Display label
)
```

**Example**:
```r
satisfaction_metric_card(data, "Q4_1_were_trated_respectfully", 
                        "Treated Respectfully")
```

### Shiny UI Layout Components

#### tagList()

Container for multiple UI elements.

```r
tagList(
  tags$hr(),
  tags$h3("Section Title"),
  div(...),
  fluidRow(...)
)
```

#### fluidRow() and column()

Create responsive grid layouts.

```r
fluidRow(
  column(4, metric_card(...)),  # 4/12 width
  column(4, metric_card(...)),  # 4/12 width
  column(4, metric_card(...))   # 4/12 width
)
```

**Column widths**: Use 1-12 (Bootstrap grid system). Common patterns:
- `column(4, ...)` - 3 columns per row
- `column(6, ...)` - 2 columns per row
- `column(12, ...)` - Full width

#### Conditional Rendering

Only show sections if columns exist:

```r
if ("column_name" %in% names(data)) {
  div(
    style = "margin-bottom: 20px;",
    h4("Section Title"),
    metric_card(data, "column_name", "Label")
  )
} else {
  NULL
}
```

### Common Dashboard Patterns

#### 1. Survey Duration Overview

```r
if ("_duration" %in% names(data)) {
  duration_filtered <- data$`_duration`[data$`_duration` != ""]
  valid_count <- sum(!is.na(duration_filtered) & duration_filtered != "")
  
  if (valid_count > 0) {
    duration_numeric <- as.numeric(duration_filtered)
    avg_seconds <- mean(duration_numeric, na.rm = TRUE)
    
    if (!is.na(avg_seconds) && is.finite(avg_seconds) && avg_seconds > 0) {
      div(
        style = "margin-bottom: 20px;",
        div(
          style = "background: white; padding: 15px; border-radius: 4px; box-shadow: 0 1px 3px rgba(0,0,0,0.1);",
          div(style = "display: flex; justify-content: space-between; align-items: center;",
              div(
                style = "flex: 1;",
                div(style = "font-size: 14px; font-weight: 600; color: #333;", 
                    "Average Survey Duration"),
                div(style = "font-size: 12px; color: #666; margin-top: 5px;",
                    sprintf("Based on %d surveys", valid_count))
              ),
              div(
                style = "font-size: 28px; font-weight: bold; color: #007bff;",
                sprintf("%.1f min", avg_seconds / 60)
              )
          )
        )
      )
    }
  }
}
```

#### 2. Metric Cards Section

```r
div(
  style = "margin-bottom: 20px;",
  h4("Section Title", style = "color: #555; font-size: 16px; margin-bottom: 10px;"),
  p(style = "font-size: 12px; color: #666; margin-bottom: 10px;",
    "Description of what this section shows."),
  fluidRow(
    column(4, metric_card(data, "column1", get_label("column1", "Label 1"))),
    column(4, metric_card(data, "column2", get_label("column2", "Label 2"))),
    column(4, metric_card(data, "column3", get_label("column3", "Label 3")))
  )
)
```

#### 3. Breakdown Section

```r
div(
  style = "margin-bottom: 20px;",
  h4("Category Breakdown", style = "color: #555; font-size: 16px; margin-bottom: 10px;"),
  breakdown_card(data, "category_column", 
                get_label("category_column", "Category Distribution"),
                metadata = metadata,
                choice_list_name = "choice_list_name")
)
```

#### 4. Date Range Display

```r
if ("date" %in% names(data)) {
  div(
    style = "margin-bottom: 20px;",
    h4("Data Collection Timeline", style = "color: #555; font-size: 16px; margin-bottom: 10px;"),
    div(
      style = "background: white; padding: 15px; border-radius: 4px; box-shadow: 0 1px 3px rgba(0,0,0,0.1);",
      div(style = "font-size: 14px; color: #666; margin-bottom: 10px;", "Collection Period"),
      div(style = "font-size: 13px; color: #333;",
          sprintf("From %s to %s (%d days)",
                 min(data$date, na.rm = TRUE),
                 max(data$date, na.rm = TRUE),
                 as.numeric(difftime(max(data$date, na.rm = TRUE), 
                                    min(data$date, na.rm = TRUE), 
                                    units = "days"))))
    )
  )
}
```

### Complete Example

```r
sbcc_endline_women_dashboard <- function(data, metadata = NULL) {
  if (is.null(data) || nrow(data) == 0) {
    return(NULL)
  }
  
  get_label <- function(column_name, fallback) {
    if (!is.null(metadata)) {
      label <- get_variable_label(metadata, column_name)
      if (label != column_name) return(label)
    }
    return(fallback)
  }

  tagList(
    tags$hr(),
    tags$h3("Findings Overview", style = "margin-top: 20px; margin-bottom: 15px; color: #333;"),

    # Survey duration
    if ("_duration" %in% names(data)) {
      # ... duration code ...
    },

    # Metrics section
    div(
      style = "margin-bottom: 20px;",
      h4("Response Metrics", style = "color: #555; font-size: 16px; margin-bottom: 10px;"),
      fluidRow(
        if ("Q307" %in% names(data)) {
          column(4, metric_card(data, "Q307", get_label("Q307", "Response Q307")))
        },
        if ("Q309" %in% names(data)) {
          column(4, metric_card(data, "Q309", get_label("Q309", "Response Q309")))
        }
      )
    ),

    # Breakdown section
    if ("name" %in% names(data)) {
      div(
        style = "margin-bottom: 20px;",
        h4("Enumerator Activity", style = "color: #555; font-size: 16px; margin-bottom: 10px;"),
        breakdown_card(data, "name", get_label("name", "Surveys by Enumerator"),
                      metadata = metadata)
      )
    }
  )
}
```

### Best Practices

1. **Always check for empty data**: Return `NULL` if data is empty
2. **Use conditional rendering**: Only show sections if columns exist
3. **Use metadata labels**: Always use `get_label()` helper with fallbacks
4. **Consistent styling**: Follow existing dashboard patterns for visual consistency
5. **Responsive layout**: Use `fluidRow()` and `column()` for responsive design
6. **Error handling**: Handle missing columns gracefully (return `NULL` or skip section)
7. **Documentation**: Add roxygen comments describing what the dashboard shows

### Adding Dashboard to Exercise

Once created, add the dashboard function to `app/R/visualization_helpers.r` and reference it in your exercise:

```r
visualizations = function(data) {
  metadata <- tryCatch({
    load_survey_metadata("XXXX", "../metadata/processed")
  }, error = function(e) {
    NULL
  })
  
  your_exercise_dashboard(data, metadata = metadata)
}
```

---

## CSV Export Configuration

The `affirm.id_cols` option controls which columns appear in CSV exports of violated records. This is **critical** for users to understand and verify violations.

### What to Include

Include columns that help users:

1. **Identify the record**: ID columns, date
2. **Understand the context**: Filter column (enumerator/location), timing columns
3. **Verify the validation**: Columns used in the validation logic

### Examples by Exercise Type

#### BCM Helpdesk:
```r
options('affirm.id_cols' = c(
  "Case_ID",                    # Record identifier
  "date",                       # Collection date
  "_1_6_Helpdesk_name",        # Filter column (helps see which helpdesk)
  "_submission_time",           # When submitted
  "start", "end", "_duration",  # Timing columns (for duration checks)
  "Interviewee_Phone_Number"    # For phone uniqueness checks
))
```

#### OSM Helpdesk:
```r
options('affirm.id_cols' = c(
  "location",                   # Record identifier
  "date",                       # Collection date
  "enum_name",                  # Filter column (which enumerator)
  "time",                       # Survey time
  "_submission_time",           # When submitted
  "_duration",                  # Duration (no start/end for OSM)
  "overcrowding"                # For proportion checks
))
```

### Best Practice

**Include ALL columns used in validations**, plus:

- Primary ID columns
- Date/time columns
- Filter columns
- Any columns that provide context for understanding violations

---

## Testing Your Exercise

### Unit Testing Checklist

- [ ] **Exercise loads**: Appears in dropdown menu
- [ ] **Data fetches**: Successfully retrieves data without errors
- [ ] **Data count**: Correct number of records loaded
- [ ] **Column names**: All expected columns present
- [ ] **Validations execute**: All validation rules run without errors
- [ ] **Validation results**: Pass/fail counts are accurate and user-friendly
- [ ] **CSV exports work**: Download violated records and verify columns
- [ ] **Filtering works**: Filter dropdown appears and filters correctly
- [ ] **Visualizations render**: Dashboard displays without errors

### Manual Testing Steps

1. **Fresh start**: Restart R session and reload app
   ```r
   shiny::runApp("app")
   ```

2. **Select exercise**: Choose your exercise from dropdown

3. **Inspect data**: Check sidebar for record count and column list

4. **Review validations**: 
   - Check each validation result in the report
   - Verify descriptions are user-friendly
   - Ensure pass/fail status is correct

5. **Test CSV exports**: 
   - Click "CSV" download link for a failed validation
   - Verify CSV contains violated rows
   - Check that all configured ID columns are present
   - Verify timing columns (start, end, duration) are included

6. **Test filters**: Try different filter values (if applicable)

7. **Check visualizations**: Verify all charts and metrics

---

## Troubleshooting

### CSV Export is Empty (0 bytes)

**Cause**: Missing `data_frames` parameter or `affirm.id_cols` not set

**Solution**:

1. Ensure you pass `data_name` to each validation rule
2. Set `options('affirm.id_cols' = c(...))` at start of `run_validations`
3. Include ALL columns used in validations in `affirm.id_cols`

### Validation Error: "Column not found"

**Cause**: Column name in validation doesn't match actual data

**Solution**:

1. Use `inspect_survey()` to verify exact column names
2. Check for typos, case sensitivity, underscores vs spaces
3. Add conditional checks: `if (column %in% names(data))`

### Exercise Doesn't Appear in Dropdown

**Possible causes**:

1. File not in `app/exercises/` directory
2. File doesn't define `exercise` variable
3. Syntax error in exercise file
4. App not restarted after adding file

**Solutions**:

- Check file location: `ls app/exercises/`
- Verify `exercise <- list(...)` exists
- Test R syntax: `source("app/exercises/your_file.r")`
- Restart Shiny app

### Non-Standard Date Column Name

**Issue**: Some surveys use different column names for visit/collection dates (e.g., `Q_1_2_visit_date`, `visit_date`, `collection_date`) instead of the standard `date` column expected by validation rules.

**Solution**: Rename the date column in your `prepare_data` function:

```r
prepare_data = function(raw_data) {
  # Process with the actual date column name
  data <- prepare_survey_data(
    raw_data, 
    date_columns = c("Q_1_2_visit_date")  # Or whatever the actual column is
  )
  
  # Rename to 'date' for consistency with validation rules
  if ("Q_1_2_visit_date" %in% names(data)) {
    data <- data |> dplyr::rename(date = Q_1_2_visit_date)
  }
  
  data
}
```

**Why this matters**:

- Validation rules (`same_day_submission_rule`, `late_submission_rule`, `location_visit_frequency_rule`) expect a column named `date`
- CSV exports reference the `date` column
- Visualizations use `date` for time-based analysis

**Example exercises with non-standard date columns**:

- `osm_date_bards.r` - Uses `Q_1_2_visit_date`, renamed to `date` in prepare_data

**Example exercises with standard date column**:

- `osm_healthy_meals_hc.r` - Uses standard `date` column, no renaming needed

### Validation Fails: "date column not found"

**Issue**: Validation rules fail because the survey doesn't have a `date` column, but rules like `same_day_submission_rule()` and `late_submission_rule()` require it.

**Solution**: Derive the `date` column from `_submission_time` in your `prepare_data` function:

```r
prepare_data = function(raw_data) {
  data <- prepare_survey_data(raw_data)
  
  # Derive date column from _submission_time if date column doesn't exist
  if (!"date" %in% names(data) && "_submission_time" %in% names(data)) {
    data <- data |>
      dplyr::mutate(
        date = as.Date(substr(.data[["_submission_time"]], 1, 10))
      )
  }
  
  data
}
```

**Checklist**:

- Check if your survey has a `date` column: `"date" %in% names(data)`
- If not, derive it from `_submission_time` in `prepare_data` (see above)
- Ensure the derived date column is created before validations run
- The `substr(..., 1, 10)` extracts the date portion (YYYY-MM-DD) from the ISO timestamp

**Example exercises that derive date from submission time**:

- `gfa_sbcc_endline_women.r` - Derives `date` from `_submission_time`

### Choice List Names Don't Match Column Names

**Issue**: When using `breakdown_card()` with metadata, the choice lists show numeric codes (1, 2, 3) instead of labels ("Boys", "Girls", "Mixed") because the `choice_list_name` parameter doesn't match the actual list name in the XLSForm.

**Common mistake**:
```r
# WRONG - using the column name as the choice_list_name
breakdown_card(data, "Q_2_1_boys_girls_school", 
              "School Type",
              metadata = metadata,
              choice_list_name = "Q_2_1_boys_girls_school")  # ❌ Wrong!
```

**Solution**: Find the correct choice list name from the XLSForm:

1. **Open the XLSForm file** and look at the `survey` sheet
2. **Find your question** by the `name` column
3. **Look at the `type` column** - it will say something like `select_one class_monitoed`
4. **Use the list name** from the type (e.g., `class_monitoed`)

```r
# CORRECT - using the actual list name from the XLSForm
breakdown_card(data, "Q_2_1_boys_girls_school", 
              "School Type",
              metadata = metadata,
              choice_list_name = "class_monitoed")  # ✅ Correct!
```

**How to find choice list names programmatically**:
```r
library(readxl)
xlsform <- read_excel("path/to/form.xlsx", sheet = "survey")

# Find the question
question <- xlsform[xlsform$name == "Q_2_1_boys_girls_school", ]
print(question$type)  # Shows: "select_one class_monitoed"

# Extract the list name
list_name <- gsub("select_one |select_multiple ", "", question$type)
# Result: "class_monitoed"
```

**Example from Date Bards exercise**:
- `Q_2_1_boys_girls_school` (School type) → use `choice_list_name = "class_monitoed"`
- `Q_2_2_age_group` (School level) → use `choice_list_name = "level"`
- `Q_1_3_governorate` (Governorate) → use `choice_list_name = "governorate"`

**Why this matters**:
- The `choice_list_name` must match the `list_name` column in the XLSForm's `choices` sheet
- Column names are often different from choice list names
- Without the correct list name, visualizations show codes instead of human-readable labels

### Inconsistent Text Values (NA vs Na)

**Issue**: Free text fields may have inconsistent capitalization (e.g., "NA" vs "Na", "N/A" vs "n/a") that appear as separate categories in visualizations.

**Example from Kitchen Check List**:
- Question 3.10 "Any additional observations?" shows both "NA" (4) and "Na" (4) as separate categories
- Question 4.6 "Additional observations on equipment?" shows "NA" (10) and "Na" (7) separately

**Solution**: Standardize text values in the `prepare_data` function:

```r
prepare_data = function(raw_data) {
  data <- prepare_survey_data(raw_data)
  
  # Standardize NA/Na variations in text fields to be consistent
  # Question 3.10 (obs_mealprep) and 4.6 (obs_equipment)
  text_columns_to_standardize <- c("obs_mealprep", "obs_equipment")
  
  for (col in text_columns_to_standardize) {
    if (col %in% names(data) && is.character(data[[col]])) {
      # Convert "Na" to "NA" for consistency
      data[[col]] <- ifelse(data[[col]] == "Na", "NA", data[[col]])
    }
  }
  
  data
}
```

**Other common standardizations**:
```r
# Standardize yes/no variations
data[[col]] <- tolower(trimws(data[[col]]))  # "Yes" -> "yes", " No " -> "no"

# Standardize N/A variations
data[[col]] <- gsub("^n/?a$", "NA", data[[col]], ignore.case = TRUE)

# Remove extra whitespace
data[[col]] <- trimws(data[[col]])
```

**Why this matters**:
- Inconsistent text values split data across multiple categories
- Makes visualization counts misleading
- Complicates analysis and reporting
- Should be fixed at data preparation stage

**Example fix from Kitchen Check List** (`osm_kitchen_checklist.r`):
- Standardizes "Na" → "NA" in `obs_mealprep` (Q 3.10) and `obs_equipment` (Q 4.6) columns
- Result: Single "NA" category instead of split counts

**Important**: Always verify which column name corresponds to which question by checking the XLSForm `survey` sheet, as column names may not directly match question numbers.

### Free-Text Fields Showing Alphabetically Sorted Words

**Issue**: When using `breakdown_card()` on free-text fields, the visualization shows alphabetically sorted individual words instead of the original responses. For example, "The kitchen is clean" becomes "The clean is kitchen".

**Example**:
```
3.10 Any additional observations?
Based on 15 responses

Also 1 1 1700 30 and and apple around code contact curect data data date date deu...
1 (6.7%)
```

**Root cause**: The `breakdown_card()` function is designed for select questions (single or multi-select). It has logic to normalize multi-select responses by sorting space-separated codes (so "1 2" and "2 1" are treated the same). However, it treats **any value with spaces** as a multi-select response, which breaks free-text fields.

**Solution**: **Do not use `breakdown_card()` for free-text fields**. Instead, either:

1. **Remove free-text fields from the dashboard** (recommended):
```r
# In your dashboard function:
div(
  style = "margin-bottom: 20px;",
  h4("Observations", style = "color: #555; font-size: 16px; margin-bottom: 10px;"),
  p(style = "font-size: 12px; color: #666; margin-bottom: 10px;",
    "Free-text observation fields (Q 3.10, Q 4.6) are available in the CSV exports for detailed review. These fields cannot be meaningfully summarized in the dashboard."),
  fluidRow(
    # Only include structured fields (yes/no, select_one, select_multiple)
    column(4,
      breakdown_card(data, "structured_field", "Structured Question",
                    metadata = metadata, choice_list_name = "list_name")
    )
  )
)
```

2. **Display a simple count** (alternative):
```r
# Count non-empty responses
obs_count <- sum(!is.na(data$obs_mealprep) & 
                 data$obs_mealprep != "" & 
                 data$obs_mealprep != "NA")

div(
  class = "metric-card",
  div("Observations Provided: ", span(obs_count, style = "font-weight: bold;"))
)
```

**How to identify free-text fields**:
```r
library(readxl)
xlsform <- read_excel("path/to/form.xlsx", sheet = "survey")

# Find free-text fields (type = "text")
text_fields <- xlsform[xlsform$type == "text", c("name", "label")]
print(text_fields)

# Example output:
#   name            label
#   obs_mealprep    3.10 Any additional observations?
#   obs_equipment   4.6 Any additional observations on the Kitchen equipment?
```

**Example from Kitchen Check List exercise** (`osm_kitchen_checklist.r`):

- `obs_mealprep` (Q 3.10) - type: `text` - ❌ Removed from dashboard
- `obs_equipment` (Q 4.6) - type: `text` - ❌ Removed from dashboard  
- `obs_facilities` (Q 5.3) - type: `select_one yes_no` - ✅ Kept in dashboard

**Why this matters**:
- Free-text fields are for qualitative data and don't fit quantitative breakdown visualizations
- The alphabetical sorting makes the data completely unreadable
- Free-text responses are still available in CSV exports for manual review
- Dashboards should focus on structured data (metrics, breakdowns, trends)

**Rule of thumb**: If the XLSForm type is `text`, it's free-text and should NOT use `breakdown_card()`.

### Do I Need to Update globalVariables?

**Short answer: Probably not.**

**Issue**: You might see warnings in R CMD check about "no visible binding for global variable" when using column names from your exercise.

**Background**: The `globalVariables()` declaration in `app/R/validation_rules.r` suppresses R CMD check warnings about non-standard evaluation (NSE). This is primarily needed for CRAN packages, but less critical for applications like ours.

**What's Already Declared**:
The `globalVariables()` list already includes:
- ✅ `.data` - Core NSE variable
- ✅ All standard helper columns (`.same_day_check`, `.duration_check`, `.price_outlier_check`, etc.)
- ✅ Output columns (`price_outliers_flagged`)

**When You DON'T Need to Add Variables**:
- Exercise-specific column names (e.g., `school_name`, `monitor_name`, `bread_02`)
- Columns that exist in your survey data
- Variables accessed via `.data[[col]]` syntax

These are handled automatically and don't produce warnings because of how the code uses `.data[[]]` for programmatic access.

**When You MIGHT Need to Add Variables**:
Only if you create a **NEW reusable validation rule** in `validation_rules.r` that uses **NEW helper columns** (starting with `.`):

```r
# Example: New validation rule with a new helper column
custom_validation_rule <- function() {
  function(data, id, data_name) {
    data[[".my_custom_check"]] <- TRUE  # <-- New helper column
    # ... validation logic ...
    affirm::affirm_true(
      data,
      condition = .data[[".my_custom_check"]],
      # ...
    )
  }
}
```

In this case, add `.my_custom_check` to the `globalVariables()` list:

```r
# In app/R/validation_rules.r
utils::globalVariables(c(
  ".data",
  # ... existing variables ...
  ".my_custom_check"  # <-- Add your new helper column
))
```

**For Regular Exercise Addition**:
When adding a new exercise using existing validation rules, you do **NOT** need to modify `globalVariables()` at all. Just use the standard pattern:

```r
run_validations = function(data) {
  affirm::affirm_init(replace = TRUE)
  options('affirm.id_cols' = c("case_id", "date", "monitor_name", ...))
  
  data |>
    duration_check_rule()(id = 1, data_name = "My Exercise") |>
    same_day_submission_rule()(id = 2, data_name = "My Exercise")
}
```

**Summary**:
- 🎯 **Adding a new exercise?** → No changes to `globalVariables()` needed
- 🔧 **Creating a new validation rule with helper columns?** → Add the helper column names
- ⚠️ **Seeing R CMD check warnings?** → Usually safe to ignore for applications (or add specific columns if it bothers you)

**Example**: The Informal Market Price Monitoring exercise uses columns like `bread_02`, `bread_05`, `ID02`, `ID03`, `ID04`, `ID06` - none of these needed to be added to `globalVariables()` because they're regular data columns accessed properly via `.data[[]]`.

---

## Summary

Adding a new exercise involves:

1. **Discover** survey ID and structure using `tools/discover_surveys.r`
2. **Create** exercise file in `app/exercises/`
3. **Define** metadata (id, name, type, description, filter)
4. **Implement** fetch_data() and prepare_data() functions
5. **Configure** run_validations() with affirm
6. **Set** affirm.id_cols for CSV exports
7. **Define** visualizations (optional)
8. **Integrate** metadata for variable/choice labels (optional, recommended)
   - Download XLSForm from SharePoint
   - Process metadata with `process_xlsform()`
   - Update exercise to load metadata
   - Enhance dashboard with metadata labels
9. **Test** thoroughly in the app
10. **Verify** CSV exports contain all needed columns
11. **Document** exercise-specific details

The exercise will be automatically discovered and loaded by the app's registry system.

---

## Additional Resources

- **affirm Package**: https://pcctc.github.io/affirm/
- **Exercise Registry**: `app/R/exercise_registry.r` - How exercises are loaded
- **Validation Rules**: `app/R/validation_rules.r` - All available validation functions
- **Validation Coverage**: `guides/VALIDATION_COVERAGE.md` - **Comprehensive validation mapping for School Feeding exercises**
- **Visualization Helpers**: `app/R/visualization_helpers.r` - UI components
- **Interactive Discovery**: `docs/guides/INTERACTIVE_DISCOVERY.md` - Survey exploration guide
- **Metadata Integration**: `docs/guides/ADDING_METADATA.md` - Detailed metadata guide
- **Metadata Helpers**: `app/R/metadata_helpers.r` - Metadata processing functions
- **Metadata Documentation**: `metadata/README.md` - Metadata directory overview

For questions or issues, consult existing exercise files (`bcm_helpdesk_validation.r`, `osm_helpdesk.r`, `bcm_welcome_meals.r`, `osm_post_office.r`) as working examples.

---

## Example: Adding a New Exercise (Step-by-Step)

This section documents a real-world example of adding new exercises to demonstrate the discovery process.

### Example 1: OSM - Post Office Validation

**Goal**: Add a new exercise for post office validation monitoring.

#### Step 1: View Available Surveys

In your R console, run the discovery script:

```r
source("tools/discover_surveys.r")
```

Then view all available surveys in RStudio's viewer:

```r
View(available_surveys)
```

This opens a sortable/filterable table of all surveys. Use the search box to find "post" or "office".

#### Step 2: Search for the Survey

Search for the post office survey:

```r
# Search in the available_surveys data frame
available_surveys |> 
  filter(grepl("post|office", surveyName, ignore.case = TRUE)) |>
  select(surveyID, xlsFormId, surveyName, xlsFormName, countryName)
```

**Initial Result**: Found BCM survey (ID 5405), but it had no data yet (`originalCsvFile` was empty, survey in "Data collection" phase).

**Alternative Search**: Switched to OSM - Post Office Validation
- **Survey ID**: 5406
- **xlsFormId**: 2168
- **Survey Name**: OSM - Post Office Validation

#### Step 3: Inspect Survey Structure

Once you have the survey ID, inspect the data:

```r
inspect_survey(5406)
```

Then explore the data structure:

```r
# View all columns
names(current_survey_data)

# Check dimensions and sample data
glimpse(current_survey_data)
```

**Results**:
- **Rows**: 50
- **Columns**: 60

**Key columns identified**:
- **Filter column**: `_1_6_Post_office_name` (post office identifier like "101", "79", "bal_circle")
- **Date column**: `date` (format: "2025-08-14")
- **Monitor/Enumerator**: `monitor_name` (e.g., "Eman Al-Jaraydeh", "Malek Al-momani")
- **Timing columns**: `start`, `end`, `_duration`, `_submission_time`, `time`
- **Location**: `governorate` (e.g., "Mafraq", "Zarqa", "Amman", "Irbid")
- **ID columns**: `_id`, `uuid`, `instanceID`

**Survey purpose**: Post office readiness validation
- Equipment checks (iris scanners, card readers, printers, internet)
- WFP materials (signage, posters, leaflets)
- Staff training and beneficiary processes
- Crowd management and accessibility

**Validation considerations**:
- Duration varies widely (93 seconds to 32,323 seconds = 9 hours!)
- Location-based monitoring (post office names)
- No case IDs or phone numbers (facility monitoring, not individual surveys)
- Likely need location visit frequency checks

#### Step 4: Define Validations

Based on the data structure, we determined appropriate validations:

**Validation Strategy**:
1. ✅ **Duration check**: Set bounds (1 min to 1 hour) - we saw durations from 93 seconds to 32,323 seconds
2. ✅ **Same day submission**: Ensure surveys submitted same day as collection
3. ✅ **Late submission**: Check for delays beyond 24 hours
4. ✅ **Location visit frequency**: Post offices shouldn't be visited too frequently (max 2x per month)
5. ✅ **Missing data check**: Ensure data completeness
6. ❌ **No uniqueness checks**: This is facility monitoring (not individual case IDs)
7. ❌ **No proportion checks**: Equipment status doesn't need proportion validation

**CSV Export Columns**:
Set `affirm.id_cols` to include all columns needed for verification:
- `_1_6_Post_office_name` - Which post office
- `date` - When visited
- `monitor_name` - Who conducted the visit
- `governorate` - Location context
- `time`, `_submission_time` - Timing information
- `start`, `end`, `_duration` - For duration verification

#### Step 5: Create Exercise File

Created `app/exercises/osm_post_office.r` with:

```r
exercise <- list(
  id = "5406",
  name = "OSM - Post Office Validation",
  type = "survey",
  description = "Jordan OSM post office validation data quality checks",
  
  filter_column = "_1_6_Post_office_name",
  filter_label = "Post Office",
  
  fetch_data = function(client) {
    with_error_handling(
      fetch_fn = function() {
        databridges_survey_fetch(
          client = client,
          survey_id = 5406L,
          access_type = "full"
        )
      },
      source_name = "OSM - Post Office Validation (Survey 5406)"
    )
  },
  
  prepare_data = function(raw_data) {
    prepare_survey_data(raw_data)
  },
  
  visualizations = function(data) {
    osm_monitoring_dashboard(data)
  },
  
  run_validations = function(data) {
    affirm::affirm_init(replace = TRUE)
    options('affirm.id_cols' = c(
      "_1_6_Post_office_name", "date", "monitor_name", "governorate",
      "time", "_submission_time", "start", "end", "_duration"
    ))
    
    exercise_name <- "OSM - Post Office Validation"
    
    result <- data |>
      duration_check_rule(min_seconds = 60, max_seconds = 3600)(id = 1, data_name = exercise_name) |>
      same_day_submission_rule()(id = 2, data_name = exercise_name) |>
      late_submission_rule()(id = 3, data_name = exercise_name) |>
      location_visit_frequency_rule(
        location_column = "_1_6_Post_office_name",
        max_visits_host = 2,
        max_visits_camp = 2
      )(id = 4, data_name = exercise_name) |>
      missing_data_rule()(id = 5, data_name = exercise_name)
    
    result |> dplyr::select(-dplyr::starts_with("."))
  }
)
```

**Key decisions documented**:
- Used `_1_6_Post_office_name` as filter (allows filtering by post office)
- Set duration bounds to 1 min - 1 hour (60-3600 seconds) to catch anomalies
- Location frequency set to max 2 visits per month per post office
- Created custom `osm_post_office_dashboard()` for visualizations
- No case ID or phone uniqueness checks (facility monitoring, not individual surveys)

#### Step 5b: Create Custom Visualization Dashboard

Created `osm_post_office_dashboard()` in `app/R/visualization_helpers.r` that displays:

**Dashboard Sections**:
1. **Survey Duration Analysis**:
   - Average duration across all surveys
   - Minimum duration (fastest completion)
   - Maximum duration (slowest completion)
   - Target: 5 min minimum, 24 hours maximum

2. **Submission Compliance**:
   - Count of surveys NOT submitted same day as data collection
   - Count of late submissions (>24 hours after visit date)
   - Color-coded: Green for compliant, Yellow/Red for issues

3. **Visit Frequency Monitoring**:
   - Tracks post offices visited too frequently
   - Checks against thresholds: host communities (1x/month), camps (2x/month)

4. **Coverage Analysis**:
   - Post office coverage (visits by post office)
   - Geographic coverage (visits by governorate)
   - Monitor activity (surveys by monitor)

The dashboard uses existing helper functions (`metric_card`, `breakdown_card`) and follows the same visual styling as other dashboards.

Updated exercise file to use:
```r
visualizations = function(data) {
  osm_post_office_dashboard(data)
}
```

#### Step 6: Test the Exercise

[To be continued during testing...]

---

### Example 2: OSM - Informal Price Monitoring (Paused)

**Status**: Paused - awaiting column annotation information

**Progress so far**:

1. ✓ **Survey Discovered**:
   - Official name: "JORDAN - Informal price monitoring in refugee camps"
   - App display name: "OSM - Informal price monitoring"
   - Survey ID: 5409
   - xlsFormId: 2175

2. ✓ **Initial Data Structure** (1 row, 36 columns):
   - `ID01` - Date (2025-10-23)
   - `ID02` - "formal" (market type?)
   - `ID03` - "Azraq_camp" (location)
   - `ID04` - "Bayan Numan" (likely enumerator)
   - `ID06`, `ID07`, `ID08`, `ID11` - Unknown purpose
   - `_duration`, `_submission_time`, `end` - Standard timing columns
   - 6 more columns (31-36) not yet viewed

3. ⏸️ **Next Steps** (when resumed):
   - Identify all 36 column names
   - Determine what each ID column represents
   - Check if `start` column exists
   - Define appropriate validations
   - Configure CSV export columns
   - Create exercise file

**Command used**:
```r
View(available_surveys)  # Find survey in table
inspect_survey(5409)     # Initial inspection
names(current_survey_data)  # Would show all columns
```

---

### Example 3: SF - OSM Date Bards Distribution

**Status**: ✅ Complete

**Goal**: Add a new School Feeding exercise for OSM date bars distribution monitoring at schools.

#### Step 1: Discover the Survey

Search for date bars related surveys:

```r
source("tools/discover_surveys.r")
search_surveys("date")
```

**Result**:
- Survey found: "SF - OSM Date Bards Distribution"
- **Survey ID**: 5421
- **xlsFormId**: 2210
- **baseXlsFormId**: 2197

#### Step 2: Inspect Survey Structure

```r
inspect_survey(5421)
```

**Results**:
- **Rows**: 65
- **Columns**: 69

**Key columns identified**:
- **Filter column**: `Q1_1_interviewer_name` (10 unique interviewers)
- **Date column**: `Q_1_2_visit_date` (⚠️ **Non-standard name**, needs renaming to `date`)
- **Location**: `Q_1_3_governorate`, `directorate`, `camps_hc`, `school`
- **Timing columns**: `start`, `end`, `_duration`, `_submission_time`
- **School info**: `Q_2_1_boys_girls_school`, `Q_2_2_age_group`, `enroll_stud`, `children_atten`
- **Storage questions**: `Q_4_1_DB_astore`, `Q_4_1_DB_store_type`, `Q_4_3_size_adequalte`, `Q_4_3_ventilatd`, `Q_4_5_does_store_have_pallets`, `Q_4_6_away_from_chemicals`
- **Distribution**: `distributed`, `delivery`
- **Issues**: `selling_db`, `heared_selling`, `often_exch`, `dropout`
- **Feedback**: `Q6_any_feedback_school`, `Q6_any_feedback_if_yes`

**Survey purpose**: School-based monitoring of date bars distribution
- Storage conditions monitoring
- Distribution tracking
- Student feedback collection
- Issues identification (selling, exchange, dropout)

#### Step 3: Process Metadata

1. **Located XLSForm file**: `metadata/raw/20251103/School feeding xls/DB_Schools_25_camps_community_20250320.xlsx`

2. **Added mapping** in `app/R/metadata_helpers.r`:
```r
filename_mappings <- list(
  # ... existing mappings ...
  "DB_Schools_25_camps_community_20250320.xlsx" = "5421"
)
```

3. **Processed metadata**:
```r
source("app/R/metadata_helpers.r")

xlsform_path <- "metadata/raw/20251103/School feeding xls/DB_Schools_25_camps_community_20250320.xlsx"
xlsform <- load_xlsform(xlsform_path)
metadata <- process_xlsform(xlsform)

# Save processed metadata
saveRDS(metadata, "metadata/processed/5421_metadata.rds")

# Update combined metadata file
all_metadata <- readRDS("metadata/processed/all_metadata.rds")
all_metadata[["5421"]] <- metadata
saveRDS(all_metadata, "metadata/processed/all_metadata.rds")
```

**Metadata results**:
- **Variable count**: 104
- **Choice lists**: 35

#### Step 4: Define Validations

**Validation Strategy**:
1. ✅ **Duration check**: Set bounds (3 min to 1 hour) for school visit surveys
2. ✅ **Same day submission**: Ensure surveys submitted same day as school visit
3. ✅ **Late submission**: Check for delays beyond 24 hours
4. ✅ **Location visit frequency**: Schools shouldn't be visited too frequently (max 2x per month)
5. ✅ **Missing data check**: Ensure data completeness
6. ❌ **No uniqueness checks**: School monitoring (not individual beneficiary surveys)
7. ❌ **No proportion checks**: Not applicable to this monitoring type

**Key Decision - Date Column Handling**:
- Survey uses `Q_1_2_visit_date` instead of standard `date` column
- **Solution**: Rename in `prepare_data` function to ensure compatibility with validation rules

**CSV Export Columns**:
```r
options('affirm.id_cols' = c(
  "school",                     # School identifier
  "date",                       # Visit date (renamed from Q_1_2_visit_date)
  "Q1_1_interviewer_name",      # Interviewer
  "Q_1_3_governorate",          # Governorate
  "directorate",                # Directorate
  "camps_hc",                   # Camps or host community
  "Q_2_1_boys_girls_school",    # School type
  "Q_2_2_age_group",            # Age group
  "_submission_time",           # When submitted
  "start", "end", "_duration"   # Timing info
))
```

#### Step 5: Create Exercise File

Created `app/exercises/osm_date_bards.r`:

```r
exercise <- list(
  id = "5421",
  name = "SF - OSM Date Bards Distribution",
  type = "survey",
  description = "School Feeding - OSM date bars distribution monitoring and data quality checks",
  
  filter_column = "Q1_1_interviewer_name",
  filter_label = "Interviewer",
  
  fetch_data = function(client) {
    with_error_handling(
      fetch_fn = function() {
        databridges_survey_fetch(
          client = client,
          survey_id = 5421L,
          access_type = "full"
        )
      },
      source_name = "SF - OSM Date Bards Distribution (Survey 5421)"
    )
  },
  
  prepare_data = function(raw_data) {
    # Process with actual date column name
    data <- prepare_survey_data(
      raw_data, 
      date_columns = c("Q_1_2_visit_date")
    )
    
    # Rename to 'date' for consistency with validation rules
    if ("Q_1_2_visit_date" %in% names(data)) {
      data <- data |> dplyr::rename(date = Q_1_2_visit_date)
    }
    
    data
  },
  
  visualizations = function(data) {
    metadata <- tryCatch({
      load_survey_metadata("5421", "../metadata/processed")
    }, error = function(e) {
      NULL
    })
    
    osm_date_bards_dashboard(data, metadata = metadata)
  },
  
  run_validations = function(data) {
    affirm::affirm_init(replace = TRUE)
    options('affirm.id_cols' = c(
      "school", "date", "Q1_1_interviewer_name",
      "Q_1_3_governorate", "directorate", "camps_hc",
      "Q_2_1_boys_girls_school", "Q_2_2_age_group",
      "_submission_time", "start", "end", "_duration"
    ))
    
    exercise_name <- "SF - OSM Date Bards Distribution"
    
    result <- data |>
      duration_check_rule(min_seconds = 180, max_seconds = 3600)(id = 1, data_name = exercise_name) |>
      same_day_submission_rule()(id = 2, data_name = exercise_name) |>
      late_submission_rule()(id = 3, data_name = exercise_name) |>
      location_visit_frequency_rule(
        location_column = "school",
        max_visits_host = 2,
        max_visits_camp = 2
      )(id = 4, data_name = exercise_name) |>
      missing_data_rule()(id = 5, data_name = exercise_name)
    
    result |> dplyr::select(-dplyr::starts_with("."))
  }
)
```

**Key decisions**:
- Used `Q1_1_interviewer_name` as filter (allows filtering by interviewer)
- Set duration bounds to 3 min - 1 hour (180-3600 seconds) for school visits
- School visit frequency set to max 2 visits per month
- ⚠️ **Important**: Renamed `Q_1_2_visit_date` to `date` in `prepare_data` for validation compatibility

#### Step 5b: Create Custom Visualization Dashboard

Created `osm_date_bards_dashboard()` in `app/R/visualization_helpers.r`:

**Dashboard Sections**:
1. **Survey Duration Analysis**:
   - Average duration across all school visits
   - Minimum duration (fastest completion)
   - Maximum duration (slowest completion)
   - Target: 3 min minimum, 1 hour maximum

2. **Submission Compliance**:
   - Count of surveys NOT submitted same day as school visit
   - Count of late submissions (>24 hours after visit date)
   - Color-coded: Green for compliant, Yellow/Red for issues

3. **School Visit Frequency Monitoring**:
   - Tracks schools visited too frequently
   - Max 2 visits per month per school

4. **School Coverage**:
   - School type breakdown (boys/girls/mixed)
   - Age group distribution

5. **Geographic Coverage**:
   - Visits by governorate
   - Camp vs. host community breakdown

6. **Interviewer Activity**:
   - Surveys by interviewer

The dashboard uses metadata integration for displaying human-readable labels from the XLSForm.

#### Step 6: Test the Exercise

**Testing steps**:

1. ✅ **Exercise file loads**:
```r
source('app/R/data_source_helpers.r')
source('app/R/validation_helpers.r')
source('app/R/validation_rules.r')
source('app/R/metadata_helpers.r')
source('app/R/visualization_helpers.r')
source('app/exercises/osm_date_bards.r')
# Result: Exercise loaded successfully
```

2. ✅ **Appears in registry**:
```r
source('app/R/exercise_registry.r')
exercises <- load_exercises()
# Result: 7 exercises total, Date Bards found with ID 5421
```

3. ✅ **Metadata loads**:
```r
metadata <- load_survey_metadata("5421", "metadata/processed")
# Result: 104 variables, 35 choice lists
```

**Exercise successfully integrated!**

#### Step 7: Update Documentation

**New pattern documented**: Non-standard date column names

Added troubleshooting section for surveys that use different date column names (like `Q_1_2_visit_date`, `visit_date`, etc.) instead of the standard `date` column.

**Solution**: Rename the column in `prepare_data` function to maintain compatibility with validation rules.

---

### Key Learnings from Date Bards Exercise

1. **Non-standard date columns**: Some surveys use different date column names. Always check and rename to `date` in `prepare_data` for validation compatibility.

2. **School-based monitoring**: School feeding exercises focus on facilities rather than individual beneficiaries, so no Case ID or phone uniqueness checks needed.

3. **Location visit frequency**: For schools, max 2 visits per month is appropriate regardless of camp vs. host community.

4. **Metadata richness**: School feeding surveys often have extensive metadata (104 variables, 35 choice lists) which greatly improves the user experience with proper labels.

5. **Dashboard customization**: School monitoring dashboards benefit from showing school-specific metrics like school type, age group, and storage conditions.

---

### Example 4: SF - OSM Healthy Meals Distribution HC

**Status**: ✅ Complete

**Goal**: Add School Feeding exercise for OSM healthy meals distribution monitoring in host communities (HC).

**Note**: Similar exercise exists for camps (ID 5423) - we're specifically adding the HC version (ID 5422).

#### Step 1: Discover and Distinguish

Search for healthy meals surveys:

```r
source("tools/discover_surveys.r")

# Look for surveys containing 'healthy' or 'meals'
result <- available_surveys[grep('healthy|meals', available_surveys$surveyName, ignore.case = TRUE), ]
print(result[, c('surveyID', 'surveyName', 'xlsFormId')])
```

**Results**:
- **Survey ID 5422**: "SF - OSM Healthy Meals Distribution HC" ← **This one (Host Community)**
- Survey ID 5423: "SF - OSM Healthy Meals Distribution in Camps" ← Skip this one
- Survey ID 5404: "BCM - Welcome Meals - 2025" ← Different exercise

#### Step 2: Inspect Survey Structure

```r
inspect_survey(5422)
```

**Results**:
- **Rows**: 45
- **Columns**: 69
- ✅ **Standard `date` column** (no renaming needed - simpler than Date Bards!)

**Key columns**:
- **Filter options**: `monitor_name` (11 unique), `cbo_name` (10 unique), `school_name` (45 unique)
- **Date**: `date` (standard name)
- **Timing**: `start`, `end`, `_duration`, `_submission_time`
- **School info**: `school_name`, `school_type`, `enroll_stud`, `children_atten`
- **Meal monitoring**: `meal_quality`, `meal_deli`, `meals_conditions`, `meals_seal`, `distribution`
- **SBCC**: Multiple `sbcc_*` columns for social behavior change communication monitoring

#### Step 3: Process Metadata (Streamlined)

**Located XLSForm**: `metadata/raw/20251103/School feeding xls/hm_schools_hc_20250320.xlsx`

**Quick check of choice lists**:
```r
library(readxl)
survey <- read_excel('...', sheet = 'survey')
# Check key questions to find list names
# school_type → "school_type"
# school_name → "school"
# monitor_name → "field"
# cbo_name → "cbo"
# meal_quality → "meal_quality"
```

**Process metadata** (standard workflow):
```r
# 1. Add mapping
# In metadata_helpers.r: "hm_schools_hc_20250320.xlsx" = "5422"

# 2. Process
source("app/R/metadata_helpers.r")
xlsform <- load_xlsform('metadata/raw/.../hm_schools_hc_20250320.xlsx')
metadata <- process_xlsform(xlsform)
saveRDS(metadata, 'metadata/processed/5422_metadata.rds')

# 3. Update combined file
all_metadata <- readRDS('metadata/processed/all_metadata.rds')
all_metadata[['5422']] <- metadata
saveRDS(all_metadata, 'metadata/processed/all_metadata.rds')
```

**Result**: 90 variables, 36 choice lists

#### Step 4: Create Exercise File (Simplified Pattern)

Created `app/exercises/osm_healthy_meals_hc.r`:

**Key decisions**:
- ✅ **Standard `date` column** - No renaming needed in `prepare_data`
- Used `monitor_name` as filter (11 field monitors)
- Same validation strategy as Date Bards
- Location column: `school_name` for visit frequency check
- CSV exports include: school_name, date, monitor_name, cbo_name, school_type, monitoring type, timing columns

**Simplified `prepare_data` function**:
```r
prepare_data = function(raw_data) {
  # Use standard preparation helper
  # Note: This survey uses standard 'date' column - no renaming needed!
  prepare_survey_data(raw_data)
}
```

This is **simpler** than Date Bards which required date column renaming.

#### Step 5: Create Dashboard

Created `osm_healthy_meals_hc_dashboard()` with:
- Duration analysis
- Submission compliance
- School visit frequency
- School coverage (school type, meal quality rating)
- CBO activity breakdown
- Monitor activity breakdown

**Key choice list mappings** (applied learning from Date Bards):
- `school_type` → `"school_type"`
- `meal_quality` → `"meal_quality"`
- `cbo_name` → `"cbo"`
- `monitor_name` → `"field"`

#### Step 6: Test

```r
source('app/R/exercise_registry.r')
exercises <- load_exercises()
# Result: 8 exercises total, Healthy Meals HC found as ID 5422
```

✅ **Exercise successfully integrated!**

---

### Key Learnings from Healthy Meals HC Exercise

1. **Standard date columns simplify development**: When the survey uses a standard `date` column, no renaming is needed in `prepare_data`. This is the **preferred pattern**.

2. **Similar surveys require careful identification**: Multiple related surveys may exist (HC vs. Camps). Always verify survey ID carefully.

3. **Workflow is now streamlined**: Following the established pattern from Date Bards made this exercise much faster:
   - Discover → Inspect → Check choice lists → Process metadata → Create exercise → Create dashboard → Test

4. **Choice list names first**: Learning from Date Bards, we checked XLSForm choice list names **before** creating the dashboard, avoiding the codes-instead-of-labels issue.

5. **Code reusability**: School feeding dashboards share similar structure (duration, compliance, frequency, coverage), making it easy to adapt existing dashboard code.

### Comparison: Date Bards vs. Healthy Meals HC

| Aspect | Date Bards (5421) | Healthy Meals HC (5422) |
|--------|-------------------|-------------------------|
| **Date column** | `Q_1_2_visit_date` (renamed to `date`) | `date` (standard) ✅ |
| **Rows** | 65 | 45 |
| **Columns** | 69 | 69 |
| **Variables** | 104 | 90 |
| **Choice lists** | 35 | 36 |
| **Filter** | interviewer_name | monitor_name |
| **Location column** | school | school_name |
| **Complexity** | Date renaming required | Standard workflow ✅ |

**Recommendation**: When creating new exercises, check if the `date` column is standard. If so, follow the simpler Healthy Meals HC pattern.

### Validation Coverage Documentation

For detailed validation requirements and implementation status for both exercises, see:
- **[Validation Coverage Document](VALIDATION_COVERAGE.md)** - Complete mapping of requirements to implementations

**Key highlights**:
- ✅ All survey practice validations implemented (duration, same-day, late submission)
- ✅ All required visualizations included
- ✅ Duration parameters updated: 10 min minimum, 24 hours maximum
- ✅ School visit frequency checks prevent duplicates within academic year
- ⚠️ Field-specific numeric validations partially covered by general `missing_data_rule()`
- 📋 All violated records exported to CSV with full context

---

## Example 6: OSM - Informal Market Price Monitoring

**Survey ID**: 5434  
**Survey Type**: Market Price Monitoring  
**Unique Characteristics**: Wide format with 366 variables tracking commodity prices

### What Makes This Survey Different

This exercise represents a **fundamentally different survey type** compared to beneficiary/facility monitoring surveys:

#### 🆕 New Survey Category: Market Price Data

All previous exercises (Welcome Meals, Helpdesks, Post Offices, School Feeding) are **monitoring surveys** that track:
- Beneficiaries visited
- Facilities inspected
- Services delivered
- Compliance checks

**Market Price Monitoring** tracks:
- Commodity prices across multiple markets
- Formal vs. informal market comparisons
- Availability indicators
- Business impact assessments

#### 📊 Wide Format with Many Variables

**Comparison**:

| Exercise Type | Typical Columns | Variable Structure |
|--------------|-----------------|-------------------|
| School Feeding | 69-90 | Mostly 1 column = 1 variable |
| Helpdesk/Post Office | 40-50 | Standard survey format |
| **Market Price** | **284** | **Wide format with patterns** |

**Variable Patterns in Market Price Survey**:
- **Multiple price points** per commodity: `price_01_Milk`, `price_02_Milk`, `price_03_Milk`, etc.
- **Availability flags**: `bread_01`, `bread_02`, `bread_03`, etc.
- **Business impact** fields: `business_impact_01`, `business_impact_02`, `business_impact_03`
- **Commodities tracked**: Bulgur, milk, cheese, eggs, lentils, meat, oil, pasta, rice, salt, tuna, wheat, yogurt, chicken, potatoes, etc.

This results in **366 processed variables** (after XLSForm parsing) vs. typical 40-100 variables for other surveys.

### Implementation Considerations

#### Why Start with Minimal Validations/Visualizations?

Unlike beneficiary monitoring where validation patterns are well-established, market price data requires:

1. **Domain-specific validation rules**:
   - Price ranges (what's reasonable for each commodity?)
   - Price changes over time (what constitutes an outlier?)
   - Availability patterns (seasonal variations?)
   - Formal vs. informal market comparisons

2. **Specialized visualizations**:
   - Price trends over time
   - Commodity-specific dashboards
   - Market type comparisons
   - Geographic price variations
   - Availability heat maps

3. **Data structure understanding**:
   - How do the `price_01_`, `price_02_`, `price_03_` fields relate?
   - Are these different measurement units, different shops, or different time points?
   - How should they be aggregated?

**Decision**: Start with basic survey practice validations (duration, submission timing) and simple market overview visualizations, then **iteratively add** commodity-specific analysis based on stakeholder requirements.

### Steps Taken

#### Step 1: Discover Survey

```r
source("tools/discover_surveys.r")
search_surveys("informal")
inspect_survey(5434)
```

**Key findings**:
- 444 rows
- 284 columns
- Survey category: "Market (for data collection progress only)"
- Sub-category: "Market Prices"

#### Step 2: Identified Unique Structure

**ID columns**:
- `ID01`: Date
- `ID02`: Market type (formal/informal) 
- `ID03`: Camp location
- `ID04`: Field monitor name

**Commodity columns** (examples):
- `Bulgur_packaged`
- `Powdered_Milk`
- `price_01_Milk`, `price_02_Milk`, `price_03_Milk`
- `bread_01`, `bread_02`, `bread_03`
- `rice_white`, `sugar_ava`, `salt_ava`

#### Step 3: Process Metadata

Added mapping to `metadata_helpers.r`:

```r
"Informal_Price_Monitoring_in_Refugee_Camps.xlsx" = "5434"
```

Fixed metadata processing functions to handle XLSForms without label columns:
- Updated `extract_variable_labels()` to create label column if missing
- Updated `extract_choice_lists()` to create label column if missing

**Result**: 366 variables, 29 choice lists processed successfully.

#### Step 4: Create Minimal Exercise File

Created `app/exercises/osm_informal_market_price.r` with:

**Metadata**:
```r
id = "5434"
name = "OSM - Informal Market Price Monitoring"
filter_column = "ID04"  # Field monitor name
```

**Data preparation**:
```r
prepare_data = function(raw_data) {
  data <- prepare_survey_data(raw_data)
  # Rename ID01 to date for consistency with other exercises
  if ("ID01" %in% names(data)) {
    data <- data |> dplyr::rename(date = ID01)
  }
  data
}
```

**Minimal validations** (survey practice only):
- Duration check (3-120 minutes)
- Same-day submission
- Late submission

**No commodity-specific validations yet** - waiting for stakeholder requirements.

**Basic visualizations**:
- Survey practice metrics (duration, submission timing)
- Market overview (type, location, monitor)
- Data structure note explaining the 366 variables

#### Step 5: Test

```r
# From app directory
source("R/exercise_registry.r")
exercises <- load_exercises()
# Result: 10 exercises loaded, including ID 5434
```

✅ **Exercise successfully integrated with minimal implementation!**

### Key Learnings

#### 1. Not All Surveys Follow the Same Patterns

**Previous assumption**: All surveys are beneficiary/facility monitoring with similar validation needs.

**Reality**: Different survey types (market price, household surveys, etc.) require:
- Different validation strategies
- Different visualization approaches  
- Different data structure considerations

**Documentation update needed**: Our guides should acknowledge different survey categories and provide guidance for each.

#### 2. Wide Format Data Requires Special Handling

With 366 variables, challenges include:
- **Column selection**: Which variables matter most for quality monitoring?
- **Aggregation**: How to summarize across commodity types?
- **Visualization**: Can't use standard breakdown cards for all 366 variables
- **Performance**: Loading and processing many columns

**Best practice**: Start minimal, identify key variables, build targeted visualizations.

#### 3. Metadata Processing Robustness

Encountered issue: Some XLSForm files don't have `label` columns in survey/choices sheets.

**Fix applied**:
- Check if `label` column exists before using it
- Create label from name if missing
- Prevents errors when processing diverse XLSForm files

**Lesson**: Always code defensively when processing external data formats.

#### 4. Iterative Development is Valid

Not every exercise needs full implementation immediately.

**Valid approach**:

1. ✅ Get basic structure working
2. ✅ Load data successfully
3. ✅ Show basic metrics
4. ⏳ Add domain-specific features based on actual requirements

This is especially important for:
- New survey types without established patterns
- Complex data structures needing stakeholder input
- Surveys with specialized domain knowledge requirements

### Next Steps for Market Price Exercise

When stakeholder requirements are defined, consider adding:

#### Potential Validations
- Price range checks (min/max by commodity)
- Price change flags (outlier detection)
- Availability consistency checks
- Cross-market price comparisons
- Required commodity coverage

#### Potential Visualizations  
- **Price Trends**: Line charts showing price changes over time by commodity
- **Commodity Dashboards**: One dashboard per major commodity group
- **Market Comparison**: Formal vs. informal price differences
- **Geographic Heatmaps**: Price variations by camp location
- **Availability Matrix**: Which commodities are available where

#### Data Analysis Questions
- What's the relationship between `price_01_`, `price_02_`, `price_03_`?
- Are there reference prices to compare against?
- What constitutes a "significant" price change?
- How should seasonal variations be handled?
- What are acceptable availability rates?

### Documentation Implications

This example highlights the need for:

1. **Survey Type Classification** in documentation:
   - Beneficiary monitoring
   - Facility monitoring
   - Market price monitoring
   - Household surveys
   - ...and guidance for each type

2. **Validation Rule Categories**:
   - Survey practice (universal)
   - Data quality (survey-specific)
   - Domain logic (requires expertise)

3. **Iterative Development Guidance**:
   - It's OK to start minimal
   - Add features based on requirements
   - Document what's missing and why

4. **Wide Format Data Handling**:
   - How to identify key variables
   - When to aggregate vs. show individual columns
   - Performance considerations

---

### Updated Survey Type Comparison

| Survey Type | Examples | Typical Columns | Key Focus | Validation Strategy |
|------------|----------|-----------------|-----------|---------------------|
| **Facility Monitoring** | Helpdesk, Post Office, Kitchen | 40-80 | Compliance, observations | Completeness, frequency, facilities |
| **Beneficiary Monitoring** | Welcome Meals | 40-60 | Service delivery, satisfaction | Coverage, timeliness, demographics |
| **School Feeding** | Date Bards, Healthy Meals | 70-100 | Meal distribution, quality | Duration, frequency, school tracking |
| **Market Price** | Informal Price Monitoring | 280+ | Commodity prices, availability | Price ranges, availability patterns, market comparisons |

Each type requires **tailored validation and visualization approaches**.


