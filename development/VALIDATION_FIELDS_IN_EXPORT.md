# Show Validation Calculation Fields in Export

## Overview

This refinement adds calculated validation fields to CSV exports, allowing users to see the specific values that triggered validation failures.

**Status**: ✅ **Implemented** (January 2026)  
**Approach**: Keep helper columns with `.` prefix  
**Coverage**: All 13 exercises updated  
**Latest enhancement**: Added threshold columns for outlier detection (Jan 23, 2026)

---

## Problem

The validation CSV exports showed which rules failed but didn't show the underlying calculated values used to determine the failure.

**What was missing**:
- The calculated submission datetime
- The calculated hours of delay
- The calculated visit end time
- The count of missing fields
- Threshold values for outlier detection

**Impact**: Users couldn't:
- Understand why a specific record failed
- Verify calculations were correct
- Prioritize follow-up based on severity
- Debug data quality issues effectively

---

## Solution Implemented

### What We Did

All validation helper columns (prefixed with `.`) are now included in CSV exports. These columns show the calculated values that determined each validation result.

**Key changes**:
1. Removed lines that stripped helper columns from exercise files
2. Removed `select()` statements from validation rules that were deleting columns
3. Added helper columns to `affirm.id_cols` configuration for CSV export
4. Added threshold columns (`.xxx_min_threshold`, `.xxx_max_threshold`) for outlier checks
5. Updated globalVariables declaration in validation_rules.r

### Why Keep the `.` Prefix

Helper columns retain their `.` prefix because:
- Clearly identifies calculated validation fields vs. original survey data
- Simple implementation - no renaming logic needed
- Users can easily filter/hide these columns in Excel if needed
- Maintains technical transparency

---

## Files Modified

### Core Files

**`app/R/validation_rules.r`**:
- Removed `select()` statements that stripped helper columns from:
  - `time_metrics_by_location_rule` (travel, waiting, service, jod outlier checks)
  - `id_phone_uniqueness_rule` (duplicate counts)
  - `conditional_section_rule` (trigger/section data flags)
- Added threshold calculations: `.xxx_min_threshold` and `.xxx_max_threshold`
- Updated globalVariables to declare all helper columns

### All 13 Exercise Files

Each exercise file was updated to:
1. Remove the `select(-starts_with("."))` line
2. Add helper columns to `affirm.id_cols` configuration
3. Update comments to reflect that helper columns are now kept

**Exercises updated**:
1. `bcm_helpdesk.r`
2. `bcm_helpdesk_validation.r`  
3. `bcm_post_office_validation.r`
4. `bcm_welcome_meals.r`
5. `osm_helpdesk.r`
6. `osm_post_office.r`
7. `osm_kitchen_checklist.r`
8. `osm_healthy_meals_hc.r`
9. `osm_healthy_meals_camps.r`
10. `osm_date_bars.r`
11. `osm_informal_market_price.r`
12. `gfa_sbcc_endline_women.r`
13. `fsom.r`

---

## Helper Columns Now Exported

| Helper Column | Validation Rule | What It Shows |
|--------------|----------------|---------------|
| `.submission_datetime` | late_submission | Parsed submission timestamp |
| `.visit_end` | late_submission | Visit end time (date + 23:59:59) |
| `.hours_delay` | late_submission | Hours between visit and submission |
| `.late_check` | late_submission | Boolean: passed/failed check |
| `.submission_date` | same_day_submission | Submission date (parsed) |
| `.same_day_check` | same_day_submission | Boolean: same day or not |
| `.row_na_count` | missing_data | Count of missing fields per row |
| `.missing_check` | missing_data | Boolean: passed/failed check |
| `.year_month` | location_visit_frequency | Month of visit (YYYY-MM) |
| `.location_clean` | location_visit_frequency | Cleaned location name |
| `.is_camp` | location_visit_frequency | Boolean: is camp location |
| `.visit_count` | location_visit_frequency | Number of visits to location |
| `.max_allowed` | location_visit_frequency | Max allowed visits |
| `.frequency_check` | location_visit_frequency | Boolean: passed/failed check |
| `.id_dup_count` | id_phone_uniqueness | Number of ID duplicates on same day |
| `.phone_dup_count` | id_phone_uniqueness | Number of phone duplicates on same day |
| `.travel_avg` | time_metrics_by_location | Average travel time for location |
| `.travel_min_threshold` | time_metrics_by_location | Min cutpoint (0.5× avg) |
| `.travel_max_threshold` | time_metrics_by_location | Max cutpoint (2× avg) |
| `.travel_outlier` | time_metrics_by_location | Boolean: is outlier |
| `.waiting_avg` | time_metrics_by_location | Average waiting time for location |
| `.waiting_min_threshold` | time_metrics_by_location | Min cutpoint (0.5× avg) |
| `.waiting_max_threshold` | time_metrics_by_location | Max cutpoint (2× avg) |
| `.waiting_outlier` | time_metrics_by_location | Boolean: is outlier |
| `.service_avg` | time_metrics_by_location | Average service time for location |
| `.service_min_threshold` | time_metrics_by_location | Min cutpoint (0.5× avg) |
| `.service_max_threshold` | time_metrics_by_location | Max cutpoint (2× avg) |
| `.service_outlier` | time_metrics_by_location | Boolean: is outlier |
| `.jod_avg` | time_metrics_by_location | Average transport cost for location |
| `.jod_min_threshold` | time_metrics_by_location | Min cutpoint (0.5× avg) |
| `.jod_max_threshold` | time_metrics_by_location | Max cutpoint (2× avg) |
| `.jod_outlier` | time_metrics_by_location | Boolean: is outlier |
| `.has_trigger` | conditional_section | Boolean: trigger condition met |
| `.section_has_data` | conditional_section | Boolean: section has data |
| `.gps_distance_meters` | gps_location_match | Distance to target (meters) |
| `gps_distance_to_target_m` | gps_location_match | Distance to target (without `.`) |
| `price_outliers_flagged` | price_outlier | Which prices were outliers |

**Note**: Boolean columns (`.late_check`, `.same_day_check`, etc.) are technically redundant with the `violated_rules` column but provide explicit pass/fail status per rule.

---

## Examples

### Example 1: Late Submission

**Before**:
```csv
date,_submission_time,violated_rules
2026-01-15,"2026-01-17T10:30:00Z","3: Survey submitted within 24 hours of visit date"
```

**After**:
```csv
date,_submission_time,.submission_datetime,.visit_end,.hours_delay,.late_check,violated_rules
2026-01-15,"2026-01-17T10:30:00Z","2026-01-17 10:30:00","2026-01-15 23:59:59",34.5,FALSE,"3: Survey submitted within 24 hours of visit date"
```

**What users see**:
- `.hours_delay = 34.5` - Survey was 34.5 hours late
- `.late_check = FALSE` - Failed the 24-hour check
- `.submission_datetime` - Exact submission time parsed
- `.visit_end` - When the visit day ended

### Example 2: Outlier Detection with Thresholds

**Before**:
```csv
location,jod_transport,violated_rules
"Helpdesk A",5.00,"9: Transport cost near location average"
```

**After**:
```csv
location,jod_transport,.jod_avg,.jod_min_threshold,.jod_max_threshold,.jod_outlier,violated_rules
"Helpdesk A",5.00,1.50,0.75,3.00,TRUE,"9: Transport cost near location average"
```

**What users see**:
- `.jod_avg = 1.50` - Average cost for this location
- `.jod_min_threshold = 0.75` - Lower cutpoint (0.5× avg)
- `.jod_max_threshold = 3.00` - Upper cutpoint (2× avg)
- `.jod_outlier = TRUE` - This value is an outlier
- Actual value = 5.00 - Exceeds the 3.00 threshold

Users can now see exactly why 5.00 JOD was flagged as an outlier.

---

## Enhancement: Outlier Thresholds (Jan 23, 2026)

### Issue Reported

**User feedback**: "when i'm looking at the checks against an average (e.g. when we have variables that look like .jod_outlier) it's not clear from the new downloaded csvs what was the cutpoint to become an outlier"

### Root Cause

The initial implementation kept the `.jod_outlier` boolean and `.jod_avg` average, but didn't show the actual threshold values (min/max cutpoints) used to determine outliers. Additionally, validation rules were removing these columns with `select()` statements.

### Fix Applied

1. Added threshold columns to all outlier checks:
   - `.travel_min_threshold` and `.travel_max_threshold`
   - `.waiting_min_threshold` and `.waiting_max_threshold`
   - `.service_min_threshold` and `.service_max_threshold`
   - `.jod_min_threshold` and `.jod_max_threshold`

2. Removed `select()` statements from `time_metrics_by_location_rule` that were stripping the average and outlier columns

3. Updated BCM Helpdesk exercise file to include new threshold columns in `affirm.id_cols`

4. Updated globalVariables declaration to include threshold columns

---

## Benefits

### 1. Transparency
- Users see exactly how validations were calculated
- Easy to verify calculations are correct
- Builds trust in the data quality system

### 2. Prioritization
- Can sort by severity: "34.5 hours late" vs "2 hours late"
- Understand magnitude: "15 missing fields" vs "2 missing fields"
- Focus on high-priority issues first

### 3. Debugging
- Quickly identify data entry errors
- Find patterns in validation failures
- Easier to train enumerators on common issues

### 4. Reporting
- Aggregate by delay ranges, missing counts, etc.
- Better insights into data quality trends
- Support for performance metrics and SLAs

### 5. Outlier Analysis
- Clear visibility into threshold calculations
- Understand why specific values were flagged
- Validate that thresholds are appropriate for each location

---

## Usage Notes

### In Excel

Users can:
- **Filter** by helper columns to see only severe violations (e.g., `.hours_delay > 48`)
- **Sort** by calculated values to prioritize follow-up
- **Hide columns** starting with `.` if they want a simplified view
- **Pivot** on helper columns for analysis (e.g., average delay by location)

### For Developers

When adding new validation rules:
1. Create helper columns with `.` prefix for intermediate calculations
2. Do NOT remove helper columns with `select()` statements
3. Add helper column names to `affirm.id_cols` in exercise files
4. Declare helper columns in globalVariables in validation_rules.r

See `guides/ADDING_NEW_EXERCISES.md` for detailed instructions.
