---
layout: post
title: "The Personality That Wasn't"
date: 2026-06-23
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-drafthouse]
tags: [eidos, agent-identity, review, spi-design]
series: issue-62-multi-llm-reviewers
---

*Part of a series on [#62 — Multi-LLM reviewers with personality library](https://github.com/casehubio/drafthouse/issues/62). Previous: [Four Layers of CI Rot](2026-06-20-mdp16-four-layers-of-ci-rot.md).*

## The Personality That Wasn't

I started with what seemed reasonable: a `PersonalityProvider` SPI and a `PersonalityDocument` record. Key, name, perspective, instructions. Four fields. Clean contract. DraftHouse defines it, Eidos implements it, CDI displaces the mock. Done.

Five spec revisions later, the SPI was gone and DraftHouse was consuming Eidos's existing `AgentRegistry` and `SystemPromptRenderer` directly. The four-field record had been a degenerate projection of `AgentDescriptor` — a structured identity model with disposition axes, capability declarations, vocabulary grounding, and a render pipeline that already produced system prompts. I'd been about to build a parallel concept for something that already existed.

## What the Reviews Caught

The first review identified the core problem: Eidos already models agent identity at a structural level. Creating `PersonalityDocument` would mean two vocabularies for the same thing — is a reviewer an `AgentDescriptor` or a `PersonalityDocument`? The flat `instructions` string throws away the structured disposition axes (conflictMode, ruleFollowing, riskAppetite) that give composable reviewer behaviour.

The second catch was subtler and more important. I'd designed the personality to flow through `AgentTask.systemPrompt()` — the server-side LLM invocation path. But the main reviewer agents are the MCP callers themselves. Claude Code calls `start_debate`, then `raise_point`, then `respond_to`. The server never invokes the LLM for the main reviewer — it only does that for sub-agents (verify, arbitrate). The personality instructions needed to go *back to the caller* in the `start_debate` response, not be stored server-side.

Later rounds caught the seeding gap (when `JpaAgentRegistry` displaces the mock, the four reviewer descriptors vanish unless a separate `@Startup` bean registers them), the non-deterministic default (picking "first from `ConcurrentHashMap`" is not deterministic), and the `MAX_BRIEFING = 500` character limit that would have thrown `AgentValidationException` on any briefing longer than three sentences.

## What We Built

Four reviewer personalities — structural, content, readability, completeness — each an `AgentDescriptor` with distinct disposition axes and a briefing that defines their review focus. A `@DefaultBean` in-memory `AgentRegistry` and `SimplePromptRenderer` for standalone operation, displaced by Eidos's real implementations via CDI when on the classpath.

`start_debate` resolves a reviewer from the registry, renders instructions through `SystemPromptRenderer`, and returns them in the response. The caller adopts the instructions as its persona for the session. `get_debate_summary` was restructured from raw markdown to JSON — a breaking change, but the structured format carries reviewer metadata that the old format couldn't.

Three new MCP tools: `list_reviewers` (structured discovery with disposition axes and capability tags), `get_reviewer_instructions` (session recovery — re-render after a server restart), and the `agentId` parameter on `start_debate`. Each session stores only the `agentId` for provenance — the rendered instructions are ephemeral, returned to the caller and never persisted.

## The Pipeline Shape

The design philosophy that emerged is one-reviewer-per-session. Focused reviews produce better results than broad ones. Multiple perspectives come from sequential sessions — a review pipeline where each stage's output feeds the next. We didn't build the pipeline (that's #72), but the session-per-reviewer model and the document working set already support it. `list_reviewers` gives a future orchestrator structured disposition data to select reviewers programmatically.

The Eidos alignment pays off beyond this feature. When `EidosSystemPromptRenderer` ships, reviewer prompts gain vocabulary-resolved labels and LLM-driven semantic enrichment — quality improves without touching DraftHouse code. The disposition axes open a path to intent-driven reviewer selection ("I want an adversarial review" → pick a reviewer with `conflictMode=competing`).
