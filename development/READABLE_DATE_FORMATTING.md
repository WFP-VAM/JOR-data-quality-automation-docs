# Readable Date Formatting in App and CSV Exports

## Overview

Datetime columns in both the app display and CSV exports now show in human-readable format instead of ISO 8601.

**Status**: ✅ **Implemented** (January 2026)  
**Approach**: Format dates before validations + make validation rules robust  
**Coverage**: All 13 exercises  

---

## Problem

Original datetime columns showed ISO 8601 format with full timezone information:
```
_submission_time: 2025-08-07T10:24:22.849301+02:00  ❌ Hard to read
start: 2025-10-05T14:04:16.645+02:00  ❌ Hard to read
end: 2025-10-05T14:11:06.302+02:00  ❌ Hard to read
```

This affected both:
- **App display** - The validation table shown in the Shiny app
- **CSV exports** - Downloaded validation reports

---

## Solution Implemented

**Step 1**: Format datetime columns before running validations
```r
data <- format_datetime_for_export(data)
```

**Step 2**: Make validation rules robust to handle both formats
- Updated `late_submission_rule()` to parse both `2025-08-07T10:24:22` and `2025-08-07 10:24:22`
- Updated `same_day_submission_rule()` to extract date from either format

**Result**: Datetime columns now show in readable format everywhere:

| Column | Before | After |
|--------|--------|-------|
| `_submission_time` | `2025-08-07T10:24:22.849301+02:00` | `2025-08-07 10:24:22` |
| `start` | `2025-10-05T14:04:16.645+02:00` | `2025-10-05 14:04:16` |
| `end` | `2025-10-05T14:11:06.302+02:00` | `2025-10-05 14:11:06` |

And helper columns remain formatted:
- `.submission_datetime` → `2025-08-07 10:24:22`
- `.visit_end` → `2025-08-07 23:59:59`

---

## Implementation Details

### Challenge: Affirm Stores Data Internally
The `affirm` package stores data when each validation rule runs. This stored data is what gets exported to CSV and displayed in the app. Therefore:
- ✅ Formatting **before** validations affects both app display and CSV
- ❌ Formatting **after** validations has no effect

### Challenge: Validation Rules Expected Specific Format
Original validation rules parsed datetimes with:
```r
as.POSIXct(submission_time, format = "%Y-%m-%dT%H:%M:%S")  # Expects T separator
```

This breaks if dates are formatted to space-separated format before validations.

### Solution: Robust Parsing in Validation Rules

Updated validation rules to handle BOTH formats:
```r
.submission_datetime = {
  val <- as.character(_submission_time)
  # Try ISO 8601 format first (with T)
  result <- as.POSIXct(val, format = "%Y-%m-%dT%H:%M:%S")
  # If that fails, try space-separated format
  if (all(is.na(result))) {
    result <- as.POSIXct(val, format = "%Y-%m-%d %H:%M:%S")
  }
  result
}
```

**This allows**:
- Format dates before validations (for readable display)
- Validation calculations still work correctly
- No risk of NA values in `.hours_delay` or other calculations

---

## Available Formatted Columns

### Original Columns (Now Formatted)
- `_submission_time` → `2025-08-07 10:24:22`
- `start` → `2025-08-07 10:24:22`
- `end` → `2025-08-07 10:24:22`
- `date` → `2025-08-07`

### Helper Columns (Also Formatted)
- `.submission_datetime` → `2025-08-07 10:24:22` (from late_submission_rule)
- `.visit_end` → `2025-08-07 23:59:59` (from late_submission_rule)
- `.submission_date` → `2025-08-07` (from same_day_submission_rule)

---

## Usage

### In the App
All datetime columns display in readable format:
```
_submission_time: 2025-08-07 10:24:22  ✅ Easy to read
start: 2025-10-05 14:04:16  ✅ Easy to read
end: 2025-10-05 14:11:06  ✅ Easy to read
```

### In Excel (CSV Exports)
Same readable format:
```csv
Case_ID,_submission_time,start,end,.submission_datetime,.hours_delay
BCM001,2025-08-07 10:24:22,2025-08-07 09:00:00,2025-08-07 10:00:00,2025-08-07 10:24:22,10.4
```

All columns are:
- ✅ Easy to read
- ✅ Sortable/filterable in Excel
- ✅ Consistent format throughout

---

## Examples

### Example 1: App Display

**Validation table shows**:
```
Case_ID: BCM001
_submission_time: 2025-08-07 10:24:22  ✅ Readable
.hours_delay: 10.4  ✅ Calculated correctly
violated_rules: "3: Survey submitted within 24 hours"
```

### Example 2: CSV Export

**Before**:
```csv
Case_ID,_submission_time,.hours_delay
BCM001,2025-08-07T10:24:22.849301+02:00,10.4
```

**After**:
```csv
Case_ID,_submission_time,.hours_delay
BCM001,2025-08-07 10:24:22,10.4
```

All columns are readable, calculations work perfectly!

---

## Benefits

### 1. Readability
- ✅ Easy to read at a glance in app and CSV
- ✅ No timezone clutter or microseconds
- ✅ Consistent `YYYY-MM-DD HH:MM:SS` format

### 2. Safety
- ✅ Calculations verified to work correctly
- ✅ Robust parsing handles both date formats
- ✅ No NA values in `.hours_delay` or other metrics

### 3. Simplicity
- ✅ One-line addition to each exercise
- ✅ Minimal changes to validation rules
- ✅ Works for all 13 exercises

### 4. Excel-Friendly
- ✅ Excel recognizes formatted dates
- ✅ Can sort and filter easily
- ✅ Copy/paste works smoothly

---

## For Developers

When creating new validation rules that work with datetime data:

1. **Parse datetimes into helper columns** with `.` prefix:
   ```r
   .submission_datetime = as.POSIXct(.data[[submission_column]], 
                                     format = "%Y-%m-%dT%H:%M:%S")
   ```

2. **Format helper columns as POSIXct**, not character:
   - ✅ `as.POSIXct()` returns a datetime object
   - ✅ When exported to CSV, R automatically formats it as `YYYY-MM-DD HH:MM:SS`
   - ❌ Don't use `format()` to convert to character - it breaks calculations

3. **Add helper columns to `affirm.id_cols`**:
   ```r
   options('affirm.id_cols' = c(
     "date", "_submission_time",  # Original columns
     ".submission_datetime", ".visit_end"  # Helper columns (already formatted)
   ))
   ```

4. **Result**: Helper columns appear in CSV exports in readable format automatically!

---

## Technical Details

### Why Helper Columns Are Already Formatted

When you create a helper column as POSIXct:
```r
.submission_datetime = as.POSIXct("2025-08-07T10:24:22.849301+02:00", 
                                  format = "%Y-%m-%dT%H:%M:%S")
```

R stores it internally as a number (seconds since 1970-01-01), but when exported to CSV:
1. R's default CSV writing uses `format()` on datetime objects
2. Default POSIXct format is `%Y-%m-%d %H:%M:%S`
3. Result in CSV: `2025-08-07 10:24:22` ✅

This happens automatically - **no explicit formatting needed!**

### Why We Don't Format Original Columns

Original columns need to stay in ISO 8601 format because:
1. **Validation rules expect it** - They parse using `format = "%Y-%m-%dT%H:%M:%S"`
2. **Calculations depend on it** - Changing format breaks `difftime()` calculations
3. **Helper columns already provide formatted versions** - No need to duplicate effort

---

## Files Modified

### Core Files
1. **`app/R/validation_rules.r`**:
   - Updated `late_submission_rule()` with robust datetime parsing
   - Updated `same_day_submission_rule()` to handle both formats

2. **`app/R/validation_helpers.r`**:
   - `format_datetime_for_export()` function (converts ISO 8601 to readable format)

### All 13 Exercise Files
Each exercise now calls `format_datetime_for_export(data)` before running validations:

1. ✅ `bcm_helpdesk.r`
2. ✅ `bcm_helpdesk_validation.r`
3. ✅ `bcm_post_office_validation.r`
4. ✅ `bcm_welcome_meals.r`
5. ✅ `osm_helpdesk.r`
6. ✅ `osm_post_office.r`
7. ✅ `osm_kitchen_checklist.r`
8. ✅ `osm_healthy_meals_hc.r`
9. ✅ `osm_healthy_meals_camps.r`
10. ✅ `osm_date_bars.r`
11. ✅ `osm_informal_market_price.r`
12. ✅ `gfa_sbcc_endline_women.r`
13. ✅ `fsom.r`

---

## Summary

✅ **Datetime columns display in readable format** in both app and CSV  
✅ **All calculations work correctly** - verified with test data  
✅ **Implemented across all 13 exercises**  
✅ **Format**: `YYYY-MM-DD HH:MM:SS` for datetimes, `YYYY-MM-DD` for dates  
✅ **Safe**: Robust parsing prevents calculation failures
