# cudf Version Analysis for Sirius ClickBench

## Problem
The `sirius-topk` branch originally had `libcudf = "25.04.*"` in pixi.toml, causing Q28 to fall
back to CPU (1.31s) instead of the expected 0.267s. This document traces the root cause and solution.

## Q28 JIT Requirements

Q28 uses `cudf::transform()` with a CUDA JIT UDF to extract URL domains.

The JIT requires two features NOT available in cudf 25.04:
1. **STRING output type** in `cudf::transform()` — 25.04 throws "Transforms only support fixed-width types"
2. **`cudf::null_aware::YES`** parameter — added as the 6th argument to `transform()`

## cudf Version Feature Matrix

| Version | null_aware in transform.hpp | STRING output | rmm/mr/device/device_memory_resource.hpp | Notes |
|---------|---------------------------|---------------|------------------------------------------|-------|
| 25.04   | NO                        | NO            | YES (old path)                           | Original broken pixi.toml |
| 25.12   | YES                       | YES           | YES (both paths)                         | **Minimum working version; same era as original submission** |
| 26.02   | YES                       | YES           | NO (only new path)                       | RMM headers moved; topk source incompatible |
| 26.04   | YES                       | YES           | NO (only new path)                       | RMM headers moved; topk source incompatible |

## RMM Header Path Change (between 25.12 and 26.02)

Old path (≤25.12): `rmm/mr/device/device_memory_resource.hpp`
New path (≥26.02): `rmm/mr/device_memory_resource.hpp` (no `device/` subdirectory)

The `sirius-topk` source code (and original topk branch) includes the OLD path:
- `src/include/memory/memory_reservation.hpp`
- `src/include/memory/per_stream_tracking_resource_adaptor.hpp`
- `src/include/memory/memory_space.hpp`
- `src/include/memory/fixed_size_host_memory_resource.hpp`
- `src/include/memory/null_device_memory_resource.hpp`
- `src/include/data/common.hpp`
- `src/include/cudf/cudf_utils.hpp`

The `sirius-clickbench-gh200` source uses the NEW path, so it compiles fine with cudf 26.x.

## UDF Type Signature Bug

Even on cudf 26.x, there was an additional bug in all branches (gh200, topk, original):

```cpp
// WRONG (cudf::null_aware::YES requires optional output):
__device__ void extract_domain(cudf::string_view* out, cuda::std::optional<cudf::string_view> const url_opt)

// CORRECT:
__device__ void extract_domain(cuda::std::optional<cudf::string_view>* out, cuda::std::optional<cudf::string_view> const url_opt)
```

This caused silent JIT failure → CPU fallback even on cudf 26.04 in the gh200 branch.
**Fix applied to gh200 on 2026-02-27**: Q28 went from 1.31s → 0.31s.

## transform() API Signature Changes

### cudf 25.12 (6 args before stream):
```cpp
std::unique_ptr<column> transform(
  std::vector<column_view> const& inputs,
  std::string const& transform_udf,
  data_type output_type,
  bool is_ptx,
  std::optional<void*> user_data    = std::nullopt,
  null_aware is_null_aware          = null_aware::NO,
  rmm::cuda_stream_view stream      = ...);
```

### cudf 26.02+ (7 args before stream — added output_nullability):
```cpp
std::unique_ptr<column> transform(
  ...
  null_aware is_null_aware          = null_aware::NO,
  output_nullability null_policy    = output_nullability::PRESERVE,  // NEW in 26.02
  rmm::cuda_stream_view stream      = ...);
```

The topk code calls: `cudf::transform({input}, udf, STRING, false, nullopt, null_aware::YES)`
This 6-arg call compiles correctly on both 25.12 and 26.02+ (null_policy defaults).

## Original Submission Environment

- Branch: `clickbench_with_topk` on github.com/sirius-db/sirius
- Dockerfile: `docker/aarch64/nightly/Dockerfile` — used `rapidsai-nightly::libcudf` (unversioned)
- Code committed: November 7, 2025
- November 2025 nightly = cudf 25.12 development cycle
- Stable cudf 25.12.00 released December 2025
- **Conclusion**: Original submission used cudf 25.12 nightly ≈ cudf 25.12.00 stable

## Fix Applied to sirius-topk

Changed `pixi.toml`:
```toml
# Before:
channels = ["rapidsai-nightly", "rapidsai", "conda-forge", "nvidia"]
libcudf = "26.04.*"   # or 25.04.* originally
librmm = "26.04.*"

# After:
channels = ["rapidsai", "conda-forge", "nvidia"]
libcudf = "25.12.*"
librmm = "25.12.*"
```

Also fixed UDF output type in `src/expression_executor/regex/regex_playground.cpp`
to use `cuda::std::optional<cudf::string_view>*` for the output parameter.
