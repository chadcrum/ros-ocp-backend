# CSV Schema Mismatch Fix - GPU Metrics Support

**Date**: 2025-11-26
**Issue**: Processor rejecting ALL CSV files from Cost Management Operator
**Root Cause**: CSV schema version mismatch - operator generating 49 columns, processor expecting 37
**Status**: FIXED - Applied upstream fix from RedHatInsights/ros-ocp-backend

---

## Problem Description

### Symptoms
- Processor consuming Kafka messages successfully
- ALL CSV files failing validation with error: "CSV file does not have all the required columns"
- NO experiments being created in Kruize database (beyond test data)
- Both namespace AND container CSV files failing (unexpected - only namespace failure was expected)

### Investigation Timeline

**Initial Observation**: Test CSV files (37 columns) from `test-ocp-dataflow-jwt.sh` worked perfectly, but real operator CSV files (49 columns) all failed validation.

**CSV Column Comparison**:
- **Test CSV (works)**: 37 columns - matches processor's CSVColumnMapping
- **Operator CSV (fails)**: 49 columns - includes 12 new GPU/accelerator fields

### New Columns in Operator CSV

The Cost Management Operator added GPU/accelerator support with 12 additional columns:

1. `cpu_throttle_container_min` - Additional CPU metric
2-12. **GPU/Accelerator fields**:
   - `accelerator_model_name`
   - `accelerator_profile_name`
   - `accelerator_core_usage_percentage_min`
   - `accelerator_core_usage_percentage_max`
   - `accelerator_core_usage_percentage_avg`
   - `accelerator_memory_copy_percentage_min`
   - `accelerator_memory_copy_percentage_max`
   - `accelerator_memory_copy_percentage_avg`
   - `accelerator_frame_buffer_usage_min`
   - `accelerator_frame_buffer_usage_max`
   - `accelerator_frame_buffer_usage_avg`

### Root Cause

**File**: `/home/ccrum/git/insights-on-prem/ros-ocp-backend/internal/utils/aggregator.go`
**Function**: `check_if_all_required_columns_in_CSV()`

**Problematic Code** (local/on-prem version):
```go
func check_if_all_required_columns_in_CSV(df dataframe.DataFrame) error {
    all_required_columns := make([]string, 0, len(types.CSVColumnMapping))
    for k := range types.CSVColumnMapping {
        all_required_columns = append(all_required_columns, k)
    }

    columns_in_csv := df.Names()
    if !elementsMatch(all_required_columns, columns_in_csv) {  // EXACT MATCH
        return fmt.Errorf("CSV file does not have all the required columns")
    }
    return nil
}
```

**Problem**: Uses `elementsMatch()` which requires **EXACT column match** - CSV must have exactly 37 columns, no more, no less. Any extra columns (like GPU fields) cause validation failure.

---

## Solution: Upstream Fix

### Source

**Repository**: [https://github.com/RedHatInsights/ros-ocp-backend](https://github.com/RedHatInsights/ros-ocp-backend)
**Commit**: `b8f1bfcea1bdd9febb29b795da41fab988f132f7`
**Date**: October 15, 2025
**Author**: saltgen (Sagnik Dutta)
**Title**: "RHINENG-21378 Ignore additional columns in CSV"
**Link**: https://github.com/RedHatInsights/ros-ocp-backend/commit/b8f1bfcea1bdd9febb29b795da41fab988f132f7

### Fix Applied

**Changed Function**:
```go
func check_if_all_required_columns_in_CSV(df dataframe.DataFrame) error {
    all_required_columns := make([]string, 0, len(types.CSVColumnMapping))
    for k := range types.CSVColumnMapping {
        all_required_columns = append(all_required_columns, k)
    }

    columns_in_csv := df.Names()
    if hasMissingColumns(all_required_columns, columns_in_csv) {  // CONTAINS CHECK
        return fmt.Errorf("CSV file does not have all the required columns")
    }
    return nil
}
```

**New Helper Function**:
```go
func hasMissingColumns(requiredColumns []string, csvColumns []string) bool {
    slices.Sort(requiredColumns)
    for _, reqCol := range requiredColumns {
        if !slices.Contains(csvColumns, reqCol) {
            log.Warnf("missing columns in CSV: %v", reqCol)
            return true
        }
    }
    return false
}
```

**Additional Change**: Added `"slices"` import to support `slices.Sort()` and `slices.Contains()`.

### How It Works

**Before (Exact Match)**:
- Validation checks if CSV columns exactly match required columns (37 columns)
- Extra columns → validation fails ❌
- Missing columns → validation fails ❌

**After (Contains Check)**:
- Validation checks if CSV **contains all** required columns
- Extra columns → **IGNORED**, validation succeeds ✅
- Missing columns → validation fails (expected) ❌

This allows backward compatibility: processor works with both old 37-column CSVs AND new 49-column CSVs with GPU metrics.

---

## Changes Made to Local Repository

### File Modifications

1. **`/home/ccrum/git/insights-on-prem/ros-ocp-backend/internal/utils/aggregator.go`**
   - Added `"slices"` import
   - Replaced `elementsMatch()` call with `hasMissingColumns()` in `check_if_all_required_columns_in_CSV()`
   - Added new `hasMissingColumns()` helper function

2. **`/home/ccrum/git/insights-on-prem/ros-ocp-backend/internal/services/report_processor.go`**
   - Changed error handling from `return` to `continue` (allows processing next file if one fails)
   - Lines 72, 81: Ensures both CSV files in upload are processed even if namespace CSV fails

### Build and Deploy

```bash
# Build new container image
cd /home/ccrum/git/insights-on-prem/ros-ocp-backend
make build-image IMAGE_TAG=csv-fix-20251126-162407

# Tag as latest
podman tag localhost/ros-ocp-backend:csv-fix-20251126-162407 localhost/ros-ocp-backend:latest
```

**Deployment**: Image needs to be pushed to cluster-accessible registry and processor deployment updated.

---

## Verification Steps

### 1. Confirm CSV Files are Being Processed

```bash
# Monitor processor logs for successful CSV processing
oc logs -n cost-onprem -l app.kubernetes.io/component=processor -c rosocp-processor --tail=50 | grep -v "does not have all the required columns"

# Should see workload processing messages instead of column errors
```

### 2. Check Experiments Created

```bash
# Query Kruize for experiments
cd /home/ccrum/git/insights-on-prem/cost-onprem-chart/scripts
./query-kruize.sh --experiments

# Should show experiments for actual cluster workloads, not just test data
```

### 3. Verify Recommendations Generated

```bash
# Wait 15+ minutes after experiments created, then check
./query-kruize.sh --recommendations

# Should show recommendations for cluster workloads
```

---

## Related Issues

### On-Prem Deployment Issues

- This fix resolves the primary blocker preventing real operator data from being processed
- Prior testing with `test-ocp-dataflow-jwt.sh` was successful because test data used old 37-column schema
- User Workload Monitoring enablement (separate issue) is also required for operator to collect metrics

### Upstream Compatibility

- **insights-on-prem/ros-ocp-backend**: Needs this fix applied
- **RedHatInsights/ros-ocp-backend**: Already has fix (upstream source)
- **Cost Management Operator**: Generates new 49-column schema (working as designed)

---

## Future Considerations

### Schema Versioning

- Processor now forwards-compatible with new CSV columns (ignores unknown columns)
- If operator adds more fields in future, processor will continue working
- Only missing required columns will cause validation failure

### GPU Metrics Support

- Current processor ignores GPU metrics (doesn't process them)
- If GPU metrics needed, update `types.CSVColumnMapping` to include accelerator fields
- Kruize may need updates to handle GPU recommendations

---

## Testing Notes

**Test Data Difference**:
- `test-ocp-dataflow-jwt.sh` generates 37-column CSV (old schema)
- Real operator generates 49-column CSV (new schema with GPU)
- This discrepancy masked the issue during initial testing

**Recommendation**: Update test script to generate 49-column CSV to match real operator behavior.

---

## References

- Upstream Commit: https://github.com/RedHatInsights/ros-ocp-backend/commit/b8f1bfcea1bdd9febb29b795da41fab988f132f7
- Jira Issue: RHINENG-21378
- Troubleshooting Doc: `/home/ccrum/git/insights-on-prem/docs/troubleshooting-ros-deployment.md`
- Test Script: `/home/ccrum/git/insights-on-prem/cost-onprem-chart/scripts/test-ocp-dataflow-jwt.sh`
