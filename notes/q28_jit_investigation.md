# Q28 JIT Failure Investigation

## Summary
Q28 uses REGEXP_REPLACE with `cudf::transform()` JIT to extract URL domains.
The JIT works on cudf 26.04 nightly but fails on cudf 25.12 stable.

## Error
```
Jitify fatal error: Deserialization failed. This could be due to corrupted
cache files or a bug in Jitify.
```

This error occurs from `gpu_context.cpp:185` (inside GPUExecutePendingQueryResult).
The JIT IS being dispatched (pattern match succeeds), but NVRTC compilation fails.

## What "Deserialization failed" means in Jitify
In Jitify, the compilation pipeline is:
1. NVRTC compiles CUDA C++ string → PTX
2. PTX is "serialized" (packed for storage)
3. Serialized form is stored in cache
4. When executing, it is "deserialized" (loaded from cache/memory)

If NVRTC compilation fails, Jitify may report a deserialization failure rather than a clear compilation error.

## Troubleshooting Attempted
1. `JITIFY_CACHE_DIR=/tmp/fresh` — no effect
2. `CUDF_JIT_CACHE_PATH=/tmp/fresh` — no effect
3. `LIBCUDF_ENV_PREFIX=/home/ubuntu/ducndh/sirius-topk/.pixi/envs/default` — no effect
4. All 3 env vars combined — no effect

Conclusion: Not a cache corruption issue, but an NVRTC compilation failure.

## Root Cause Hypothesis
The UDF uses `cuda::std::optional<cudf::string_view>` which requires specific
CUDA/cudf headers to be available at JIT compile time. In cudf 25.12 **stable**:
- The header declares `null_aware::YES` support (in `transform.hpp`)
- But the NVRTC runtime might not have the correct header include paths
  configured for `cuda::std::optional`

In cudf 26.04 nightly:
- The JIT compilation works correctly
- Likely has updated jitify or better NVRTC include path setup

## Why Original Submission Worked
The original submission used the **nightly** Docker image:
```dockerfile
# docker/aarch64/nightly/Dockerfile
conda install -c rapidsai-nightly libcudf  # no version pin
```
In November 2025, this installed a 25.12 **nightly** build that had different
jitify internals than the stable 25.12.00 release (December 2025).

The nightly may have had:
- A newer version of Jitify with better header path handling
- Different NVRTC include path configuration
- A bug fix for sm_90a (GH200 Hopper architecture) JIT compilation

## Query Results with cudf 25.12 vs Original Submission
- All 42 non-Q28 queries: match within ≤5% of original ✓
- Q28: 1.334s vs original 0.267s (5x slower) — JIT fails, CPU fallback ✗

## Key Comparison: What Works vs What Doesn't
| Branch     | cudf    | Q28     | Q8     | Q9     | Q27    |
|------------|---------|---------|--------|--------|--------|
| original   | 25.12n  | 0.267s  | 0.084s | 0.089s | 0.165s |
| topk 25.12 | 25.12.0 | 1.334s  | 0.085s | 0.090s | 0.167s |
| gh200 (fixed)| 26.04n| 0.313s  | 0.132s | 0.137s | 0.191s |

## Options to Fix Q28 on cudf 25.12 Stable
1. **Use cudf 25.12 nightly** — nightly channel no longer has 25.12 builds
2. **Patch NVRTC headers** — configure cudf to find NVRTC include paths for optional
3. **Alternative JIT approach** — implement domain extraction without `cuda::std::optional`
4. **Upgrade to cudf 26.x** — requires RMM header path updates in 7 source files
