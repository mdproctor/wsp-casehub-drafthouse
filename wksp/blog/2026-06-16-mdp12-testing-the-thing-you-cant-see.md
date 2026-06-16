---
layout: post
title: "Testing the thing you can't see"
date: 2026-06-16
type: phase-update
entry_type: note
subtype: diary
projects: [DraftHouse]
tags: [e2e, playwright, sse, web-components]
---

## Testing the thing you can't see

The debate panel, review tracker, and cross-panel coordination all render from SSE-delivered events. The data moves through five layers before a pixel changes: `DebateMcpTools` dispatches to Qhorus, Qhorus persists the message, `DebateEventResource` polls and serialises, `EventSource` delivers to the browser, and the Web Component's subscriber callback rebuilds the DOM.

Testing the UI in isolation — injecting mock data into the `DebateEventBus` — would prove the panel renders when fed data. But the interesting bugs live in the gaps between layers: SSE serialisation dropping fields, the projection folding an entry type wrong, the event bus reconnect handler clearing state it shouldn't. The only test that catches those is one that drives the full chain.

So the pattern: `@QuarkusTest @WithPlaywright` in the same test class. CDI injects `DebateMcpTools`; Playwright injects `BrowserContext`. The test calls `raisePoint()` on the server, navigates the browser with `?debate=<sessionId>`, waits for `locator(".entry-raise").waitFor()`, and asserts the DOM. The SSE poll interval is 500ms; Playwright's default 30s timeout makes the wait invisible.

The design review surfaced a genuine domain gap: `EntryType.DECLINED` exists, `DebateChannelProjection` handles it, the review tracker renders it with strikethrough — but `respondTo()` didn't accept `"declined"` as an input. The entry type was unreachable from the MCP API. The #51 spec had flagged this ("including DECLINED after prerequisite fix") but no issue existed. Filed #57, added the switch case. One line of production code, but without it the E2E tests for DECLINED rendering would have been untestable through the intended path.

The sub-agent error test needed a different trick. `MockDebateAgentProvider` always succeeds — it returns a fixed string. But `ChannelAgentDispatcher` has a no-handler-matched path: pass `taskType="NONEXISTENT"` to `requestSubagent()`, and the handler dispatch pattern (`SubTaskType.valueOf()` throws, `handles()` returns false for all handlers) triggers `dispatchError()` with a `SUB_TASK_ERROR` entry. The error path is testable from the MCP surface without modifying the mock.

Three tests can't fully execute right now. The `casehub-ledger` SNAPSHOT added a `TENANCY_ID` column to `LedgerMerkleFrontier` that Flyway hasn't migrated in the test H2 database. The message persists (separate transaction phase), but the exception kills `ChannelGateway.fanOut()` — the async chain to `ChannelAgentDispatcher` never fires. The `subTaskFinding`, `subTaskError`, and `restartContext` tests use `Assumptions.assumeTrue()` instead of silent returns, so CI reports them as SKIPPED rather than falsely PASSED. They'll activate when ledger ships the migration.

38 tests total: 18 debate panel (entry types, round grouping, auto-scroll, sub-agent rendering, restart-context), 16 review tracker (status derivation, progress bar, filter toggle, sort order, agent trail), 4 cross-panel (section-ref scroll, text-ref scroll, no-location no-op). The cross-panel tests for `scrollToLocation("§3")` exercise an asymmetry I hadn't considered during design: §3 resolves to "Features" on side A (which has a Preface heading) but "Scroll Sync" on side B (which doesn't). Both scroll correctly — the assertion just checks `scrollTop > 0` on each side rather than demanding identical positions.
