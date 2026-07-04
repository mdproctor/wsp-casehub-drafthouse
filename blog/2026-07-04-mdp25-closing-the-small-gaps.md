---
layout: post
title: "Closing the small gaps"
date: 2026-07-04
type: phase-update
entry_type: note
subtype: diary
projects: [DraftHouse]
tags: [websocket, testing, ui]
series: issue-88-websocket-tests-and-badge-dropdown
---

*Part of a series on [#88](https://github.com/casehubio/drafthouse/issues/88) and [#85](https://github.com/casehubio/drafthouse/issues/85). Previous: [Killing the polling loop](2026-07-04-mdp24-killing-the-polling-loop.md).*

Last session ripped out SSE polling and replaced it with a single WebSocket. That left two gaps worth filling: the reconnection model had no tests proving it actually works, and the document badge in the topbar was a dead click target with no dropdown.

Both are S-scale — the kind of work that gets deferred forever if you don't batch it.

## The reconnection contract

The WebSocket push design relies on a specific behavioural contract: on reconnect, the server pushes a `reconnected` event first, the client re-subscribes, and the server sends a full catch-up. Panels reset on `reconnected` and rebuild from the catch-up stream. No incremental recovery, no `since` tokens — just a clean slate with full history.

That's a claim with no test behind it. Three tests now verify the critical paths:

The first closes and reopens a connection mid-debate, re-subscribes, and asserts that the catch-up contains every point raised before and during the first connection. The positional assertion matters: `reconnected` must be the first message, not just present somewhere in the stream. If the server pushed sessions or catch-up before the reconnection signal, panels would accumulate duplicate entries.

The second ends a debate while a client is connected, disconnects, reconnects, and re-subscribes to the dead session. The server should silently ignore it — no catch-up, no error, connection stays open. This is the real-world reconnection scenario: pages reconnects automatically and blindly re-sends every tracked subscription, including ones for sessions that ended while the connection was down.

The third fires `raisePoint()` and `pushMetadata()` simultaneously from two threads synchronized with a `CyclicBarrier`. Both events must arrive as valid JSON — no garbled output from interleaved `sendText()` calls. This validates the Vert.x per-connection serialization guarantee under our actual usage pattern.

## The dropdown that was always supposed to exist

The document badge — `📄 3` in the topbar — has been there since the multi-document working set landed in June. It showed the count, had `cursor: pointer`, and did nothing when clicked. The comparison was only changeable via MCP tools.

`<drafthouse-doc-picker>` is a Shadow DOM custom element that replaces the badge span. Click it, a dropdown appears with A/B toggle buttons per document. Click A on a doc, it becomes the left side of the diff. The POST goes to the existing `/api/debate/{id}/comparison` endpoint — no server-side changes needed.

The interesting design problem was first-time assignment. When no comparison exists yet, clicking A on a document can't immediately POST — there's no pathB to send. The solution is a pending state: the first click marks a pending slot (dashed border), the second click fills the other slot and triggers the POST. Once a comparison exists, reassignment is single-click — the other slot keeps its current value.

The pending state is never cleared on POST dispatch. It persists until the server confirms the change via a `comparison-changed` event. This prevents a race where the user clicks, the POST fails silently, and the UI shows the wrong state.
