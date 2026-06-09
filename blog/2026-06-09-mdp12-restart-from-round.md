---
layout: post
title: "The Round That Knows What It Means"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [drafthouse]
tags: [qhorus, channel-projection, cdi, mcp-tools]
---

The interesting design question for `restart_from_round` wasn't the channel branching — that's just creating a new Qhorus APPEND channel and posting a provenance marker. It was whether the tool needed a `mode` parameter to distinguish "resume" from "redo."

I spent longer than I should have on this. Resume means: the LLM session expired and the agent wants to pick up where they left off. Redo means: the last two rounds went off track and the agent wants to discard them. The difference, once you stare at it long enough, is entirely expressed by which round number you pass. Pass the last round you trust and you get exactly the state you want. The mode parameter would have been documentation for a distinction the round number already makes.

The RESTART_CONTEXT provenance message is the other decision worth recording. The first instinct was to add `RESTART_CONTEXT` to `EntryType` as a new constant. But `EntryType` is in the `api` module, and `SummaryRenderer` has an exhaustive switch over every `EntryType` value — adding a new constant there would fail compilation in the renderer. The correct answer was a string check before `EntryType.valueOf()`: two lines, no enum pollution, the channel message is still DHMETA-structured and queryable. That's the version that went into the spec.

The code review caught things I'd missed. There were two real bugs:

The first: the spec proposed adding `|round=N` to `SUB_TASK_FINDING` by reading round from `AbstractDebateSubAgentHandler.buildResponse()`'s trigger meta. But `request_subagent` never encoded `round` in the `SUB_TASK_REQUEST` header in the first place. The trigger meta had no round key. The finding would have hardcoded `round=0` for every sub-task, regardless of when it was requested. Fix: add a `round` parameter to `request_subagent` and encode it in the header.

The second: `RoundBoundedProjection` had `@Override public String projectionName()`, but `projectionName()` lives in `RenderableProjection`, not in `ChannelProjection`. `RoundBoundedProjection` only implements `ChannelProjection`. That `@Override` would have been a compilation error. I'd read the wrong interface.

The `isEmpty()` issue was subtler. `ProjectionResult.isEmpty()` returns `lastMessageId == null`. What I needed it to mean was "no messages contributed to the state." Those are different things. When `RoundBoundedProjection` scans a channel with messages but skips all of them because they exceed `maxRound`, the cursor still advances — every scanned message updates `lastId`, whether the projection accepted it or not. The result has `isEmpty() == false` but a state identical to `identity()`. The fix is a domain-specific content check in the tool layer:

```java
ReviewState s = result.state();
String summary = (s.points().isEmpty() && s.memos().isEmpty() && s.subTaskFindings().isEmpty())
    ? "No debate activity up to round " + round + "."
    : renderer.render(s);
```

That's in `renderBounded()` now. It's the right place — `SummaryRenderer` renders a non-empty state; deciding what counts as empty is a caller concern.

The CDI fix (#44) was the oldest debt on the list. Quarkus augmentation was failing because `JpaMessageStore` in Qhorus injects `CurrentPrincipal`, and `MockCurrentPrincipal @DefaultBean` lives in `casehub-platform`. Library modules declare it with `test` scope — correct, because it's invisible to production augmentation which is the goal. Application modules with a `quarkus:build` goal need `runtime` scope. Drafthouse had neither. One dependency line, issue closed.

The `SubAgentE2ETest` turned out to be the most satisfying part. The async dispatch chain — `request_subagent` → Qhorus message → `ChannelGateway.fanOut()` → `DebateChannelBackend.post()` → `Event<ChannelAgentRequest>.fireAsync()` → `ChannelAgentDispatcher` → `MockDebateAgentProvider` — all ran correctly in the `@QuarkusTest` container on the first pass. `MockDebateAgentProvider` returns "Mock sub-agent finding." synchronously, so the only thing Awaitility actually has to wait for is the CDI event dispatch latency. It was under 100ms in every run.
