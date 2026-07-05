# Handover — 2026-07-05

**Branch:** `main` (clean)

## Last Session

Closed #90 — section highlight heading-match edge cases. Added `_normalizeLocation()` to `_findHeading`: strips quotes (ASCII + Unicode smart quotes), common LLM prefixes (Section:, Heading:, Under, In the), suffixes (section, heading, area, part), generalized separator splitting (/ - — and or), word-overlap fallback scoring. Removed diagnostic console.log from review tracker. 15 new parameterized E2E tests (45 total for section highlight). All 426 runtime tests pass.

Build note: `casehub-ledger` HEAD has a breaking change (moved `LedgerEntry` from `runtime.model` to `api.model`) that qhorus hasn't absorbed. Restored the June 30 jar to unblock. Use `-nsu` flag on drafthouse builds to avoid re-pulling broken SNAPSHOT, or rebuild ledger from commit `aecf98e`.

## Immediate Next Step

```
/work
```

Pick up #89 (AgentProvider migration — `casehub-platform-agent` has shipped) or #53 (brainstorming UI).

## What's Left

| # | Title | Scale | Complexity | Blocked by | Blocks | Notes |
|---|-------|-------|------------|------------|--------|-------|
| #89 | Migrate from LangChain4j ChatModel to platform AgentProvider | M | Med | — | — | `casehub-platform-agent` shipped; `ClaudeAgentSdkDebateAgentProvider` stub ready |
| #84 | Brainstorming UI on casehub-pages (epic) | L | High | — | #53 | #75 done; #53 still open |
| #53 | Brainstorming UI — richer option exploration | L | High | #84 (epic) | — | Needs design before code |
| #72 | Review pipeline orchestration | L | High | — | — | Design issue |
| #71 | Claude-to-Claude continuous conversation protocol | L | High | — | — | Design issue |
| #60 | Selection-scoped conversation channels | M | Med | — | — | |
| #61 | GraalVM native image build | M | Med | — | — | Paused |

## References

| Context | Where |
|---|---|
| Architecture record | `ARC42STORIES.MD` |
| GitHub | `casehubio/drafthouse` |
| Ledger breakage | casehub-ledger HEAD moved `LedgerEntry` to api module; qhorus compile fails against it. Compatible jar: commit `aecf98e` or cached `casehub-ledger-0.2-20260630.090012-236.jar` |
