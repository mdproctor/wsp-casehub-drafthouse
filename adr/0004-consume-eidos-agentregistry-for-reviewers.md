# 0004 — Consume Eidos AgentRegistry for reviewer personality resolution

Date: 2026-06-23
Status: Accepted

## Context and Problem Statement

DraftHouse needs distinct reviewer personalities for debate sessions — different system prompts for structural, content, readability, and completeness reviewers. The question is where the personality resolution model lives: a new DraftHouse-specific SPI, or the existing Eidos agent identity model.

## Decision Drivers

* ARC42STORIES.MD §3 explicitly places "Agent identity management" out of DraftHouse scope — it belongs in casehub-eidos
* Eidos already has `AgentDescriptor` with structured disposition axes, capabilities, vocabulary grounding, and `SystemPromptRenderer` that renders to system prompts
* A parallel `PersonalityDocument(key, name, perspective, instructions)` would duplicate a degenerate subset of `AgentDescriptor`
* CDI displacement pattern (`@DefaultBean` displaced by `@ApplicationScoped`) is already proven in this codebase

## Considered Options

* **Option A** — New `PersonalityProvider` SPI in `casehub-eidos-api` with `PersonalityDocument` record
* **Option B** — Consume existing `AgentRegistry` + `SystemPromptRenderer` from `casehub-eidos-api`
* **Option C** — Config-driven personalities in `application.properties`

## Decision Outcome

Chosen option: **Option B**, because Eidos already solved agent identity at a structural level and inventing a parallel concept creates two competing models for the same thing.

### Positive Consequences

* No new SPI types — zero concepts to maintain
* Structured disposition axes (conflictMode, ruleFollowing, etc.) give composable reviewer behaviour that a flat string cannot
* When Eidos ships `EidosSystemPromptRenderer`, reviewers get vocabulary-resolved labels and LLM-driven semantic enrichment automatically
* Platform coherence — one agent identity model across CaseHub

### Negative Consequences / Tradeoffs

* DraftHouse takes a compile dependency on `casehub-eidos-api` (0.2-SNAPSHOT)
* `SimplePromptRenderer` mock is less capable than `EidosSystemPromptRenderer` — no vocabulary resolution, no enrichment
* CDI displacement of `EidosSystemPromptRenderer` requires an annotation change in Eidos (casehubio/eidos#64)

## Pros and Cons of the Options

### Option A — New PersonalityProvider SPI

* ✅ No dependency on Eidos API
* ✅ Simpler contract for DraftHouse's immediate needs
* ❌ Duplicates `AgentDescriptor` as `PersonalityDocument` — two vocabularies for agent identity
* ❌ Flat `instructions` string discards structured disposition axes
* ❌ Violates ARC42STORIES.MD §3 scope boundary

### Option B — Consume AgentRegistry + SystemPromptRenderer

* ✅ Zero new types — uses what exists
* ✅ Structured identity with disposition axes, capabilities, vocabulary grounding
* ✅ Platform-coherent — one model across CaseHub
* ❌ Compile dependency on eidos-api
* ❌ CDI displacement coordination needed with Eidos

### Option C — Config-driven personalities

* ✅ Zero dependencies, simplest implementation
* ❌ Config strings are poor for multi-paragraph instruction documents
* ❌ No CDI displacement path for Eidos
* ❌ Architecturally dead-end — no structured identity, no discoverability

## Links

* casehubio/drafthouse#62 — Multi-LLM reviewers with personality library
* casehubio/eidos#64 — Register reviewer descriptors, validate renderer output, CDI fix
* casehubio/drafthouse#73 — Review channel personality unification (follow-on)
* Design spec: `docs/superpowers/specs/2026-06-23-multi-llm-reviewers-design.md`
