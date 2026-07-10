# Handover — 2026-07-11

**Branch:** `main` (issue-103-cleanup-batch closed)

## What Happened This Session

### #103, #92, #89 — cleanup batch (closed)

Three issues on one branch:

- **#103 Timeline cleanup:** Added `timestamp` and `label` metadata to ROUND_SNAPSHOT
  entries, actual git commit timestamps via `gitCommitTimestamp()`, removed label
  `split(' — ')` hack in timeline panel. Findings 1&2 (listener cleanup) were already
  resolved by #104 Lit migration.

- **#92 Pages-push SDK:** Replaced manual JSON formatting with `casehub-pages-push`
  typed SDK. `PushMessage.event()` for outgoing wire format, `PushRequest.parse()` for
  incoming messages (sealed interface), `TopicRegistry` replaces manual watcher maps.
  Server now sends `ack`/`error` responses for request correlation.

- **#89 AgentProvider migration:** Removed `quarkus-langchain4j-anthropic` dependency
  entirely. `DocumentReviewer` converted from `@RegisterAiService` to concrete class
  using `AgentProvider.invoke()`. `PlatformDebateAgentProvider` is the new `@DefaultBean`.
  `ClaudeAgentSdkDebateAgentProvider` wired to AgentProvider (displaces default).

**ARC42STORIES.MD synced:** 9 stale references updated (dependency table, context
diagram, building block view).

## Immediate Next Step

Pick from the backlog — all items are independent.

## What's Next

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| #99 | Live workspace watching | M | Med | Unblocked by #98 |
| #93 | Document workbench (epic) | XL | High | 5 remaining child issues |
| #96 | Design-review structured output | M | Med | JSONL sidecar — soredium#79 filed |
| #97 | Chunked orchestration research | M | High | Blocked by #96 |
| #100 | Channel-based HIL | L | High | Blocked by #97, #99 |
| #101 | Panel extraction | XL | High | Blocked by all above — panels on Lit, push protocol typed |

## References

| Context | Where |
|---------|-------|
| ARC42 stale scan | commit 801497a on main |
| Memory | `project_lit_migration_direction.md` — all components on Lit |
