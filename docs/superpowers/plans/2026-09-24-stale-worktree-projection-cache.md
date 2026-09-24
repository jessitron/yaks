# Stale worktree projection cache implementation plan

**Issue:** GitHub #10

## Goal

Keep `.yaks` as a fast, disposable projection while ensuring every command starts from all yak events committed before its freshness check. Worktrees share `refs/notes/yaks` but retain independent projections.

## Delivery strategy

Ship two independently merged yaks.

### Yak 1: remove reset git from disk

`remove-reset-git-from-disk-zrwo`

Remove the moribund reverse-recovery path before making projection files explicitly disposable.

1. Add a failing CLI test proving `reset --git-from-disk` is rejected.
2. Remove the `--git-from-disk` and `--force` reset flags. Keep plain `yx reset` and the compatible explicit `--disk-from-git` spelling as disk-from-Git.
3. Remove `ResetGitFromDisk`, its tests, exports, Cucumber scenarios, and step definitions.
4. Remove obsolete event-store API surface that only supported reverse reset where possible.
5. Add an ADR superseding ADR 0010's reverse-reset decision. Do not edit accepted ADRs.
6. Update current documentation and changelog. Historical specs remain historical unless they are presented as current documentation.
7. Run focused tests, mutation tests for the diff, and `dev check`; commit and merge with `dev merge`.
8. Mark the yak done.

### Yak 2: refresh stale worktree projections incrementally

`refresh-stale-worktree-projections-incrementally-5jhk`, blocked by Yak 1.

#### Consistency contract

- Git is exclusively authoritative; `.yaks` is disposable.
- A command must not knowingly use a projection older than the event-ref revision observed by its startup freshness check.
- Failure to establish freshness fails the command rather than serving stale data.
- Normal refresh is silent.
- Cache hits must retain the CLI's sub-100ms target. Correctness takes priority during large catch-ups and rebuilds.

#### Design

1. Add an opaque event-stream revision type and a versioned event read API.
2. Add an event-store operation that classifies projection refresh as:
   - current;
   - incremental, with events after an ancestor checkpoint in oldest-first order;
   - full rebuild, when the checkpoint is absent, invalid, or not an ancestor.
3. Add a projection checkpoint port implemented by `DirectoryStorage`. Store an explicit representation of both an OID and an empty event ref. Write it atomically only after successful projection.
4. Add a worktree-local projection lock held for the CLI command lifetime. Concurrent commands in different worktrees remain independent.
5. Before command routing, compare the checkpoint with `refs/notes/yaks` and apply the classified update. Missing/corrupt checkpoints cause a full rebuild.
6. Keep the checkpoint behind after ordinary local mutations in the first implementation. The next invocation incrementally and idempotently replays the missing range.
7. Make plain `yx reset`, migrations, and other full-rebuild paths write an exact checkpoint after successful replay. Ensure sync and compaction leave a safely detectable checkpoint state.
8. Do not add continuous projection checksums. Plain `yx reset` remains the repair tool for corruption that does not affect the checkpoint.

#### TDD scenarios

1. Reproduce issue #10 with two worktrees: prime worktree B, then update context and remove a yak in A; B immediately sees both changes without sync/reset.
2. Apply only events after an ancestor checkpoint.
3. Skip replay when checkpoint equals the live tip.
4. Full rebuild for missing/corrupt checkpoints and rewritten or deleted refs.
5. Never advance a checkpoint after failed replay.
6. If the ref advances during refresh, record only the exact revision replayed so the next invocation catches up.
7. Serialize concurrent commands sharing one worktree projection.
8. Reapplying locally projected events is idempotent across all event shapes.
9. Cover migration, sync, compaction, reset, empty repositories, and `YX_SKIP_GIT_CHECKS`.

#### Benchmark gates

Using release binaries with warmups and repeated alternating samples, compare baseline and implementation for `list`, `show`, and completion with:

- a current checkpoint;
- one and ten missing events;
- a 20-yak full rebuild;
- a 500-yak/1,000-event full rebuild.

Cache-hit medians and p95 should remain near baseline and below 100ms. Record the methodology and results in the repository.

## Explicit non-goals

- Removing the `.yaks` projection.
- Detecting arbitrary manual edits while the checkpoint remains valid.
- Preserving `reset --git-from-disk` compatibility.
- Immediately advancing the checkpoint after local writes.
- Guaranteeing a large catch-up or full rebuild completes below 100ms.
