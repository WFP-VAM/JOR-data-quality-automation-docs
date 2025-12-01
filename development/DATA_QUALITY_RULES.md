# Data Quality Validation Rules

Comprehensive data quality checks implemented using the **affirm** package for data validation and reporting.

## BCM - Helpdesk Validation Exercise

Survey ID: **5387** | Records: **69**

### 1. ⏱️ Duration Check
**Rule**: Survey duration between 5 minutes and 24 hours  
**Column**: `_duration`  
**Logic**: Duration must be ≥ 300 seconds (5 min) and ≤ 86400 seconds (24 hours)  
**Why**: Flags surveys that are too quick (data quality issues) or unreasonably long (submission delays)

### 2. 📅 Same-Day Submission
**Rule**: Survey submitted on same day as data collection  
**Columns**: `date`, `_submission_time`  
**Logic**: Extract date from submission timestamp and compare with collection date  
**Why**: Identifies delays between data collection and submission

### 3. 🆔 Case ID Uniqueness
**Rule**: Case ID is unique within each collection date  
**Columns**: `Case_ID`, `date`  
**Logic**: Check for duplicate Case IDs on the same date  
**Why**: Prevents duplicate entries for same beneficiary on same day

### 4. 📞 Phone Number Uniqueness  
**Rule**: Phone number is unique within each collection date  
**Columns**: `Interviewee_Phone_Number`, `date`  
**Logic**: Check for duplicate phone numbers on the same date  
**Why**: Prevents same beneficiary being surveyed multiple times per day

### 5. ⚠️ Late Submission Flagging
**Rule**: Survey not submitted more than 24 hours after visit  
**Columns**: `date`, `_submission_time`  
**Logic**: Calculate time difference between submission and end of visit day  
**Why**: Flags potential data quality issues from delayed submissions

### 6. 📊 Missing Data Check
**Rule**: Rows do not have excessive missing values (< 50%)  
**Logic**: Count NAs per row, flag if more than half the columns are empty  
**Why**: Identifies incomplete survey submissions

---

## OSM - Helpdesk Monitoring Exercise

Survey ID: **5373** | Records: **91**

### 1. ⏱️ Duration Check
**Rule**: Survey duration between 5 minutes and 24 hours when recorded  
**Column**: `_duration`  
**Logic**: Duration must be ≥ 300 seconds (5 min) and ≤ 86400 seconds (24 hours)  
**Note**: OSM data has empty duration strings, so check is conditional  
**Why**: Same as BCM - quality control on survey timing

### 2. 📅 Same-Day Submission
**Rule**: Survey submitted on same day as data collection  
**Columns**: `date`, `_submission_time`  
**Logic**: Compare submission date with collection date  
**Why**: Monitor submission delays

### 3. ⚠️ Late Submission Flagging
**Rule**: Survey not submitted more than 24 hours after visit  
**Columns**: `date`, `_submission_time`  
**Logic**: Calculate hours between submission and end of visit day  
**Why**: Identify delayed data entry

### 4. 📍 Location Visit Frequency
**Rule**: Locations not visited more than allowed per month  
**Columns**: `location`, `date`  
**Logic**: 

- Host community locations: max 1 visit per month
- Camp locations: max 2 visits per month  
- Detects camps by keywords: "camp" or "مخيم"
**Why**: Ensure monitoring follows approved visit schedules

### 5. 🚫 Overcrowding Check
**Rule**: Overcrowding is not more than 10% of all entries  
**Column**: `overcrowding`  
**Logic**: Calculate proportion of "Yes" responses, must be ≤ 10%  
**Why**: Monitor site capacity issues

### 6. 📊 Missing Data Check
**Rule**: Rows do not have excessive missing values (< 50%)  
**Logic**: Count NAs per row across all columns  
**Why**: Ensure data completeness

### 🔮 Future Enhancement: Infrastructure Consistency
**Note**: Commented out pending historical data availability  
**Purpose**: Compare current infrastructure details (WASH, accessibility, etc.) with historical records to flag:

- Facilities that disappeared (data entry error or actual change)
- New facilities that appeared (improvement or error)
- Inconsistent reporting patterns

---

## Validation Reports

### Success Criteria

- ✅ **Success**: All validation rules passed  
⚠️ **Warning**: Some violations found but not critical  
❌ **Error**: Critical violations that require attention

### Report Output
The affirm package generates reports showing:

1. **Validation Summary**: Total successes/warnings/errors
2. **Detailed View**: Table with:
   - Exercise name
   - Validation description
   - Type (success/warning/error)
   - Number of violations

### Example Report Structure
```
Validation summary:
  Number of successful validations: 5
  Number of validations with warnings: 0
  Number of failed validations: 1

Advanced view:

|exercise_name      |description                           |type   | violations|
|:------------------|:-------------------------------------|:------|----------:|
|BCM Validation     |Duration between 5min and 24hr        |success|         NA|
|BCM Validation     |Same-day submission                   |error  |          3|
|BCM Validation     |Case ID unique per day                |success|         NA|
|BCM Validation     |Phone number unique per day           |success|         NA|
|BCM Validation     |Submitted within 24 hours             |success|         NA|
|BCM Validation     |Rows not excessively missing          |success|         NA|
```

---

## Implementation Notes

### Using affirm Package

The validation rules use the **affirm** package for data validation. Validation rules are implemented as factory functions in `app/R/validation_rules.r` that return functions using `affirm::affirm_true()`.

### Validation Rule Structure

Each validation rule factory returns a function that:
1. Takes `data`, `id`, and `data_name` parameters
2. Uses `affirm::affirm_true()` to validate conditions
3. Returns the data (possibly with temporary helper columns)

### Available Validation Rules

See `app/R/validation_rules.r` for the complete list of available validation rule factories:

- `duration_check_rule()` - Survey duration bounds
- `same_day_submission_rule()` - Same-day submission check
- `late_submission_rule()` - Submission delay check
- `missing_data_rule()` - Missing data check
- `case_id_uniqueness_rule()` - Case ID uniqueness
- `phone_uniqueness_rule()` - Phone number uniqueness
- `location_visit_frequency_rule()` - Location visit frequency
- `proportion_check_rule()` - Proportion validation (e.g., overcrowding)

---

## Testing Validation Rules

To test validation rules on sample data:

```r
# Load data
library(reticulate)
use_python(".venv/bin/python")
data_bridges_knots <- import("data_bridges_knots")
client <- data_bridges_knots$DataBridgesShapes("data_bridges_api_config.yaml")

bcm_data <- client$get_household_survey(survey_id = 5387L, access_type = "full")

# Test a validation using affirm
library(affirm)
library(dplyr)

# Initialize affirm session
affirm_init(replace = TRUE)

# Run validation
bcm_data |>
  affirm_true(
    `_duration` >= 300 & `_duration` <= 86400,
    label = "Duration check"
  )

# View results
affirm_report_gt()
```

---

## Future Enhancements

1. **Historical Comparison** (OSM)
   - Requires storing previous visit data
   - Compare infrastructure changes over time
   - Flag unexpected changes

2. **Trend Analysis**
   - Monitor validation failure rates over time
   - Identify recurring issues by location/enumerator

3. **Automated Alerts**
   - Email notifications for critical failures
   - Integration with RStudio Connect for scheduled checks

4. **Custom Thresholds**
   - Make validation thresholds configurable per exercise
   - Seasonal adjustments for visit frequency

---

## References

- [affirm Documentation](https://pcctc.github.io/affirm/)
- [Mastering Shiny](https://mastering-shiny.org/)

