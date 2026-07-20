# Handover — 2026-07-20

**Branch:** `main` (#99 closed)

## What Happened This Session

Implemented live workspace watching (#99) — extending `load_workspace` from
one-shot replay to real-time monitoring of in-progress design reviews.

New components:
- `WorkspaceWatcher` — `io.methvin:directory-watcher` (native macOS FSEvents),
  CDI context activation, processedFiles dedup with rollback, catch-up
  reconciliation, progress.log tailing, terminal state detection
- `ProgressLogParser` — sealed interface with 6 typed event records
- `<workspace-status>` — Lit topbar element (elapsed timer, cost, agent status)

Refactored `WorkspaceReplayAdapter` — extracted 7 dispatch methods shared by
both replay and watcher. Added `tenancyId` field for watcher thread CDI bypass.
Extended `ReplayResult` with `raiseMessageIds` and `lastMessageId`.

Design review ran 8 rounds ($25.84) before implementation. Key catches: CDI
context on watcher thread, TOCTOU race fix via processedFiles dedup,
incremental WebSocket push, complete terminal state coverage.

142 tests pass (including 3 new WorkspaceWatcher tests + 16 ProgressLogParser
tests + 1 new LoadWorkspaceTest).

## Follow-up

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| #110 | ROUND_SNAPSHOT + tracker diffing during live watching | S | Low | projectRepoPath/specPath fields already stored |
| #53 | Brainstorming UI slices 3-6 | S-M each | Low-Med | Independent of #93 track |
| #93 | Document workbench (epic) | XL | High | #100 now unblocked |
| #100 | Channel-based HIL | L | High | Unblocked by #99 |
| #101 | Panel extraction | XL | High | Blocked by #100 |

## References

| Context | Where |
|---------|-------|
| Design spec | docs/specs/2026-07-20-live-workspace-watching-design.md |
| Implementation plan | plans/attic/issue-99-live-workspace-watching/ (workspace) |
| Design review workspace | ~/adr/casehub-drafthouse/live-workspace-watching-20260720-015431/ |
| Blog entry | blog/2026-07-20-mdp28-watching-the-watcher.md |
| Garden entry | GE-20260720-082267 (dedup rollback gotcha) |
