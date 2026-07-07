# Handover — 2026-07-07

**Branch:** `main` (issue-95-replay-adapter closed)

## What Happened This Session

### #95 Replay adapter — complete and merged

Built a replay adapter that reads completed design-review workspace directories and projects them as interactive debate sessions in DraftHouse via Qhorus channel dispatch.

**What was delivered:**
- `WorkspaceParser` — ports 8 Python regex patterns to Java, extracts issues, responses, confirmations, assumptions, settled decisions, signals, and tracker terminal statuses
- `WorkspaceReplayAdapter` — creates Qhorus channel, encodes entries with `DHMETA:` sentinel, dispatches via `MessageService.dispatch()` with `DispatchResult.messageId()` for `inReplyTo` linkage, batch-pushes to WebSocket
- `load_workspace` MCP tool on `DebateMcpTools` — idempotency via `channelService.findByName()`, error recovery
- `VERIFIED` and `DEFERRED` added to `EntryType`, `DebateChannelProjection`, and review tracker panel
- 3 E2E tests, 1 integration test, 12 parser unit tests, 20 projection unit tests

**Design-reviewed:** 3 rounds, 23 issues, 19 verified, 2 accepted, $16.15

## Immediate Next Step

**#96 — design-review structured output (JSONL sidecar).** This is the highest-leverage next issue: it simplifies #99 (live watching reads JSONL instead of re-parsing markdown) and makes the replay adapter more robust. Note: #96 is a soredium change (Python, `~/.claude/skills/design-review/`), not a drafthouse change. The DraftHouse side is small — add a JSONL-first branch to `WorkspaceReplayAdapter` that falls back to `WorkspaceParser` for old workspaces.

## What's Next

| # | Title | Scale | Complexity | Blocked by | Notes |
|---|-------|-------|------------|------------|-------|
| #93 | Document workbench (epic) | XL | High | — | 6 remaining child issues |
| #96 | design-review structured output | M | Med | ~~#95~~ | **Now unblocked** — JSONL sidecar |
| #98 | Document timeline | L | Med | ~~#95~~ | **Now unblocked** — version navigation |
| #99 | Live workspace watching | M | Med | ~~#95~~ | **Now unblocked** — real-time monitoring |
| #97 | Chunked orchestration research | M | High | #96 | Cost/UX tradeoff study |
| #100 | Channel-based HIL | L | High | #97, #99 | Concurrent human participation |
| #101 | Panel extraction | XL | High | all above | @casehubio components |
| #92 | Adopt pages-push typed protocol SDK | S | Low | — | Independent |
| #89 | AgentProvider migration | M | Med | — | Independent |

## References

| Context | Where |
|---------|-------|
| #95 design spec | `docs/specs/issue-95-replay-adapter/2026-07-06-replay-adapter-design.md` |
| Example workspace | `~/adr/casehub-drafthouse/document-workbench-20260705-224726/` |
| Garden entry | GE-20260707-674928 — ChannelService.delete FK constraint gotcha |
| Epic | casehubio/drafthouse#93 |
