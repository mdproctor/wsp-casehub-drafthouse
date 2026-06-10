---
layout: post
title: "The spec review that saved the implementation"
date: 2026-06-10
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-drafthouse]
tags: [debate, n-participant, spec-review, fold-safety, quarkus]
---

The quality batch came first — five issues, one branch, sequential. Most of it was mechanical: rename a misleading field here, fix a JSON escaping bug there, drop a `throws` declaration that was actively lying to callers. Issue #48 had one genuine bug buried in the batch: `start_debate` was building its JSON response by hand and wrapping `specPath` in raw quotes, so a spec file path with a `"` in it would produce malformed JSON. The `jsonString()` helper for exactly this existed three methods away. Nobody had noticed because no test used a path with quotes. I wrote the test first, confirmed the failure, then fixed it in one line.

Then #41 arrived. I'd filed it months back as "threshold-based auto-reset safety valve + context meter UI" — a fallback for long debate sessions where sub-agent spawning might not be enough. But when I started the brainstorm, the conversation expanded. The diff viewer was the whole UI. The debate system was purely MCP. Nothing linked them. That needs to be fixed before a context meter makes sense — and if we're going to build a channel viewer anyway, it should be modular enough for Claudony to embed later. So #41 became four sub-projects, and we designed the first one: N-participant sessions.

The two-party constraint was entirely in the application layer. Qhorus channels are N-party by nature. The fix was local: extend `AgentType`, convert `DebateSession` from an immutable record to a class with a `ConcurrentHashMap<AgentType, String>` participants map, update `DebateMcpTools` to validate roles via `AgentType.valueOf()` instead of hardcoded string checks. I wrote the spec in one pass, then put it up for review.

The first review caught two real problems I'd missed. The original spec said `DebateMcpTools.sender()` would call `session.participants().computeIfAbsent(role, fn)`. The reviewer flagged it immediately: `session.participants()` returns `Collections.unmodifiableMap(map)` — calling `computeIfAbsent` on the view throws `UnsupportedOperationException`. The fix is to encapsulate the mutation inside `DebateSession` with a `registerIfAbsent(role, Supplier)` method that calls `computeIfAbsent` directly on the underlying `ConcurrentHashMap`. It's obvious once you see it, but nothing in the API surface tells you the view is mutation-blocking until you run into the exception.

The same review caught that I'd listed `DebateChannelProjection.agentType()` as "no changes needed." The projection's private helper was a hardcoded switch over `"REV"` and `"IMP"` that threw `IllegalArgumentException` for anything else. The moment a SUPERVISOR posted, the fold would crash.

I pushed back on the severity: throw vs log-and-discard, I argued, was a design choice — throwing makes sense when the producer is our own code and an unknown value signals a bug. The second review settled it definitively. We decompiled `ProjectionService` from the jar and read `fold()` directly:

```java
for (Message msg : messages) {
    state = projection.apply(state, mapper.toMessageView(msg));
    lastId = msg.id;
}
return new ProjectionResult(state, lastId);
```

No try-catch. An exception thrown from `apply()` terminates the fold immediately, discards everything after it, and propagates up through the MCP tool. The session is permanently broken until the server restarts. That's not a design tradeoff — that's a fold invariant. `apply()` must return a state for any input, and the right response to a malformed message is a warning log and `return state`. We discarded, logged at ERROR for the null-agent case and WARNING for the unknown-agent case, and filed a protocol: `ChannelProjection.apply()` must never throw.

The second review also caught that my spec said "five direct call sites" for the null check. There were three — `handleRaise`, `appendToPoint`, `handleFlagHuman`. I'd confused direct calls to `agentType()` with the four methods that call `appendToPoint`. A spec error that would have sent an implementor looking for two call sites that don't exist.

Three review passes, nine commits. The spec exercise found the unmodifiable-map trap, the fold invariant, and the call-site count — all before a line of production code was written. Any one of those bugs surfacing in implementation would have taken longer to diagnose than the entire review process did to run.

The fold invariant is the one I'll remember. A left-fold with no exception handling sounds fine until you realise that a single bad message in a long-lived channel breaks every tool that projects that channel, forever. The Qhorus design decision to omit exception handling in `fold()` is probably correct — a fold that swallows exceptions is hiding bugs — but it places the correctness burden entirely on the implementor of `apply()`. That contract wasn't written down anywhere. It is now.
