---
layout: post
title: "DraftHouse — The Single Entry Point That Wasn't"
date: 2026-06-24
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-drafthouse]
tags: [eidos, refactor, mcp-tools]
---

# DraftHouse — The Single Entry Point That Wasn't

**Date:** 2026-06-24
**Type:** phase-update

---

## What I was trying to achieve: one resolution path for reviewer identity

After #62 landed the Eidos agent model for debate channels, DraftHouse had two mechanisms for the same domain operation — resolving who reviews a document. Debate channels went through `AgentRegistry` and `SystemPromptRenderer`. Review channels read a flat string from `application.properties`. Two resolution paths for one concept is the kind of debt that quietly compounds.

## What we believed going in: a straightforward config swap

The issue was scoped as S/Low — swap the config source, wire in the registry, done. The interesting part turned out to be not the swap itself but the questions it forced about where boundaries should sit.

## The boundary questions the swap exposed

The first question was whether `DocumentReviewer` — the LangChain4j `@RegisterAiService` interface — should keep its `@SystemMessage("{{personality}}")` passthrough template. The rendered instructions from `SystemPromptRenderer` produce a complete system prompt: identity, disposition axes, briefing, goal context. Passing that as a single template variable is correct — the rendered output IS the system message.

But looking at the `@UserMessage` template revealed something else. The response protocol — the AGREE/QUALIFY/DECLINE outcome semantics — was sitting in the user message alongside document content and query text. Those outcome instructions are behavioural guidance constant across every call. They belong in the system message with the agent's identity, not repeated alongside each query. Moving them cleaned up a concern boundary that had been wrong from the start.

The second question was whether `list_reviewers` and `get_reviewer_instructions` should stay on `DebateMcpTools`. Both are agent infrastructure that applies to any channel using the reviewer registry — not debate-specific operations. After migration, `start_review` references `list_reviewers` in its tool description. Having that tool on a different class is organisationally incoherent. We moved both to `DraftHouseMcpTools` and generalised `get_reviewer_instructions` to take a resource path directly instead of requiring a debate session ID.

The third question was the one that nearly slipped through: whether `ReviewerResolver` was actually a single entry point. The spec claimed it was, but `DebateMcpTools` still had a direct `@Inject AgentRegistry` for lightweight name lookups in `getDebateSummary` and `exportDebateSummary`. A code review caught this — one `findDescriptor()` method on the resolver centralised the tenancy-ID concern and eliminated the direct registry injection entirely.

## The snapshot question

The out-of-scope rationale for `DebateSession` not adopting `ResolvedReviewer` needed correcting too. I'd written "different lifecycle model — mutable class vs immutable record", but that's wrong. `DebateSession`'s identity fields (`channelId`, `agentId`, `channelName`) are all `private final` — the mutability is confined to participants and selection. The real reason is semantic: debate sessions are long-lived, and `restart_from_round` deliberately re-resolves via the registry to pick up descriptor changes. Storing a snapshot would either go stale or need invalidation logic. Review sessions are short-lived — the snapshot is safe.

## Where this leaves the tool surface

`DraftHouseMcpTools` now owns all cross-channel agent infrastructure: `start_review`, `list_reviewers`, `get_reviewer_instructions`. `DebateMcpTools` is purely debate mechanics. `ReviewerResolver` is the single entry point — resolve, list, and find all go through it, and the tenancy ID is referenced in exactly one place.

The flat `casehub.drafthouse.reviewer.personality` config key and `DraftHouseConfig.Reviewer.personality()` are gone. `DraftHouseConfig.Reviewer` retains only `maxDocChars()` — an operational limit, not an identity concern.
