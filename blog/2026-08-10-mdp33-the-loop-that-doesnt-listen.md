---
layout: post
title: "The Loop That Doesn't Listen"
date: 2026-08-10
entry_type: note
subtype: diary
projects: [casehub-drafthouse]
tags: [orchestration, autonomous, virtual-thread, termination]
series: issue-71-autonomous-triggering
---

The orchestrator was wired up — `DebateAgentInvoker`, `DebatePromptAssembler`, `DebateResponseBuilder`, the autonomous flag, everything constructed and stored on the session. But nothing started it. `converse()` sat there, fully assembled, waiting for a trigger that didn't exist.

The trigger belongs in `DebateChannelBackend.post()`. Every debate message on the channel passes through this method via Qhorus gateway fan-out — it already pushes WebSocket events and dispatches sub-agent requests via CDI. Adding the autonomous trigger here means any message source can kick off the orchestrator, not just the `raise_point` MCP tool.

The interesting part is the isolation model. `ConversationOrchestrator.converse()` runs an internal `ArrayDeque` — it seeds the queue with the triggering message, invokes agents, builds responses, dispatches them to the channel, and enqueues the responses for the next iteration. It's a closed loop. Messages that arrive on the channel from outside — a human clicking FLAG_HUMAN in the UI, someone calling `raise_point` manually — go through `messageService.dispatch()` and reach `DebateChannelBackend.post()`, but they never enter the orchestrator's queue. The dispatcher is output-only: responses go out to the channel, nothing comes back in.

This means external interruption can't work by posting to the channel. You need `orchestrator.terminate()`, which sets a volatile boolean the loop checks each iteration. I wired FLAG_HUMAN detection in `post()` — when a FLAG_HUMAN arrives on an autonomous session, it calls `terminate()` directly. Same for `endDebate()`.

The exactly-once trigger uses an `AtomicBoolean` on `DebateSession` — `markConverseStarted()` is a single `compareAndSet(false, true)`. Multiple messages can arrive in the window between the first dispatch and the virtual thread starting; only the first wins the CAS race. The virtual thread itself is straightforward: `converse()` returns `Uni<ConversationOutcome>` but the Uni wraps a synchronous loop, so `.await().indefinitely()` on a virtual thread is the right subscription model.

`ContestedEscalation` was the other discovery worth noting. The name suggests it handles FLAG_HUMAN escalation — the spec originally assumed this. It doesn't. It checks for points in DISPUTED status that have been disputed more than N times. It's about agents that can't agree, not about human intervention. Two different termination paths, easily conflated.
