# Sirius ClickBench Benchmark Results

Tracking benchmark runs for the Sirius GPU SQL engine on GH200 (96 GB HBM3, aarch64, CUDA 12.9).

## Machine Info
- GPU: NVIDIA GH200 (96 GB HBM3, aarch64)
- CUDA: 12.9
- OS: Linux 6.8.0-1040-nvidia-64k

## Branches Under Test

| Branch | Path | cudf Version | Notes |
|--------|------|--------------|-------|
| `sirius-clickbench-gh200` | `/home/ubuntu/ducndh/sirius-clickbench-gh200` | 26.04 nightly | Original submission + Q28 bug fix applied |
| `sirius-dev-fused-kernel` | `/home/ubuntu/ducndh/sirius-dev-fused-kernel` | 26.04 nightly | Has fused kernel + top-K optimization |
| `sirius-topk` | `/home/ubuntu/ducndh/sirius-topk` | 25.12.00 stable | Top-K branch with cudf upgraded from 25.04 to 25.12 |

## Key Findings

### Q28 Bug (Critical)
- **Root cause**: `cudf::transform()` in cudf ≤ 25.04 does NOT support STRING output type → JIT fails → entire query falls back to DuckDB CPU (1.3s)
- **cudf 25.12** introduced: (1) STRING output support in transform(), (2) `null_aware::YES` parameter
- **Additional bug**: UDF output type was `cudf::string_view*` but `null_aware::YES` requires `cuda::std::optional<cudf::string_view>*` → silent CPU fallback even on cudf 26.x
- **Fix applied to gh200 branch**: Changed UDF output to `cuda::std::optional<cudf::string_view>*` → Q28: 1.31s → 0.31s

### cudf Version Compatibility
| Version | null_aware | STRING output | Old RMM headers | Notes |
|---------|-----------|---------------|-----------------|-------|
| 25.04   | ✗         | ✗             | ✓               | Original topk pixi.toml; JIT completely broken |
| 25.12   | ✓         | ✓             | ✓               | Minimum version for JIT to work; stable release |
| 26.02   | ✓         | ✓             | ✗               | RMM headers moved; topk code incompatible |
| 26.04   | ✓         | ✓             | ✗               | RMM headers moved; topk code incompatible |

### Top-K Optimization Assessment
Comparing `fused-kernel` (has `cudf::top_k_order()`) vs `gh200` (no top-K, same cudf 26.04):
→ **No measurable benefit** for ClickBench LIMIT queries. Bottleneck is always GROUP BY aggregation.

### cudf 26.04 Performance Regressions vs 25.12
Some queries are slower on cudf 26.04 nightly:
- Q8: 0.085s → 0.133s (integer COUNT DISTINCT)
- Q9: 0.089s → 0.137s
- Q27: 0.097s → 0.192s
- Q32: 0.058s → 0.119s

cudf 26.04 improvements:
- Q20: 0.095s → 0.041s
- Q23: 0.119s → 0.066s
- Q28: 1.31s → 0.31s (with Q28 bug fix)

## Directory Structure
- `environments/` - pixi.toml snapshots for each build
- `results/` - Benchmark timing CSVs per run
- `notes/` - Analysis notes
- `scripts/` - Helper scripts for running benchmarks

## Original Submission
- Branch: `clickbench_with_topk` on https://github.com/sirius-db/sirius
- Used nightly Dockerfile (rapidsai-nightly::libcudf, unversioned) circa Nov 2025
- Nov 2025 nightly = cudf 25.12 development cycle → effectively cudf 25.12
- Q28 time in original submission: 0.267s
