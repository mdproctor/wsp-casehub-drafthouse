---
layout: post
title: "The spec that argued itself into a simpler design"
date: 2026-06-11
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-drafthouse]
tags: [debate, sse, mutiny, spec-review, cursor-polling]
---

I went into this thinking push was the right architecture. A `ChannelBackend` observer catches every debate message, emits it to an in-process event bus, and the SSE endpoint subscribes. Clean separation. Claudony has exactly this pattern — `ClaudonyChannelBackend` → `ChannelEventBus` → `MeshResource`. So I designed DraftHouse's version: five new components, a `DebateObserverBackend`, a `DebateEventBus`, a transport DTO, an SSE resource, and browser wiring. The spec came out in one pass.

Then we reviewed it.

The first round caught twelve things. Most were quality — use the existing `EntryType` enum instead of raw strings, put the transport DTO in `runtime/` not `api/`, self-register the observer backend instead of coupling through the factory. One was structural: the catch-up → live gap. I'd written "subscribe to the bus before running catch-up, then concatenate" — which sounds correct until you look at how Mutiny actually works.

`Multi.createFrom().emitter()` is lazy. The callback that registers the subscriber runs at subscription time, not at Multi creation time. And `Multi.createBy().concatenating().streams(catchUp, live)` subscribes to `live` only after `catchUp` completes. So "subscribe first" doesn't actually subscribe. The bus has no subscriber during catch-up. Any message dispatched in that window is in neither stream — silently dropped, no error, no warning.

I'd proposed the fix before understanding the problem. "Subscribe first, catch up second, no dedup needed" — the spec said confidently. The second review dismantled it. The emitter callback doesn't run until Mutiny subscribes to the Multi, which doesn't happen until the concatenation advances past the first stream. The Multi object is a blueprint, not an active subscription. My fix was fighting the framework.

The same review surfaced the second problem: Claudony had already built and abandoned the bus-push pattern for SSE. The comment in `MeshResource.java` is explicit — cross-thread emit from `ChannelGateway.fanOut()` virtual threads to the SSE response thread caused frames not to be flushed reliably. They switched to 500ms cursor-based polling and never looked back.

Two independent failures of the same architecture, one theoretical (Mutiny lazy subscription) and one empirical (cross-thread flushing in production). The design I'd started with was wrong in two ways I hadn't considered. The fix wasn't to patch the bus-push design — it was to abandon it.

The third spec revision dropped three of the five components. No `DebateEventBus`. No `DebateObserverBackend`. No separate mapper class. What remained: `DebateStreamEntry` (transport DTO), `DebateEventResource` (SSE endpoint with cursor-based polling), and browser wiring. The endpoint polls `MessageService.pollAfter()` every 500ms with an `AtomicLong` cursor that carries forward from catch-up to live. No gap. No cross-thread issue. Proven in Claudony.

Two more review rounds refined the edges. `RESTART_CONTEXT` was living as a string-matched special case in `DebateChannelProjection.apply()` — it deserved to be in the `EntryType` enum, with both exhaustive switches updated. The `correlationId` field was doing double duty: it carried `pointId` for debate entries but `subTaskId` for sub-task entries. We split it into two explicit fields with a mapping table per entry type. `inReplyTo` (a Qhorus message id the browser can't resolve) was dropped. `SummaryRenderer`'s exhaustive switch needed the same `RESTART_CONTEXT` treatment. `dispatchError()` doesn't propagate `pointId` in its META header, so `SUB_TASK_ERROR` gets its own table row with `pointId = null`.

Implementation was six tasks. The `RESTART_CONTEXT` enum promotion. The `activeSessions()` interface addition. The `DebateStreamEntry` record with its `from(Message)` factory and eleven unit tests. The SSE endpoint with integration tests. Browser `EventSource` wiring. Doc updates.

The Mutiny lazy-subscription gotcha went into the garden as `GE-20260612-fa0894`. It scored 14/15 — the mental model "I created the Multi, so it's subscribed" is natural, wrong, and invisible until you trace the emitter callback.

Four review rounds, one architectural pivot, and the result is half the components I started with. The spec argued itself into a simpler design — not because simpler is automatically better, but because the complex design had two independent correctness failures that the simple one doesn't have.
