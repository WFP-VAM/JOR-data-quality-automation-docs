# Quick Start: Adding a New Exercise

**Fast track guide for adding a new survey to the data quality app.**

## 🚀 Quick Workflow

### 1. Discover Your Survey (5 minutes)

```r
# Run the discovery script
source("tools/discover_surveys.r")

# Option 1: Search for your survey by keyword
search_surveys("FSOM")        # Searches across multiple name columns
search_surveys("welcome")      # Try different keywords

# Option 2: If search doesn't work, browse available surveys
# The script automatically lists surveys by country when it loads
# Look for your survey in the list above

# Option 3: If you know the survey ID, inspect it directly
inspect_survey(5400)  # Use your survey ID

# Explore the data
View(current_survey_data)
names(current_survey_data)
```

**Note down**:

- Survey ID: `_____`
- Filter column: `_____`
- Key columns: `_____`
- Date column: `_____`
- ID column: `_____`

### 2. Create Exercise File (10 minutes)

Once you've found the survey you want to add, you're ready to add the relevant app files. Before doing so, set up a new branch off of `develop` to store your work:

```bash
# Switch to develop and pull latest changes
git checkout develop
git pull origin develop

# Create a new branch for your work
git checkout -b feature/add-survey-name  # Replace with descriptive name

# Example: git checkout -b feature/add-fsom-survey
```

Now create your exercise file:

```bash
# Create new file
touch app/exercises/bcm_welcome_meals.r
```

Copy this template and fill in your survey details:

```r
# [Survey Name] Exercise Definition
# [Organization] - [Survey Type] - Survey ID [XXXX]

exercise <- list(
  # Basic metadata
  id = "XXXX",                        # Your survey ID
  name = "[Survey Name]",             # Display name
  type = "survey",
  description = "[Description]",
  
  # Filter configuration
  filter_column = "[Column Name]",    # From discovery
  filter_label = "[Label]",           # Human-readable
  
  # Data fetching
  fetch_data = function(client) {
    with_error_handling(
      fetch_fn = function() {
        databridges_survey_fetch(
          client = client,
          survey_id = XXXXL,          # Integer with L suffix
          access_type = "full"
        )
      },
      source_name = "[Survey Name] (Survey XXXX)"
    )
  },
  
  # Data preparation
  prepare_data = function(raw_data) {
    # Use standard preparation helper
    data <- prepare_survey_data(raw_data)
    
    # Derive date column from _submission_time if date column doesn't exist
    # Some surveys don't have a 'date' column but validation rules need it
    if (!"date" %in% names(data) && "_submission_time" %in% names(data)) {
      data <- data |>
        dplyr::mutate(
          date = as.Date(substr(.data[["_submission_time"]], 1, 10))
        )
    }
    
    # Add any exercise-specific transformations here
    # Example: Standardize text fields
    # if ("meal_type" %in% names(data)) {
    #   data <- data |> mutate(meal_type = trimws(tolower(meal_type)))
    # }
    
    data
  },
  
  # Visualizations (optional - set to NULL if not needed)
  visualizations = function(data) {
    your_dashboard_function(data)
  },
  
  # Validation function using affirm (required)
  run_validations = function(data) {
    # Initialize affirm session and set ID columns for CSV export
    affirm::affirm_init(replace = TRUE)
    options('affirm.id_cols' = c("Case_ID", "date", "Enumerator_Name", 
                                   "_submission_time", "start", "end", "_duration"))
    
    exercise_name <- "[Survey Name]"
    
    # Run all validations in sequence and remove helper columns at the end
    result <- data |>
      duration_check_rule()(id = 1, data_name = exercise_name) |>
      same_day_submission_rule()(id = 2, data_name = exercise_name) |>
      late_submission_rule()(id = 3, data_name = exercise_name) |>
      missing_data_rule()(id = 4, data_name = exercise_name)
    
    # Remove all helper columns before returning
    result |> dplyr::select(-dplyr::starts_with("."))
  }
)
```

### 3. Test (5 minutes)

```r
# Restart Shiny app
shiny::runApp("app")
```

**Check**:

- ✓ Exercise appears in dropdown
- ✓ Data loads
- ✓ Validations run
- ✓ Filters work
- ✓ Dashboard displays

## 📚 Full Documentation

See **`docs/ADDING_NEW_EXERCISES.md`** for:

- Complete step-by-step guide
- All validation rules with examples
- Visualization components reference
- Troubleshooting guide
- Best practices

## 🔧 Available Validation Rules

```r
# Standard rules
duration_check_rule()           # Survey duration bounds
same_day_submission_rule()      # Same-day submission
late_submission_rule()          # Within 24h submission
missing_data_rule()             # Missing data detection

# ID/Phone uniqueness
case_id_uniqueness_rule()       # Case ID per day
phone_uniqueness_rule()         # Phone per day

# Frequency checks
location_visit_frequency_rule() # Location visit limits

# Proportion checks
proportion_check_rule()         # E.g., max 10% complaints
```

All rules accept custom parameters - see full docs for details.

## 🎨 Available Visualization Components

```r
# Metric cards
metric_card(data, "column_name", "Label")
time_metric_card(data, "time_column", "Label")
satisfaction_metric_card(data, "satisfaction_column", "Label")

# Breakdowns
breakdown_card(data, "category_column", "Label")

# Layouts
fluidRow(
  column(4, metric_card(...)),
  column(4, metric_card(...)),
  column(4, metric_card(...))
)
```

## ❓ Common Issues

**Exercise doesn't appear**

- Check file is in `app/exercises/`
- Verify syntax with `source("app/exercises/your_file.r")`
- Restart app

**Data won't load**

- Verify survey ID with `inspect_survey(ID)`
- Check API credentials in `data_bridges_api_config.yaml`
- Ensure survey_id is integer: `5400L` not `"5400"`

**Column not found**

- Run `inspect_survey(ID)` to see actual column names
- Check spelling and case sensitivity
- Update column references in exercise file

## 📁 Key Files

- `tools/discover_surveys.r` - Survey ID discovery script
- `app/exercises/*.r` - Exercise definitions
- `app/R/validation_rules.r` - Reusable validation functions
- `app/R/visualization_helpers.r` - Dashboard components
- `docs/ADDING_NEW_EXERCISES.md` - Complete guide

## 🤝 Getting Help

1. Check existing exercises: `app/exercises/bcm_helpdesk_validation.r`
2. Read full docs: `docs/ADDING_NEW_EXERCISES.md`
3. Run discovery: `source("tools/discover_surveys.r")`
4. Test with known survey: `inspect_survey(5387)`

---

**Total Time**: ~20 minutes from discovery to working exercise!

