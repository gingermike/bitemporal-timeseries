# Bitemporal Timeseries Project Context

## Project Overview
This is a high-performance Rust implementation of a bitemporal timeseries algorithm with Python bindings. The system processes financial/trading data with two time dimensions: effective time (when events occurred in the real world) and as-of time (when information was recorded in the system).

## Key Concepts
- **Bitemporal Data**: Records have both `effective_from/to` and `as_of_from/to` dates
- **Conflation**: Post-processing step that merges adjacent segments with identical value hashes to reduce database rows
- **Value Hash**: xxHash-based fingerprint of value columns to detect identical records for conflation
- **ID Groups**: Records are grouped by ID columns, algorithm processes each group independently enabling parallelization

## Project Structure
- `src/lib.rs` - Core bitemporal algorithm with parallel processing (870 lines)
- `tests/integration_tests.rs` - Rust integration tests (5 comprehensive test scenarios)
- `tests/test_bitemporal_manual.py` - Python test suite (legacy, comprehensive scenarios)
- `benches/bitemporal_benchmarks.rs` - Criterion benchmarks (5 benchmark suites)
- `Cargo.toml` - Dependencies and build configuration

## Key Commands
- **Test Rust**: `cargo test`
- **Test Python**: `uv run python -m pytest tests/test_bitemporal_manual.py -v`
- **Benchmark**: `cargo bench`
- **Build Release**: `cargo build --release`
- **Build Python Wheel**: `uv run maturin develop` or `uv run maturin build --release`
- **Python Environment**: Use `uv` for all Python commands to ensure proper virtual environment usage

## CRITICAL Development Workflow
**⚠️  IMPORTANT**: After making changes to Rust code (src/lib.rs), you MUST rebuild the Python bindings with `uv run maturin develop` before running Python tests. Changes to Rust code are NOT automatically reflected in Python tests until you rebuild the bindings. This has caused confusion in the past where fixes appeared not to work when they actually did.

## Performance Characteristics
- **Baseline**: ~1.5s for 500k records (serial processing)
- **Optimized**: ~885ms for 500k records (40% improvement with adaptive parallelization)
- **Parallelization**: Uses Rayon, adaptive thresholds (>50 ID groups OR >10k total records)
- **Conflation**: Reduces output rows by merging adjacent segments with same value hash

## Algorithm Details
- **Input**: Current state RecordBatch + Updates RecordBatch  
- **Output**: ChangeSet with `to_expire` and `to_insert` batches
- **Processing**: Groups by ID columns, processes each group independently
- **Conflation**: Post-processes results to merge adjacent same-value segments
- **Update Modes**: 
  - **Delta (default)**: Updates modify existing effective periods using timeline-based processing
  - **Full State**: Updates represent complete desired state; only expire/insert when values actually change (SHA256 hash comparison)
- **Temporal Precision**: Effective dates use Date32, as_of timestamps use TimestampMicrosecond for second-level precision

## Dependencies & Purpose
- `arrow` (53.4) - Columnar data processing, RecordBatch format
- `pyo3` (0.21) - Python bindings with extension-module feature
- `pyo3-arrow` (0.3) - Arrow integration for Python
- `chrono` (0.4) - Date/time handling
- `sha2` (0.10) - SHA256 hashing for value fingerprints (client-compatible hex digests)
- `rayon` (1.8) - Data parallelism
- `ordered-float` (4.2) - Hash-able floating point values
- `criterion` (0.5) - Professional benchmarking framework

## Test Coverage
- **Rust Tests**: 5 scenarios covering head slice, tail slice, unsorted data, overwrite, no-changes
- **Python Tests**: 7 scenarios including complex multi-update and multiple current state scenarios
- **Benchmarks**: Small/medium datasets, conflation effectiveness, scaling tests, parallel effectiveness

## Design Decisions
- **Adaptive Parallelization**: Serial for small datasets to avoid overhead, parallel for large ones
- **Post-processing Conflation**: Simpler than inline conflation, maintains algorithm correctness
- **Arrow Format**: Efficient columnar processing, zero-copy operations where possible
- **Separate Test Files**: Tests moved out of lib.rs for better organization

## Previous Challenges Solved
- **Test Compilation**: Fixed by making core functions public and updating crate-type to include "rlib"
- **StringArray Builder**: Updated to use StringBuilder::new() instead of deprecated constructor
- **Performance Regression**: Fixed by implementing adaptive parallelization thresholds
- **Project Organization**: Separated tests, benchmarks, and core algorithm into appropriate files

## Gitea CI/CD
- **Workflow**: `.gitea/workflows/build-wheels.yml` - Complete Linux wheel building and publishing
- **Architectures**: x86_64 and aarch64 cross-compilation support
- **Publishing**: Automatic publishing to Gitea package registry on version tags
- **Setup Guide**: `docs/gitea-publishing.md` - Complete configuration instructions
- **Triggers**: Version tags (`v1.0.0`) and manual workflow dispatch

## Notes
- Algorithm is deterministic and thread-safe when processing different ID groups
- Conflation maintains temporal correctness while optimizing storage
- Python integration allows seamless DataFrame → RecordBatch conversion with timestamp precision preservation
- Benchmarks show excellent scaling characteristics with parallelization
- **Timestamp Precision**: as_of columns preserve microsecond precision through the entire pipeline (Python → Arrow → Rust → Arrow → Python)
- **Python Conversion**: Automatic handling of pandas timestamp[ns] → Arrow timestamp[us] conversion for Rust compatibility
- **Infinity Handling**: Uses `2260-12-31 23:59:59` as infinity representation (avoids NaT, maintains datetime type, overflow-safe)

## Recent Updates
- **2025-07-27**: Implemented microsecond timestamp precision for as_of_from/as_of_to columns
- **2025-07-27**: Fixed Python wrapper to preserve exact timestamps from input through processing  
- **2025-07-27**: Updated all tests and benchmarks to use proper timestamp schemas
- **2025-07-27**: Fixed infinity handling to use `2260-12-31 23:59:59` instead of NaT for clear debugging
- **2025-07-27**: Added complete Gitea Actions workflow for Linux wheel building and publishing
- **2025-08-04**: Fixed non-overlapping update issue where current state records were incorrectly re-emitted when updates had same ID but no temporal overlap. Enhanced `process_id_timeline` to separate overlapping vs non-overlapping updates and process them appropriately.
- **2025-08-04**: Fixed `as_of_from` timestamp inheritance issue where re-emitted current state segments retained old timestamps instead of inheriting the update's `as_of_from` timestamp. Modified `emit_segment` to pass and use update timestamps for current state records affected by overlapping updates.
- **2025-08-29**: Changed hash implementation from Blake3 numeric values to SHA256 hex digest strings for client compatibility. Updated all schemas, tests, and benchmarks to use string-based hashes. Hash values now match Python's `hashlib.sha256().hexdigest()` format.
- **2025-08-29**: Fixed full_state mode logic that was incorrectly expiring ALL current records regardless of value changes. Updated implementation to only expire/insert records when values actually change, using SHA256 hash comparison for efficiency. This prevents unnecessary database operations for unchanged records in full_state mode.

## CRITICAL BUG: Full State Mode Missing Tombstone Records

### 🐛 **ISSUE IDENTIFIED (2025-08-29)**
Full state mode has a critical gap in deletion handling - it expires records that don't exist in the updates but fails to create tombstone records.

### **Problem Description:**
When using full_state mode, if a record exists in current state but NOT in updates, it should be "deleted":
1. ✅ **Current Behavior**: The old record is correctly expired 
2. ❌ **Missing Behavior**: A new tombstone record should be created with `effective_to = system_date`

### **Example Scenario:**
```
Current State: ID=2, effective_to=INFINITY (2260-12-31)
Updates: [ID=2 not present] 
Expected Result:
  - Expire: ID=2 with effective_to=INFINITY  
  - Insert: ID=2 with effective_to=TODAY (tombstone)
Actual Result:
  - Expire: ID=2 with effective_to=INFINITY
  - Insert: [nothing] ❌ MISSING TOMBSTONE
```

### **Impact:**
- Records appear to be deleted from the perspective of expiration, but there's no historical record showing when they became ineffective
- Violates bitemporal principles by not maintaining complete audit trail
- Makes it impossible to query "what was the state as of date X" for deleted records

### **Root Cause Location:**
File: `/src/lib.rs`, lines ~189-248 (full_state mode logic)
The algorithm correctly identifies records to expire but doesn't generate tombstone records for deletions.

### **Failing Test:**
- Test: `full_state_delete` in `tests/scenarios/basic.py`
- Run: `uv run python -m pytest tests/test_bitemporal.py::test_update_scenarios -k "full_state_delete" -v`

### **TODO for Tomorrow:**

#### 🔧 **Implementation Tasks:**
1. **Enhance full_state logic** to detect "deleted" records (exist in current, not in updates)
2. **Create tombstone records** for deleted records with:
   - Same ID values and value columns as expired record
   - `effective_to = system_date` (truncate the effective period)
   - `as_of_from = system_date` (new knowledge timestamp)
   - `as_of_to = INFINITY`
3. **Add tombstone records to insert batch** alongside regular updates

#### 🧪 **Testing Tasks:**
1. Fix failing test `full_state_delete`
2. Add additional tombstone scenarios:
   - Multiple deleted records
   - Mixed updates and deletions
   - Edge cases (records ending exactly on system_date)
3. Verify no regression in existing full_state functionality

#### 📚 **Documentation Tasks:**
1. Update README.md full_state section to mention tombstone behavior
2. Add tombstone example to documentation
3. Update Python docstrings to clarify deletion behavior

#### ⚡ **Performance Considerations:**
- Tombstone generation should integrate with existing batch processing
- Consider impact on conflation logic (tombstones may not conflate)
- Ensure parallel processing compatibility

### **Algorithm Sketch:**
```rust
// In full_state mode, after processing regular updates:
for current_record in &current_records {
    if !update_records.contains_matching_id(current_record.id_values) {
        // This is a deletion - create tombstone
        let tombstone = BitemporalRecord {
            id_values: current_record.id_values.clone(),
            value_hash: current_record.value_hash.clone(),
            effective_from: current_record.effective_from,
            effective_to: system_date,  // ← KEY: Truncate to system date
            as_of_from: system_date,
            as_of_to: INFINITY,
            original_index: None,
        };
        tombstone_records.push(tombstone);
    }
}
```

## Development Best Practices
- Ensure to keep README.md up to date with changes to the code base or approach
- **CRITICAL**: Fix the full_state tombstone issue before any production deployment

This file should be updated whenever a new piece of context or information is added / discovered in this project