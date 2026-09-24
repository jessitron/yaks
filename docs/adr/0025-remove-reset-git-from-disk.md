# 25. Remove Reset Git from Disk

Date: 2026-09-24

## Status

accepted (amends ADR 0010)

## Context

ADR 0010 defined `yx reset --git-from-disk` as a destructive recovery mechanism. It read the worktree-local `.yaks/` projection, wiped `refs/notes/yaks`, and recreated the event stream from the projection.

The projection is derived state, not an authoritative source. Treating it as a recovery source weakens that boundary and is particularly unsafe across worktrees, where each worktree has an independent projection but all worktrees share the Git event ref. A stale projection could therefore replace newer authoritative events.

The command is moribund, while its confirmation flow, event-store `wipe` operation, replay logic, tests, and documentation add significant complexity to projection lifecycle changes.

## Decision

Remove `yx reset --git-from-disk` and its `--force` option without a compatibility period.

Keep `yx reset` and the explicit `yx reset --disk-from-git` spelling. They discard and rebuild the `.yaks/` projection from the authoritative Git event stream.

Remove event-store wipe functionality that existed only to support reverse reset. Event compaction remains the supported mechanism for reducing event history.

## Consequences

- Git is unambiguously authoritative and `.yaks/` is disposable derived state.
- A stale or corrupted projection can no longer overwrite the event stream through the CLI.
- Projection freshness and repair can be simplified around a one-way Git-to-disk flow.
- Users can no longer reconstruct a damaged event ref from `.yaks/`; recovery must use Git backups/remotes or other Git-level mechanisms.
- Existing scripts using `--git-from-disk` or reset's `--force` option fail argument parsing and must be updated.
