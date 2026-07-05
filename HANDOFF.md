# Handover — 2026-07-05

**Branch:** `main` (clean)

## Last Session

Closed #90 (section highlight heading-match edge cases) in previous session. This session: attempted to manually verify point selection works in the browser. DraftHouse server is running on port 9001 with an active debate session (`1b7c596c-41c0-46ec-a631-dfee9bc1c8ac`). Installed Playwright MCP (`@playwright/mcp@latest`) to enable browser inspection — requires session restart to connect.

Build note: `casehub-ledger` HEAD has a breaking change (moved `LedgerEntry` from `runtime.model` to `api.model`) that qhorus hasn't absorbed. Use `-nsu` flag on drafthouse builds to avoid re-pulling broken SNAPSHOT, or rebuild ledger from commit `aecf98e`.

## Immediate Next Step

Verify point selection works in the browser using Playwright MCP. Server is already running on port 9001 with a debate session. Navigate to:

```
http://localhost:9001/?a=/Users/mdproctor/drafthouse-demo/sample-a.md&b=/Users/mdproctor/drafthouse-demo/sample-b.md&debate=1b7c596c-41c0-46ec-a631-dfee9bc1c8ac
```

Use `browser_snapshot` to inspect the UI, click review points in the review tracker panel, and confirm section highlighting activates on the diff panel.

## What's Left

| # | Title | Scale | Complexity | Blocked by | Blocks | Notes |
|---|-------|-------|------------|------------|--------|-------|
| #89 | Migrate from LangChain4j ChatModel to platform AgentProvider | M | Med | — | — | `casehub-platform-agent` shipped; stub ready |
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
| Playwright MCP | installed via `claude mcp add playwright -- npx @playwright/mcp@latest` — project-local config |
| Ledger breakage | casehub-ledger HEAD moved `LedgerEntry` to api module; qhorus compile fails. Compatible jar: commit `aecf98e` |
