# Handover — 2026-07-07

**Branch:** `main` (issue-98-document-timeline closed)

## What Happened This Session

### #98 Document timeline — complete and merged

Added a document timeline showing how a document evolved across review rounds.
`ROUND_SNAPSHOT` entries flow through the channel → WebSocket → pages-event pipeline.
A thin timeline strip above the diff panel consumes snapshot events, with clickable
markers for version comparison and review tracker trail highlighting.

**What was delivered:**
- `SnapshotSource` sealed interface with `GitCommit` variant, `DocumentSnapshot`, `DocumentTimeline` in `server/api/`
- `ROUND_SNAPSHOT` entry type added to `EntryType` — domain entry, not infrastructure provenance
- `DebateStreamEntry` gains nullable `commitHash` and `documentPath` fields
- `DebateChannelProjection.apply()` override intercepts ROUND_SNAPSHOT (PP-20260610-a47ef5 compliant)
- `WorkspaceParser` extracts `commitHash` from tracker and `projectRepoPath` from `.source-dirs`
- `WorkspaceReplayAdapter` emits ROUND_SNAPSHOT at round boundaries, pre-loads content via `git show`
- `GET /api/debate/{id}/snapshot/{index}` REST endpoint serves pre-loaded content
- `<drafthouse-timeline>` Web Component — adjacent/shift-click comparison, review tracker trail highlight
- Diff panel `loadContent()` method, `timeline-comparison-changed` listener
- Review tracker `point-selected` enriched with `raiseRound`, `fixRound`, `verifyRound`
- 5 E2E tests with programmatic git repo fixture, 4 unit tests, 2 integration tests

**Design-reviewed:** 3 rounds, 11 issues, 11 verified, $14.10
**Code-reviewed:** APPROVE — 1 Important fixed (process timeout), 4 Minor → #103

**Also this session:**
- Filed Hortora/soredium#79 for design-review structured output (JSONL sidecar)
- Garden entry GE-20260707-a48ac6 — GitHub Packages SNAPSHOT version metadata gotcha
- Protocol PP-20260707-0cb860 — panel configure() idempotency guard

## Immediate Next Step

**#99 — Live workspace watching.** Now unblocked by #98. The same `ROUND_SNAPSHOT` entry type and timeline panel work for live watching — the live adapter emits snapshots as rounds complete and the timeline grows incrementally.

## What's Next

| # | Title | Scale | Complexity | Blocked by | Notes |
|---|-------|-------|------------|------------|-------|
| #93 | Document workbench (epic) | XL | High | — | 5 remaining child issues |
| #96 | design-review structured output | M | Med | — | JSONL sidecar — soredium#79 filed |
| #99 | Live workspace watching | M | Med | ~~#98~~ | **Now unblocked** — real-time monitoring |
| #97 | Chunked orchestration research | M | High | #96 | Cost/UX tradeoff study |
| #100 | Channel-based HIL | L | High | #97, #99 | Concurrent human participation |
| #101 | Panel extraction | XL | High | all above | @casehubio components |
| #103 | Timeline panel minor cleanup | XS | Low | — | Listener cleanup, timestamp, labels |
| #92 | Adopt pages-push typed protocol SDK | S | Low | — | Independent |
| #89 | AgentProvider migration | M | Med | — | Independent |

## References

| Context | Where |
|---------|-------|
| #98 design spec | `docs/superpowers/specs/2026-07-07-document-timeline-design.md` |
| #98 design review workspace | `~/adr/casehub-drafthouse/document-timeline-*/` |
| Garden entry | GE-20260707-a48ac6 — GitHub Packages SNAPSHOT version metadata gotcha |
| Protocol | PP-20260707-0cb860 — panel configure() idempotency |
| Soredium issue | Hortora/soredium#79 — design-review structured output |
| Epic | casehubio/drafthouse#93 |
