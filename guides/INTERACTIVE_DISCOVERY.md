# Interactive Survey Discovery Guide

## Overview

This guide demonstrates the **interactive workflow** for discovering survey IDs and understanding survey structure before creating a new exercise in the Data Quality App.

---

## Why Interactive Discovery?

Before you can create an exercise, you need to know:

1. **Survey ID** - The unique identifier for your data source
2. **Column names** - What fields are available in the survey
3. **Filter options** - Which columns make good filters (enumerators, locations, etc.)
4. **Data types** - Are dates stored as strings? Are numbers formatted correctly?
5. **Data quality** - What's the structure? Any missing data patterns?

**Previously**: Developers had to manually inspect the API, guess column names, and iterate through errors.

**Now**: Interactive discovery script automates the exploration and provides instant insights!

---

## The Interactive Workflow

### Visual Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    INTERACTIVE DISCOVERY                     │
└─────────────────────────────────────────────────────────────┘
                             │
                             ▼
        ┌───────────────────────────────────────┐
        │   1. Run Discovery Script             │
        │   source("tools/discover_surveys.r")        │
        └───────────────────────────────────────┘
                             │
                             ▼
        ┌───────────────────────────────────────┐
        │   2. Search for Your Survey           │
        │   search_surveys("welcome meals")     │
        └───────────────────────────────────────┘
                             │
                             ▼
        ┌───────────────────────────────────────┐
        │   3. Inspect Survey Structure         │
        │   inspect_survey(5400)                │
        └───────────────────────────────────────┘
                             │
                             ▼
        ┌───────────────────────────────────────┐
        │   4. Explore Data Interactively       │
        │   View(current_survey_data)           │
        │   names(current_survey_data)          │
        │   table(current_survey_data$column)   │
        └───────────────────────────────────────┘
                             │
                             ▼
        ┌───────────────────────────────────────┐
        │   5. Document Your Findings           │
        │   • Survey ID                         │
        │   • Filter columns                    │
        │   • Key metric columns                │
        │   • Date columns                      │
        │   • ID columns                        │
        └───────────────────────────────────────┘
                             │
                             ▼
        ┌───────────────────────────────────────┐
        │   6. Create Exercise File             │
        │   Use findings in exercise definition │
        └───────────────────────────────────────┘
```

---

## Common Scenarios

### Scenario A: You have an xlsFormId but need the survey ID

**Situation**: You know the form ID (e.g., 2168) but need the corresponding survey ID.

**Solution**:
```r
library(reticulate)
library(dplyr)

# Setup client
use_python(".venv/bin/python")
data_bridges_knots <- import("data_bridges_knots")
client <- data_bridges_knots$DataBridgesShapes("data_bridges_api_config.yaml")

# Get all surveys
surveys <- client$get_household_surveys_list(page = 1L)

# Find by xlsFormId
surveys |> 
  filter(xlsFormId == 2168) |>
  select(surveyID, xlsFormId, xlsFormName, surveyStatus)
```

**Example output**:
```
  surveyID xlsFormId                   xlsFormName surveyStatus
      5404      2168  BCM - Welcome meals - 20251021       Active
```

**Result**: xlsFormId 2168 → **surveyID 5404** ✓

**Why this matters**: Survey IDs can change when forms are republished, but xlsFormId is more stable. Always verify the current surveyID.

---

### Scenario B: You know the survey name but not the ID

**Situation**: You know it's called "BCM - Welcome Meals" but don't have the ID.

**Solution**: Use the discovery script's search function (see Step 2 below).

---

### Scenario C: You have nothing - need to explore available surveys

**Situation**: Starting from scratch, need to see what surveys exist.

**Solution**: Use the full discovery workflow (see steps below).

---

## Step-by-Step Walkthrough

### Step 1: Run the Discovery Script

Open R/RStudio and run:

```r
source("discover_surveys.r")
```

**What happens**:

- ✓ Connects to DataBridges API client
- ✓ Loads helper functions: `search_surveys()` and `inspect_survey()`
- ✓ Attempts to list available surveys (if API supports it)
- ✓ Provides instructions for next steps

**Expected output**:
```
========================================
SETTING UP DATABRIDGES CLIENT
========================================
✓ Client connected successfully!

========================================
STEP 1: DISCOVER AVAILABLE SURVEYS
========================================
Fetching list of household surveys...

Found 15 surveys
[Table of surveys displayed]

========================================
READY FOR DISCOVERY!
========================================
Use the functions above to find your survey ID,
then update your exercise file accordingly.
```

**Note**: If listing surveys fails (API doesn't support it), you'll see alternative methods.

---

### Step 2: Search for Your Survey

Use the `search_surveys()` function to find surveys by keyword:

```r
# Search by keyword
search_surveys("welcome")
search_surveys("BCM")
search_surveys("meals")
```

**Example interaction**:

```r
> search_surveys("welcome")

Found 2 survey(s) matching 'welcome':

  surveyId                     surveyTitle
      5400            BCM - Welcome Meals
      5401  OSM - Welcome Meals Distribution
```

**What to do**:

- ✓ Note the Survey ID (e.g., `5400`)
- ✓ Verify this is the survey you want
- ✓ Try different search terms if needed

**Tips**:

- Search is case-insensitive
- Partial matches work (e.g., "wel" finds "welcome")
- Try organization names: "BCM", "OSM", "WFP"
- Try survey types: "helpdesk", "monitoring", "distribution"

---

### Step 3: Inspect Survey Structure

Once you have a survey ID, inspect its detailed structure:

```r
inspect_survey(5400)
```

**This comprehensive function shows**:

#### A. Data Overview
```
✓ Successfully loaded survey 5400

========================================
SURVEY DATA OVERVIEW
========================================
Rows: 1250
Columns: 45
```

#### B. Column Names (First 30)
```
========================================
COLUMN NAMES (First 30)
========================================
 1. _id
 2. _submission_time
 3. _duration
 4. date
 5. Enumerator_Name
 6. Beneficiary_ID
 7. meal_quality_good
 8. meal_quantity_sufficient
 9. meal_temperature_appropriate
10. has_complaint
11. distribution_location
12. meal_type
...
```

#### C. Potential Filter Columns
```
========================================
POTENTIAL FILTER COLUMNS
========================================
These columns might be good for filtering:

  • Enumerator_Name ( 8 unique values )
  • distribution_location ( 12 unique values )
  • site_name ( 5 unique values )
```

**How it identifies filter columns**:

- Contains keywords like "enum", "enumerator", "name", "location", "site"
- Has between 2-50 unique values (not too few, not too many)
- This helps you choose a good filter for the UI

#### D. Sample Data
```
========================================
SAMPLE DATA (First 5 rows, first 10 cols)
========================================
[First 5 rows of data displayed with actual values]
```

**What to look for**:

- Date formats: Are they consistent?
- Text values: Any unusual formatting?
- Missing data: Many NAs?
- Numeric ranges: Do values make sense?

---

### Step 4: Explore Data Interactively

The `inspect_survey()` function saves the data as `current_survey_data`. Now you can explore!

#### Basic Exploration

```r
# Open in RStudio viewer (great for visual inspection)
View(current_survey_data)

# See all column names
names(current_survey_data)

# Check data structure
str(current_survey_data)

# Get dimensions
dim(current_survey_data)
nrow(current_survey_data)
ncol(current_survey_data)
```

#### Explore Specific Columns

```r
# Check unique values in a column
unique(current_survey_data$Enumerator_Name)

# Count records by category
table(current_survey_data$distribution_location)

# Cross-tabulation
table(current_survey_data$Enumerator_Name, 
      current_survey_data$distribution_location)

# Check for missing values
sum(is.na(current_survey_data$Beneficiary_ID))
colSums(is.na(current_survey_data))
```

#### Analyze Key Metrics

```r
# Summary statistics
summary(current_survey_data$meal_quality_good)

# Calculate percentages for Yes/No columns
table(current_survey_data$has_complaint)
prop.table(table(current_survey_data$has_complaint)) * 100

# Check date range
range(current_survey_data$date, na.rm = TRUE)

# Check time distributions
hist(as.numeric(current_survey_data$`_duration`), 
     main = "Survey Duration Distribution",
     xlab = "Duration (seconds)")
```

#### Validate Data Quality

```r
# Check for duplicates
sum(duplicated(current_survey_data$Beneficiary_ID))

# Missing data by row
rowSums(is.na(current_survey_data))

# Find rows with lots of missing data
which(rowSums(is.na(current_survey_data)) > 10)

# Check date consistency
summary(as.Date(current_survey_data$date))
```

---

### Step 5: Document Your Findings

Create a checklist with all the information you discovered:

#### Findings Template

```markdown
## Survey Discovery Notes - [Survey Name]

**Discovery Date**: 2025-10-23
**Discovered By**: [Your Name]

### Basic Information

- **Survey ID**: 5400
- **Survey Name**: BCM - Welcome Meals
- **Record Count**: 1,250 records
- **Column Count**: 45 columns
- **Date Range**: 2025-01-01 to 2025-10-20

### Filter Configuration

- **Recommended Filter Column**: `Enumerator_Name`
- **Filter Label**: "Enumerator"
- **Unique Filter Values**: 8
- **Alternative Filters**: 
  - `distribution_location` (12 values)
  - `site_name` (5 values)

### Key Columns for Validations

- **ID Column**: `Beneficiary_ID` (for uniqueness checks)
- **Phone Column**: `Interviewee_Phone_Number` (if exists)
- **Date Column**: `date` (needs conversion to Date class)
- **Duration Column**: `_duration` (in seconds)
- **Submission Column**: `_submission_time` (timestamp)

### Key Columns for Visualizations
**Meal Quality Metrics**:

- `meal_quality_good` (Yes/No)
- `meal_quantity_sufficient` (Yes/No)
- `meal_temperature_appropriate` (Yes/No)

**Service Quality**:

- `served_on_time` (Yes/No)
- `staff_courteous` (Yes/No)
- `has_complaint` (Yes/No)

**Distribution Details**:

- `distribution_location` (categorical)
- `meal_type` (categorical)

**Demographics**:

- `beneficiary_gender` (categorical)
- `beneficiary_age_group` (categorical)

### Data Quality Notes

- Date format: YYYY-MM-DD (needs conversion)
- Duration range: 180-3600 seconds (3 min - 1 hour)
- Missing data: ~5% average across columns
- Duplicates: 0 duplicate Beneficiary_IDs found

### Validation Rule Recommendations
1. ✓ Duration check (300-3600 seconds / 5 min - 1 hour)
2. ✓ Same-day submission
3. ✓ Late submission (within 24 hours)
4. ✓ Beneficiary ID uniqueness per day
5. ✓ Complaint rate check (max 10%)
6. ✓ Missing data check (max 30% per row)

### Next Steps

- [ ] Create exercise file: `app/exercises/bcm_welcome_meals.r`
- [ ] Implement data preparation
- [ ] Add validation rules
- [ ] Create custom dashboard
- [ ] Test in app
```

---

### Step 6: Create the Exercise File

Now you have everything you need! Follow the main guide using your documented findings:

```r
# See full instructions in:
# - docs/ADDING_NEW_EXERCISES.md (comprehensive)
# - QUICK_START_NEW_EXERCISE.md (fast reference)
```

**Use your findings to fill in**:

- Survey ID
- Filter column and label
- Validation rule parameters
- Visualization column names

---

## Real-World Example 1: Survey Name Known

### Scenario
You need to add a new "BCM - Welcome Meals" survey to the app.

### Interactive Session

```r
# 1. Start discovery
> source("tools/discover_surveys.r")
✓ Client connected successfully!

# 2. Search for survey
> search_surveys("welcome")
Found 2 survey(s) matching 'welcome':
  surveyId            surveyTitle
      5404  BCM - Welcome Meals

# 3. Inspect structure
> inspect_survey(5404)
✓ Successfully loaded survey 5404
Rows: 1250
Columns: 45

POTENTIAL FILTER COLUMNS:
  • Enumerator_Name ( 8 unique values )
  • distribution_location ( 12 unique values )

# 4. Explore data
> View(current_survey_data)
[Opens in viewer]

> names(current_survey_data)
[Shows all 45 column names]

> table(current_survey_data$Enumerator_Name)
Ahmad     6
Fatima    8
Hassan   10
[...]

> table(current_survey_data$has_complaint)
No   Yes 
1187   63

> prop.table(table(current_survey_data$has_complaint)) * 100
No      Yes 
94.96   5.04

# ✓ Complaint rate is ~5% (well under 10% threshold)

> summary(as.numeric(current_survey_data$`_duration`))
   Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
  180.0   420.0   540.0   612.3   720.0  3200.0

# ✓ Duration mostly 7-12 minutes, max ~50 minutes

# 5. Document findings
# [Created notes file with all information]

# 6. Create exercise
# [Created app/exercises/bcm_welcome_meals.r using findings]
```

**Result**: Complete exercise created in ~20 minutes with confidence that all column names and thresholds are correct!

---

## Real-World Example 2: Troubleshooting Wrong Survey ID

### Scenario
Created exercise with survey ID 5400, but app throws error when loading data. You have xlsFormId 2168.

### Problem
```r
> shiny::runApp("app")
# Select "BCM - Welcome Meals"

ERROR fetching from BCM - Welcome Meals (Survey 5400): 
TypeError: object of type 'NoneType' has no len()
```

### Troubleshooting Session

```r
# 1. Setup client
> library(reticulate)
> library(dplyr)
> use_python(".venv/bin/python")
> data_bridges_knots <- import("data_bridges_knots")
> client <- data_bridges_knots$DataBridgesShapes("data_bridges_api_config.yaml")

# 2. Get all surveys
> surveys <- client$get_household_surveys_list(page = 1L)
> cat("Found", nrow(surveys), "surveys\n")
Found 20 surveys

# 3. Look at the data structure
> surveys |> 
    select(surveyID, xlsFormId, xlsFormName) |>
    head(10)
    
   surveyID xlsFormId                                   xlsFormName
       5409      2175  Informal price monitoring in refugee camps - 20251022
       5408      2172                              OSM - Helpdesk - 20251022
       5407      2169               BCM - Post offices validation - 20251021
       5406      2170               OSM - Post offices validation - 20251021
       5405      2169               BCM - Post offices validation - 20251021
       5404      2168                         BCM - Welcome meals - 20251021
       5403      2167                   BCM - Helpdesk validation - 20251021
       5402      2166                             BCM - Helpdesks - 20251021
       5398      2160                                            mvamRMnEnew
       5397      2159                     OSM - Post Office Validation - OLD

# 4. Find our survey by xlsFormId
> surveys |> 
    filter(xlsFormId == 2168) |>
    select(surveyID, xlsFormId, xlsFormName, surveyStatus)
    
  surveyID xlsFormId                   xlsFormName surveyStatus
      5404      2168  BCM - Welcome meals - 20251021       Active

# 🎯 FOUND IT! Survey ID is 5404, not 5400!

# 5. Update exercise file
# Updated app/exercises/bcm_welcome_meals.r:
#   - Line 2: Survey ID 5404 (comment)
#   - Line 7: id = "5404"
#   - Line 22: survey_id = 5404L
#   - Line 26: source_name = "BCM - Welcome Meals (Survey 5404)"

# 6. Test the fix
> shiny::runApp("app")
# Select "BCM - Welcome Meals"
✓ Data loads successfully!
✓ 1250 records loaded
✓ All validations running
✓ Dashboard displaying

# SUCCESS! 🎉
```

### What We Learned

1. **Survey ID ≠ xlsFormId**: They're different identifiers
2. **Survey IDs can change**: When forms are republished, new survey IDs are created
3. **xlsFormId is more stable**: It stays consistent across versions
4. **Always verify from API**: Don't guess - check the actual surveys list
5. **The surveys dataframe is essential**: It has all the mappings you need

### Time Saved

- **Without troubleshooting guide**: Could take hours of trial and error
- **With this approach**: 5 minutes to identify and fix the issue

**Result**: Working exercise with correct survey ID!

---

## Troubleshooting Interactive Discovery

### Issue: Script won't connect to API

**Symptoms**:
```
Error: Python module 'data_bridges_knots' not found
```

**Solutions**:
```bash
# Check Python environment
which python
python --version

# Verify virtual environment
source .venv/bin/activate
pip list | grep data-bridges

# Reinstall if needed
pip install data-bridges-knots
```

---

### Issue: Survey list not available

**Symptoms**:
```
⚠ Could not fetch survey list automatically.
Error: method not available
```

**Solutions**:

- This is expected if API doesn't support listing
- Use `inspect_survey(ID)` with a known ID
- Check DataBridges dashboard for survey list
- Ask your data team

---

### Issue: Permission denied for survey

**Symptoms**:
```
Error loading survey: permission denied
```

**Solutions**:

1. Check `data_bridges_api_config.yaml` credentials
2. Verify SCOPES include `vamdatabridges_household-fulldata_get`
3. Contact API administrator for access to this survey
4. Try `access_type = "base"` instead of "full"

---

### Issue: Column names don't match expectations

**Symptoms**:

- Expected column `meal_quality` but not found
- Column names have unexpected prefixes/suffixes

**Solutions**:
```r
# Search column names
grep("quality", names(current_survey_data), value = TRUE, ignore.case = TRUE)

# Find similar names
grep("meal", names(current_survey_data), value = TRUE, ignore.case = TRUE)

# Check exact spelling
names(current_survey_data)[grepl("^meal", names(current_survey_data))]
```

---

## Best Practices

### 1. Always Run Discovery First
Don't guess column names or structure. Discovery takes 5 minutes and saves hours of debugging.

### 2. Document Your Findings
Create a findings document for each survey. Future developers (including yourself!) will thank you.

### 3. Explore Beyond the Basics
Don't just look at column names:

- Check data distributions
- Look for outliers
- Validate assumptions about data quality
- Test your validation logic with real data

### 4. Test with Real Data
Use `current_survey_data` to test validation logic before adding to exercise:

```r
# Test a validation idea
test_data <- current_survey_data

# Check if case IDs are unique per day
test_data |>
  group_by(date) |>
  summarise(
    total = n(),
    unique_ids = n_distinct(Beneficiary_ID),
    has_dupes = total != unique_ids
  ) |>
  filter(has_dupes)

# If this shows duplicates, case_id_uniqueness_rule is needed!
```

### 5. Keep the Script Updated
If you find the discovery script missing features, enhance it! It's a living tool.

---

## Advanced Discovery Techniques

### Custom Exploration Scripts

Save common exploration patterns:

```r
# Create: explore_survey.r
explore_columns <- function(data, pattern) {
  matching <- grep(pattern, names(data), value = TRUE, ignore.case = TRUE)
  cat("Found", length(matching), "columns matching '", pattern, "':\n")
  for (col in matching) {
    cat("\n", col, ":\n")
    if (is.numeric(data[[col]])) {
      print(summary(data[[col]]))
    } else {
      print(head(table(data[[col]], useNA = "always"), 10))
    }
  }
}

# Usage
explore_columns(current_survey_data, "meal")
explore_columns(current_survey_data, "quality")
```

### Data Quality Report

```r
# Generate comprehensive data quality report
data_quality_report <- function(data) {
  cat("========================================\n")
  cat("DATA QUALITY REPORT\n")
  cat("========================================\n\n")
  
  # Completeness
  cat("COMPLETENESS:\n")
  completeness <- colMeans(!is.na(data)) * 100
  low_completeness <- names(completeness[completeness < 80])
  cat("Columns with <80% completeness:\n")
  print(completeness[low_completeness])
  
  # Duplicates
  cat("\n\nDUPLICATES:\n")
  id_cols <- grep("id|ID", names(data), value = TRUE)
  for (col in id_cols) {
    dupe_count <- sum(duplicated(data[[col]]))
    if (dupe_count > 0) {
      cat(col, ":", dupe_count, "duplicates\n")
    }
  }
  
  # Date ranges
  cat("\n\nDATE RANGES:\n")
  date_cols <- grep("date|Date|time", names(data), value = TRUE)
  for (col in date_cols) {
    cat(col, ":\n")
    print(range(data[[col]], na.rm = TRUE))
  }
}

# Usage
data_quality_report(current_survey_data)
```

---

## Summary

The interactive discovery workflow provides:

- ✓ **Fast survey ID discovery** - Search and find surveys in seconds  
- ✓ **Automated structure inspection** - See columns, types, and samples automatically  
- ✓ **Interactive exploration** - Dive deep with R commands  
- ✓ **Data quality insights** - Understand your data before coding  
- ✓ **Validation testing** - Test logic with real data first  
- ✓ **Documentation foundation** - Gather all info needed for exercise creation  

**Time saved**: What used to take hours of trial-and-error now takes ~5-10 minutes of exploration!

---

## Related Documentation

- **`tools/discover_surveys.r`** - The interactive discovery script
- **`docs/ADDING_NEW_EXERCISES.md`** - Complete guide to creating exercises (includes discovery section)
- **`QUICK_START_NEW_EXERCISE.md`** - Fast-track reference for exercise creation
- **`explore_bcm.r`** - Example exploration script for BCM survey

---

## Questions?

1. **Read the discovery script**: `tools/discover_surveys.r` has inline comments
2. **Try with known survey**: `inspect_survey(5387)` (BCM Helpdesk)
3. **Check the full guide**: `docs/ADDING_NEW_EXERCISES.md`
4. **Look at examples**: `app/exercises/bcm_helpdesk_validation.r`

Happy discovering! 🔍

