---
layout: post
title: "Channel Projection and the Memory We Lost"
date: 2026-06-07
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-drafthouse]
tags: [qhorus, channel-projection, conversation-memory, domain-model]
---

The starting point was clear on paper: DraftHouse still had a local file-based debate pipeline wired up beside the Qhorus channel system. `SummaryProjector`, `DebateParser`, `ReviewSessionService` — none of them reachable from the live path. Delete the dead code, promote `SummaryProjector` to a registered `RenderableProjection`, close the issue.

It was only when I started pulling the thread that the real problem surfaced.

The old file-based pipeline had one capability the new Q&A backend quietly dropped: it passed accumulated conversation history to the AI on every round. `reviewerService.review(ctx.specContent(), ctx.debateContent(), ...)` — `debateContent` was the full running log. The Qhorus backend call strips that to five parameters: personality, docA, docB, selection context, question. No history. Each question lands in a fresh context with no memory of what was already asked.

That's a regression. I hadn't noticed it because the individual responses still looked reasonable. But ask the same reviewer two questions in a row and the second has no idea what the first established.

That changed the scope. Wiring `ProjectionService` into `ReviewerChannelBackend` — projecting channel history before each LLM call and passing the rendered result as conversation context — became the actual deliverable. The `RenderableProjection` registration was the easy half.

Then the domain model needed fixing. `MessageType.DECLINE` from the AI means "this query is out of scope." The original `SummaryProjector` mapped it to `EntryType.DISPUTE / ReviewStatus.ACTIVE` — "ongoing debate." Those aren't the same thing. An AI declining to answer is a closed state, not an unresolved one. We added `EntryType.DECLINED` and `ReviewStatus.DECLINED` to the domain model as first-class concepts.

The renderer split turned out to be necessary too. The existing `SummaryRenderer` produces human-readable markdown: emoji status markers, timestamps, P-priority and scope metadata, strikethrough for resolved points. None of that is useful to an LLM reading its own conversation history. What the LLM needs is a plain transcript — `Q: ...\nA: ...` — without decoration. So we created `ReviewConversationRenderer` alongside `SummaryRenderer`, with an explicit purpose per consumer: `SummaryRenderer` for the `project_channel` MCP tool output, `ReviewConversationRenderer` for the LLM's context window.

The `DebateChannelProjection` itself is simpler than I expected. All the fold logic from `SummaryProjector` carries over directly — the logic for RAISE, RESPONSE, HANDOFF was already correct for `MessageView` inputs. The key design decision was `actorType` not sender strings. The old code matched against literal `"REV"` and `"IMP"` senders, which worked when those were the actual sender names. In the live system senders are `"drafthouse-reviewer-{uuid}"` — an exact string match fails silently. `MessageView.actorType()` (HUMAN/AGENT/SYSTEM) is set explicitly at every dispatch site and doesn't vary with session IDs.

One thing Claude flagged during the fold handler review: `handleFlagHuman` was passing `message.content()` directly to `FlagEntry` and `ThreadEntry` without a null guard, inconsistent with `handleDecline` which used `Objects.requireNonNullElse`. The asymmetry was obvious once named but I'd missed it writing the spec. An edge case that probably never fires in production, but the inconsistency is the right thing to care about.

The deleted pile was satisfying. `DebateParser`, `RoundParser`, the sealed `DebateEvent` hierarchy, `DebateAgentProvider`, the six-field `ReviewSession` record, `LangChain4jDebateAgentProvider`, `SpecReviewerAiService`, `SpecImplementerAiService`, the entire `claude-agent` module. None of it connected to anything live. The new `server/runtime/debate/` directory has exactly one file: `DebateChannelProjection.java`.

The integration test that validated the whole point: send a first question, wait for the response, send a second question, capture the `reviewHistory` argument on the second LLM call. Confirm it contains the first question and its answer. Confirm it doesn't contain the second question — which is still OPEN at projection time, so `ReviewConversationRenderer` excludes it. That test is what the migration was actually for.
