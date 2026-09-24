# Issue 10 projection cache invalidation benchmark

## Summary

The versioned projection branch keeps the warm cache-hit path comfortably below the CLI's 100 ms target. Across `list`, `show`, and completion, branch cache-hit medians were 17.41–33.53 ms and p95s were 25.18–50.27 ms. The cache-hit overhead relative to main was small: +0.15 ms for completion, +0.33 ms for `show`, and +2.03 ms for `list` at the median.

Incrementally applying one missing event took about 18–20 ms. Ten missing events took about 32–33 ms. A first rebuild of 20 yaks took about 34 ms. All of those branch p95s remained below 51 ms.

A 500-yak, 1,000-event catch-up was feasible but slow: approximately 10.4–11.6 seconds median, depending on the command. This does not affect the cache-hit gate, but it identifies a clear future optimization opportunity in large projection catch-ups.

## Versions and environment

- **Baseline:** `main` at `aa36c369131fef577964f4004ddc11db2a49b603`
- **Measured branch:** `refresh-stale-worktree-projections` at `01d7315c93e25ecbc429f33fc2a1910405167f37`
- Both binaries were built with `cargo build --release`.
- Rust: `rustc 1.94.0 (4a4ef493e 2026-03-02)`
- Host: Linux 6.8.12-8-pve, Intel Core i7-8559U, four online logical CPUs during the run, 4 GiB RAM

The benchmark was completed before the later boundary/lock commits `106ef6b`, `2bde85a`, and `5abb763`; those commits do not change the measured event traversal or normal uncontended cache-hit algorithm, but they were not part of the measured binary.

## Methodology

All fixtures were temporary Git repositories under `/tmp`, selected with `YX_ROOT`. The project's real `.yaks` directory was never read or modified. Benchmarking made no production-code changes.

Each operation/scenario combination used:

1. five untimed warmups per binary;
2. 25 measured samples per binary;
3. strict alternation between main and branch invocations;
4. warm filesystem/page caches;
5. wall-clock timing around the release-binary subprocess using Python's `perf_counter_ns`;
6. median and nearest-rank p95 over the 25 observations.

Fixture restoration happened before, and outside, each timed branch invocation. Command output was captured. Before measurement, output from main's current projection and the branch's refreshed projection was compared byte-for-byte for every scenario and command.

The measured commands were:

```text
yx list --format plain
yx show "yak 0001"
yx completions -- yx show yak
```

The completion measurement exercises the CLI candidate-generation endpoint used by `completions/yx.bash`; it deliberately excludes Bash startup and `compgen` overhead.

Main has no stale-projection refresh behavior. For a meaningful output comparison, main therefore used an already-current projection in the missing-event and rebuild scenarios, while the branch started stale or absent and established the same current projection during the timed command. Consequently, main's figures in those rows are the old valid cache-hit cost, not the cost of equivalent invalidation logic.

### Fixtures

- **Current checkpoint/cache hit:** 20 yaks; branch checkpoint equals `refs/notes/yaks`.
- **One missing event:** 20-yak checkpoint and projection, with one subsequent `Added` event in Git.
- **Ten missing events:** 20-yak checkpoint and projection, with ten subsequent `Added` events in Git.
- **20-yak first rebuild:** 20 `Added` events with no branch `.yaks` projection or checkpoint.
- **500-yak/1,000-event catch-up:** 500 `Added` events followed by 500 `StateChanged` events. The branch started with an explicit empty checkpoint and caught up all 1,000 events.

## Results

All values are milliseconds, shown as **median / p95**.

| Scenario | Operation | Main | Branch |
|---|---:|---:|---:|
| Current checkpoint/cache hit | `list` | 31.49 / 50.80 | 33.53 / 50.27 |
| Current checkpoint/cache hit | `show` | 18.38 / 27.04 | 18.71 / 30.49 |
| Current checkpoint/cache hit | completion | 17.25 / 22.28 | 17.41 / 25.18 |
| One missing event | `list` | 17.37 / 30.76 | 20.24 / 27.10 |
| One missing event | `show` | 16.56 / 20.66 | 19.81 / 22.90 |
| One missing event | completion | 15.60 / 16.55 | 18.01 / 19.39 |
| Ten missing events | `list` | 16.89 / 18.23 | 32.12 / 36.12 |
| Ten missing events | `show` | 17.21 / 19.33 | 32.37 / 36.70 |
| Ten missing events | completion | 17.13 / 27.81 | 33.36 / 50.25 |
| 20-yak first rebuild | `list` | 16.48 / 21.00 | 33.57 / 43.42 |
| 20-yak first rebuild | `show` | 17.40 / 22.77 | 34.69 / 44.28 |
| 20-yak first rebuild | completion | 17.20 / 24.86 | 33.93 / 42.42 |
| 500-yak/1,000-event catch-up | `list` | 83.77 / 344.13 | 11,458.03 / 18,605.79 |
| 500-yak/1,000-event catch-up | `show` | 70.50 / 141.90 | 10,363.69 / 12,623.35 |
| 500-yak/1,000-event catch-up | completion | 58.88 / 969.00 | 11,638.25 / 17,916.34 |

The large-fixture baseline tail was noisy on this memory-constrained shared host, but its baseline is only contextual: main did no catch-up work. The branch's large-fixture times consistently show that replaying and materializing 1,000 events is measured in seconds rather than milliseconds.

## Assessment

**Cache-hit gate: pass.** Every branch cache-hit median and p95 is below 100 ms. Cache-hit performance remains close to main, including the projection lock, checkpoint read, and Git revision comparison.

Small stale projections also remain interactive: one-event catch-up, ten-event catch-up, and a 20-yak first rebuild all had p95 below 51 ms.

The 500-yak/1,000-event result is acceptable under the stated gate, which prioritizes correctness and does not require large catch-ups below 100 ms. However, 10–12 second medians are noticeable enough to warrant follow-up profiling if repositories of this size are expected in normal use.
