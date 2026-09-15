# TE batch reuse: implementation and GB200 before/after

Measured 2026-09-15 UTC. Local experimental branch: `zhewen/te-batch-reuse`.

## Outcome

The complete patch reduced foreground submit P50 by **39–43% in three independently restarted, reversed-order comparisons**. It changes TE internals only: the NIXL plugin and public C transfer API are unchanged. This is a **single-device specialization**, not a general prepared-state API or proof of a multi-NIC performance improvement.

Absolute times varied materially between cohorts. Report every cohort rather than selecting the fastest result or treating the historical 105.9 ms as a concurrent baseline. CPUs were not pinned; GPU-process sampling does not prove exclusive CPU or network access. No claim of statistical significance or P99 improvement is made.

## End-to-end comparison (no profiling instrumentation)

Each variant has two independent process pairs, two warmups and ten measured waves per pair: **20 measured waves per variant per cohort**. Second pass reverses variant order. Entries are medians of individual measured waves, not averages of process medians.

| Cohort | Baseline submit ms | Full patch submit ms | Reduction | Baseline post→DONE ms | Full patch post→DONE ms |
|---|---:|---:|---:|---:|---:|
| 1: `20260915T212858Z` | 131.895 | 80.027 | 39.3% | 960.199 | 880.787 |
| 2: `20260915T213524Z` | 106.699 | 60.506 | 43.3% | 751.142 | 698.170 |
| 3: `20260915T213858Z` | 136.933 | 80.083 | 41.5% | 981.155 | 870.623 |

Cohort 2 was the most stable: submit **106.699 → 60.506 ms** (43.3% lower), post→DONE **751.142 → 698.170 ms** (7.1% lower), cumulative foreground check **27.735 → 27.651 ms**. This removes synchronous preparation work; it does not eliminate asynchronous posting, DMA, polling or final cleanup.

### Ablation: cohort 3

| Configuration | Submit P50 ms | Cumulative check P50 ms | Post→DONE P50 ms |
|---|---:|---:|---:|
| MR + batch query + CQ baseline | 136.933 | 137.319 | 981.155 |
| + transport reuse only | 107.405 | 137.830 | 947.544 |
| + single-device MR selection and NIC-path reuse | 80.083 | 132.336 | 870.623 |

Transport-only was noisy: cohort 1 measured 131.895 → 133.221 ms, whereas cohort 3 measured 136.933 → 107.405 ms. Therefore **do not assign a stable 29.5 ms saving to transport reuse alone**. The complete patch consistently improved all three cohorts. Topology/device reuse and NIC-path reuse are combined in the full variant; this experiment does not isolate their individual millisecond contributions.

All variants retain the same task/slice storage, completion counter, status-query semantics and worker enqueue algorithm. In particular, the varying cumulative-check figures are not a new completion optimization.

## Exact environment

- Host `gb200-rack1-01`, NVIDIA GB200 GPU0→GPU1, ARM Neoverse-V2, Ubuntu 24.04; same-host RDMA via `rocep139s0:1`, not NVLink.
- 207,360 descriptor pairs; 32,256 bytes per descriptor, 16-byte gaps; 256 separately registered CUDA regions per endpoint. 6,688,604,160 payload bytes per handle. Descriptors are grouped by region, 810 per region; each request is one RDMA slice here.
- READ, one in-flight handle, notification enabled, fresh handles each wave, one foreground caller, two TE workers/context, 64 KiB slice size, 100 us polling, EDC off, TE metrics off.
- NIXL 1.3.2 debug runtime `4f7f1fd425976230188a303a8adb87e133b840f7`, existing batch-query plugin from `b83499357350426bfd6eb74df98ac01a789e36ba`. Same plugin binary in every variant; actual loaded paths and hashes verified at both endpoints.
- TE main pinned to `9fb95ed0339e15b8f969d32df599f3c8ebcbba2a` + previously preserved MR #4132 and adapted CQ counter patches. The exact baseline is local commit `89cfe69d`.
- Shared Release builds, CUDA enabled; multi-protocol, EDC, intra-NVLink and MNNVL disabled for timing. Native correctness tests additionally built with multi-protocol enabled and static linking.
- PyTorch 2.14.0+cu130 / CUDA 13.0; exact runtime manifests, addresses, commands and library SHA256 are retained for every pair.

## What changed

1. **Transport selection:** reuse the last successful target's transport for consecutive requests in one submit. Mixed-protocol segments continue routing every address independently. Explicitly disabling metadata caching also disables this shortcut. No cache survives a submit call.
2. **Local and peer MR/device selection:** function-local last-region cache, with a held metadata snapshot and range checks on every hit. Cache only successful first-attempt resolutions with exactly one topology HCA, a complete non-overlapping MR index, and a non-`segments:` location. Multi-device, segmented-location, overlapping or out-of-range cases keep the resolver. Local context-active checks, uncached retry selection and remote metadata-refresh behavior remain in place. The cache requires stable live registrations during the transfer, as does the existing submission path; it is not a new concurrent-deregistration contract.
3. **NIC path:** lazily construct the path once per snapshot/device encountered in a WorkerPool submit. Replacing the snapshot clears the path cache. Check rail availability and alternatives per slice; do not cache rail health. Every slice still owns its path string.

The fast path benefits descriptors that stay in the same region consecutively. Randomly interleaving regions can erase the device-cache benefit. It does not freeze a choice on multi-NIC topologies; a general solution needs reusable candidate sets while preserving random/round-robin, affinity and failover behavior.

### Complexity and confirmed work reduction

Let N be descriptors, R be consecutive MR runs and P be distinct device paths within a snapshot run. The old MR patch already eliminated full MR-list scans in this workload. This patch reduces expensive metadata/transport decisions from N to target runs, device/topology resolution from N to R on the eligible single-device path, and NIC-path construction from slices to P. Address checks, task/slice construction, copying a path into each slice and enqueue remain O(N).

A fresh 4,096-descriptor baseline diagnostic produced 4,099 metadata lookups and 4,096 calls each for local topology, peer topology and NIC-path construction (`performance-red.json`). A separate full-size diagnostic of the patch asserts, **in every one of ten waves**:

| Work per 207,360-descriptor batch | Patched calls |
|---|---:|
| Segment metadata lookup | 4 |
| Local topology resolution | 256 |
| Peer topology resolution | 256 |
| NIC-path construction | 1 |

The earlier full-size baseline diagnostic observed 207,363 / 207,360 / 207,360 / 207,360 respectively. That earlier diagnostic supplies counts, not a new timing comparison.

## Remaining submit work

Separate coarse diagnostic of the full patch, eight measured waves, API post P50 61.734 ms. These are **not** the uninstrumented timing samples above. Child stages are nested and independent medians do not add.

| Stage | P50 ms |
|---|---:|
| TE submit total | 58.923 |
| Transport selection loop | 0.638 |
| Task construction/grouping | 5.588 |
| RDMA submit total | 52.653 |
| RDMA preparation before WorkerPool | 27.731 |
| Foreground WorkerPool total | 25.141 |
| Peer mapping / path / rail checks | 13.811 |
| Software enqueue | 10.087 |

RDMA submit contains both pre-WorkerPool preparation and WorkerPool work; peer mapping and enqueue are inside WorkerPool. The 27.7 ms pre-WorkerPool interval mixes boundary checks, slice allocation/initialization and grouping; it is not an isolated allocator measurement. Transport selection is now ~0.64 ms, but task/slice storage and per-slice software enqueue remain useful next profiling targets. Nothing here demonstrates UCX-level submit performance, production TTFT/TPOT improvement, or multi-NIC throughput behavior.

## Correctness and checks

- **211 audited waves; 41,923,584 descriptor pairs; 1,352,287,125,504 payload bytes** across smoke, all three timing cohorts and the full-size diagnostic. The initial two small red-diagnostic waves are excluded from this total.
- Unexpected transfer failures, payload/gap/guard corruptions and notification errors: **0**. Full generation/region/offset payload validation at both endpoints; immediate initiator validation after DONE, descriptor/byte telemetry checks, and exact expected RDMA READ count for each case.
- Captured NIC error/retry/drop/out-of-order/duplicate counter increments: zero. NIC counters are node-wide.
- Each case checks GPU memory before launch. One-second process samples throughout each case contain only the recorded benchmark endpoint PIDs. This is sampled GPU isolation, not proof of exclusive CPU/network access.
- **93 native tests pass** with static linking and multi-protocol enabled: transport 32, locality 8, RDMA submit failure handling 4, MR index 8, rail state/recovery 15, topology 13, new batch-device cache 8, new batch routing 5.
- New device-cache test was observed red (16 resolver calls rather than 1) before cache implementation, then green. Edge coverage includes boundaries/overflow, gaps, overlapping MRs, segmented locations, multi-rail fallback, snapshot replacement and failed resolution; routing covers alternating targets, next-batch refresh, unsupported target, twosided fallback and mixed-protocol address routing.
- The upstream GID recovery test initially fails in the shared-library baseline because its link-time `--wrap` hook does not intercept the call inside the shared library. Correct static-link test build passes all 15 rail tests, without changing recovery code or that test.
- PR-scoped pre-commit checks pass with the staged-format wrapper skipped because `git-clang-format` is absent. Equivalent changed-line formatting was explicitly checked with clang-format 20.1.8, yielding zero replacements. CMake formatting and `git diff --check` pass.

## Reproduce and inspect

All local artifacts live next to the isolated worktrees:

```bash
cd /home/zhewen/repos/nixl-internal/.local/te-batch-reuse-20260915
source /home/zhewen/repos/nixl-internal/benchmark/kv_submit_repro/env.sh
# Existing measured libraries are preserved in install-baseline/route/full.
python3 run.py main                     # baseline → route → full, then reverse
python3 run.py main --variants baseline,full
# Optional rebuild: build.py creates separate baseline/route source worktrees,
# applies the exact route patch, and uses source/ for the complete patch.
python3 build.py baseline
python3 build.py route
python3 build.py full
```

Rebuilding a named variant intentionally replaces only its experiment-local installation; retain `library-hashes.json` and measured binaries if auditing an old run. The current full source differs from the measured source only by formatting/include placement and test formatting; `full-measured.patch` plus `measured-source-hashes.json` preserve the exact measured revision. No timing logs are present in the functional branch.

Audit all completed cases:

```bash
python3 analyze.py \
  smoke-20260915T212807Z \
  main-20260915T212858Z \
  main-20260915T213524Z \
  main-20260915T213647Z \
  main-20260915T213858Z
```

- `analysis.json`, `cohorts.json`: validated sample distributions and cohort medians.
- `full-measured.patch`, `route-measured.patch`, `library-hashes.json`, `measured-source-hashes.json`: exact experiment inputs.
- `performance-red.json`, `performance-green.json`, `cache-red.log`: performance regression witnesses.
- `profile-source/`, `instrument.py`, `build-profile.py`, `run-profile.py`: isolated coarse instrumentation; no hot-loop log emission.
- `build-static.py`, `static-multi-tests.json`, native test logs, `format-check.json`, `precommit-final.log`: correctness and formatting evidence.

No existing checkout or published PR was modified. This branch and report are local; no push or new PR was requested.
