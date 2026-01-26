# BCM Helpdesk Issue Resolution Fix

## Overview

This document describes the fix for a bug in the BCM Helpdesk dashboard where the "Issue Resolved" status showed all records as "not resolved" (0%), even though the data contained valid responses such as "Yes," "No," "Pending Response," and "I do not know."

## Problem Statement

**Current behavior**: The BCM Helpdesk dashboard's "Issue Resolution" metric card displayed 0% (or an incorrect percentage) for all records.

**Actual data**: The `issues_solv_yn` column contains text responses:
- "Yes" - Issue was resolved
- "No" - Issue was not resolved  
- "Pending Response" - Resolution pending
- "I do not know" - Unknown resolution status

**Expected behavior**: The dashboard should display the actual breakdown of responses with counts and percentages for each category, rather than trying to interpret which responses are "positive".

## Root Cause

The dashboard was using `satisfaction_metric_card()` which tries to calculate a single percentage by identifying "positive" responses. This function was designed for binary satisfaction questions (satisfied/not satisfied) but was inappropriate for the issue resolution question which has multiple distinct response types that all provide valuable information.

## Solution

### Implementation Date
January 23, 2026

### Changes Made

#### 1. Replaced Metric Card with Breakdown Card

Changed from a single percentage metric to a full response breakdown:

**File**: `app/R/visualization_helpers.r` (within `bcm_helpdesk_dashboard()` function)

**Before**:
```r
# Issue resolution
div(
  style = "margin-bottom: 20px;",
  h4("Issue Resolution", style = "color: #555; font-size: 16px; margin-bottom: 10px;"),
  satisfaction_metric_card(data, "issues_solv_yn",
                          get_label("issues_solv_yn", "Issues Resolved"))
),
```

**After**:
```r
# Issue resolution
div(
  style = "margin-bottom: 20px;",
  h4("Issue Resolution", style = "color: #555; font-size: 16px; margin-bottom: 10px;"),
  breakdown_card(data, "issues_solv_yn",
                get_label("issues_solv_yn", "Issue Resolution Status"),
                metadata = metadata,
                choice_list_name = "issues_solv_yn")
),
```

**Key improvements**:
- Uses `breakdown_card()` instead of `satisfaction_metric_card()`
- Shows all response options with individual counts and percentages
- Displays horizontal bar charts for visual comparison
- No interpretation needed about what constitutes "success"
- Clearer communication of actual response distribution

#### 2. Enhanced `satisfaction_metric_card()` Function (for future use)

As a byproduct of investigating this issue, the `satisfaction_metric_card()` function was enhanced to support text-based responses:

**Changes**:
- Added `positive_values` parameter with default value `"1"` for backward compatibility
- Changed from exact match (`==`) to `%in%` operator to support vectors of positive values
- Enhanced validation to exclude empty strings

**Note**: While this enhancement was made, the BCM Helpdesk dashboard ultimately uses `breakdown_card()` for clearer reporting.

## Technical Details

### How the Fix Works

The `breakdown_card()` function provides comprehensive response visualization:

1. **Counts all responses**: Automatically tabulates all unique values in the column
2. **Calculates percentages**: Shows percentage for each response category
3. **Visual representation**: Displays horizontal bars for easy comparison
4. **Sorted display**: Responses ordered by frequency (most common first)
5. **Metadata support**: Can apply choice labels from survey metadata if available

**Example output** (for issue resolution):
```
Issue Resolution Status
Based on 100 responses

Yes                    63 (63%)  [===============]
No                     25 (25%)  [=====]
Pending Response       10 (10%)  [==]
I do not know           2 (2%)   [=]
```

### Affected Dashboards

The `satisfaction_metric_card()` function is used in multiple dashboards:

**Dashboards still using default (`"1"`):**
- `bcm_monitoring_dashboard()` - BCM Helpdesk Validation
  - `Q4_1_were_trated_respectfully` - "Treated Respectfully"
  - `Q4_2_satisfied_w_validation_process` - "Satisfied with Process"
  - `Q4_4_know_validation_result` - "Know Validation Result"
  - `Q4_3_comfortable_using_validation_equip` - "Comfortable Using Equipment"

- `bcm_post_office_validation_dashboard()` - BCM Post Office Validation
  - `Q4_1_were_trated_respectfully` - "Treated Respectfully"
  - `Q4_2_satisfied_w_validation_process` - "Satisfied with Process"
  - `Q3_6_comfortable_using_validation_equipment` - "Comfortable Using Equipment"

**Dashboards now using custom positive values:**
- `bcm_helpdesk_dashboard()` - BCM Helpdesk
  - `issues_solv_yn` - "Issues Resolved" (uses `positive_values = "Yes"`)

### Color Coding

The metric card uses color-coded thresholds:
- **Green** (#28a745): ≥ 80% - Good performance
- **Yellow** (#ffc107): 60-79% - Moderate performance  
- **Red** (#dc3545): < 60% - Needs attention

## Testing

### Test Scenarios

When testing the BCM Helpdesk dashboard:

1. **All response categories shown**:
   - Should display separate rows for "Yes", "No", "Pending Response", "I do not know"
   - Each row shows count and percentage

2. **Correct percentages**:
   - Example: 63 "Yes", 25 "No", 10 "Pending", 2 "Don't know" out of 100 total
   - Should display: Yes (63%), No (25%), Pending Response (10%), I do not know (2%)

3. **Visual bars**:
   - Horizontal bars should be proportional to percentages
   - Bars should use gradient blue color

4. **Total count**:
   - "Based on N responses" text should match total number of valid responses

5. **Backward compatibility**:
   - BCM validation dashboards should still work correctly
   - They use `satisfaction_metric_card()` which remains unchanged in functionality

### Manual Verification Steps

1. Load BCM - Helpdesk exercise in the dashboard
2. Locate the "Issue Resolution" section
3. Verify it shows a breakdown card (not a percentage metric)
4. Check that all response categories are displayed with counts
5. Verify percentages add up to 100%
6. Confirm horizontal bars are visible and proportional
7. Load BCM validation exercises to ensure they still use satisfaction metrics correctly

## Files Modified

### Application Code

**Core functionality**:
- `app/R/visualization_helpers.r`
  - Updated `satisfaction_metric_card()` function (lines ~275-313)
  - Updated `bcm_helpdesk_dashboard()` function (lines ~985-991)

**No exercise files were modified** - this was purely a visualization fix.

### Documentation

- `development/BCM_HELPDESK_ISSUE_RESOLUTION_FIX.md` - This document
- `development/README.md` - To be updated with reference to this document

## Benefits

### 1. Clear Communication
- Shows actual response distribution without interpretation
- Stakeholders see what respondents actually answered
- No ambiguity about what percentages mean

### 2. Complete Information
- All response categories visible, not just "positive" vs "negative"
- Can identify trends like increasing "Pending Response" rates
- Better understanding of issue resolution patterns

### 3. Visual Clarity
- Horizontal bar charts make comparisons easy
- Sorted by frequency for quick insights
- Clean, professional presentation

### 4. Appropriate Tool Selection
- Uses `breakdown_card()` for categorical questions
- Reserves `satisfaction_metric_card()` for true binary satisfaction questions
- Right visualization for the right data type

## Future Considerations

### When to Use Each Card Type

**Use `breakdown_card()` when**:
- Question has 3+ distinct response categories
- All responses provide valuable information
- No clear "positive" or "negative" interpretation
- Need to see distribution across all options
- Examples: Issue status, reason for visit, service type

**Use `satisfaction_metric_card()` when**:
- Binary or clearly positive/negative question
- Single percentage metric is meaningful
- Color-coded thresholds add value (green/yellow/red)
- Examples: "Were you treated respectfully?" (Yes/No), satisfaction ratings (1-5)

### Potential Enhancements

1. **Conditional formatting in breakdown_card()**: Add color coding based on response type:
   ```r
   # Green for "Yes", Red for "No", Yellow for "Pending", etc.
   ```

2. **Comparison over time**: Show how response distribution changes across time periods

3. **Export functionality**: Allow downloading response breakdowns as CSV for reporting

## Related Issues

This refinement addresses:
- Incorrect issue resolution reporting in BCM Helpdesk dashboard
- Hardcoded assumptions about response formats
- Need for flexible metric calculation across different survey types

## See Also

- `development/AUTOMATIC_DATE_FIELDS.md` - Previous refinement for date handling
- `app/R/visualization_helpers.r` - Implementation of dashboard functions
- `app/exercises/bcm_helpdesk.r` - BCM Helpdesk exercise definition
