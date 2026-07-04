---
title: "Killing the Polling Loop"
date: 2026-07-04
sequence: 24
author: mdp
projects: [casehub-drafthouse]
tags: [websocket, pages, real-time, architecture]
---

# Killing the Polling Loop

DraftHouse had a latency problem hiding behind two layers of polling. When an MCP client started a debate, the browser took up to five seconds to notice — a `setInterval` polling for active sessions every five seconds. Once connected, every debate event still had up to 500ms of latency from a Mutiny `ticks()` loop on the server, polling Qhorus for new messages.

The thing is, Qhorus already pushes. `DebateChannelBackend.post()` gets called the instant a message is dispatched to the debate channel. But it was ignoring everything except `SUB_TASK_REQUEST` — filtering for agent dispatch and dropping the actual debate messages on the floor. Meanwhile, the SSE endpoint independently polled the same message store at 500ms intervals to find them.

Four `ConcurrentHashMap` queues buffered metadata events the same way. Context-usage, documents-changed, comparison-changed, selection-scope — each arrived immediately from REST endpoints or MCP tools but sat in a single-slot buffer until the next tick drained it. The events were already there. They just weren't going anywhere.

## The Fix

A single WebSocket connection replaces all of it. `WebSocketEventBus` is a CDI singleton that routes events from three producer types — ChannelBackend messages, metadata pushes, file changes — to the right WebSocket connections. `DebateWebSocket` at `/api/ws` handles subscribe/unsubscribe, catch-up delivery on connect, and file watch lifecycle.

The event chain is now: Qhorus dispatches message → `DebateChannelBackend.post()` → `WebSocketEventBus` → browser panel. Zero polling. The five-second discovery lag is gone — session lifecycle events broadcast to all connections the instant a session is created or ended.

On the browser side, the pages data pipeline handles the WebSocket connection — `createWebSocketSource` gives us reconnection with exponential backoff out of the box. The server speaks the pages wire format (`{ "op": "event", "topic": "...", "payload": ... }`), and `processWireMessage()` dispatches `pages-event` CustomEvents automatically. The panels didn't change at all. They were already subscribing to `pages-event` topics. The transport beneath them is invisible.

## The pages Gap That Shaped the Design

I initially assumed pages' `since`/`lastSeq` mechanism would give us incremental recovery on reconnect — only resend events the client missed. It doesn't. `processWireMessage()` returns early on `event` ops without calling `updateSeq()`. The sequence tracking only applies to dataset ops (`snapshot`, `append`, `replace`, `remove`). For event-only consumers, `lastSeq` is never set, and `since` is never sent on reconnect.

So reconnection uses a simpler model: the server pushes a `reconnected` event on every WebSocket open, panels reset their state, and the server sends a full catch-up. For typical debate sessions — twenty to fifty messages — it's negligible bandwidth and avoids a false assumption about incremental recovery.

I filed an epic on casehub-pages for the infrastructure improvements this surfaced: event topic subscriptions (listen/unlisten), a lightweight event connection API, and server-side Java push protocol types. When those land, the subscribe-as-signal workaround becomes a clean API call. The architecture doesn't change.

## What Got Deleted

`sse-bridge.ts`, `WatchResource.java`, the SSE endpoint, the 500ms polling loop, four ConcurrentHashMap queues, per-file EventSource connections, and session discovery polling. About 800 lines removed against 1400 added — but the additions include the WebSocket endpoint, the event bus, two test classes, and a design spec that went through four rounds of adversarial review.

The diff panel's file-change notifications also moved to WebSocket. It used to create a separate `EventSource` per loaded file, each hitting `/api/watch?path=`. Now file changes push through the same WebSocket connection as everything else.
