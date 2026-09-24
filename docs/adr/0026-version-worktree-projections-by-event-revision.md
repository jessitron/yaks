# 26. Version Worktree Projections by Event Revision

Date: 2026-09-24

## Status

accepted

## Context

Git stores the authoritative yak event stream at `refs/notes/yaks`. Git keeps that ref in the repository's common directory, so every worktree sees the same events. Each worktree has a separate `.yaks/` filesystem projection.

Previously, a command updated only the projection in the worktree where it ran. Commands in another worktree continued reading their local projection without knowing the shared event ref had advanced. They could silently display deleted yaks or old context and make decisions from stale data.

Rebuilding the full projection before every command would restore correctness but would discard the performance benefit of the filesystem read model. Removing the projection and replaying the event stream in memory was benchmarked, but uncompacted streams exceeded the CLI's 100ms target.

## Decision

Treat `.yaks/` as a disposable, versioned cache of the Git event stream.

Each worktree records the exact projected event revision in `.yaks/.projection-tip`. Before using the projection, a command compares that checkpoint with `refs/notes/yaks`:

- equal revisions require no work;
- when the checkpoint commit is an ancestor of the current tip, replay only intervening commits in chronological order;
- a missing, malformed, deleted, or non-ancestor checkpoint causes a full rebuild.

Write the checkpoint atomically and only after every projected event succeeds. An empty event stream has an explicit checkpoint value so it is distinguishable from an unversioned projection.

Serialize commands sharing one worktree with a worktree-local advisory lock. Different worktrees remain concurrent and continue coordinating writes through the event ref's compare-and-swap behavior.

Ordinary local writes update the projection but initially leave its checkpoint behind. The next command safely and idempotently catches up. This avoids claiming completeness when another writer may have inserted an event concurrently.

Git remains exclusively authoritative. `yx reset` explicitly discards and rebuilds the projection. Arbitrary filesystem tampering while the checkpoint remains valid is not continuously detected.

## Consequences

- Changes made in one worktree become visible to another without `yx sync` or manual reset.
- The common cache-hit path adds only a ref lookup and checkpoint read.
- Typical stale projections apply only a small number of missing events rather than replaying full history.
- Rewritten history and first use after upgrade incur a full rebuild.
- A failed or interrupted update leaves the old checkpoint, causing a retry rather than silent stale reads.
- Concurrent commands in the same worktree may wait up to the projection-lock timeout and then fail clearly.
- `.yaks` is not a user-editable source of truth and does not need content checksums.
