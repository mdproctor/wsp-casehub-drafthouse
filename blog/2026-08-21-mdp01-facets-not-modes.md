---
layout: post
title: "Facets, Not Modes — Composing DraftHouse from Independent Building Blocks"
date: 2026-08-21
entry_type: note
subtype: diary
projects: [casehub-drafthouse]
tags: [architecture, composability, voice, facets, mcp]
series: issue-117-composable-capabilities
---

DraftHouse grew three session types — ReviewSession, DebateSession, BrainstormSession — each with its own registry, its own MCP tools, and its own fixed layout. Adding a fourth (voice-first document drafting) would have meant a fourth parallel system. I wanted something that composed instead.

The core idea: modes are the wrong abstraction. A mode implies exclusivity — you're in brainstorm mode OR review mode. But real work doesn't respect those boundaries. You want to draft a section while a reviewer watches. You want to record voice notes while brainstorming. Modes fight that; composable facets allow it.

A facet is an independently activatable building block — voice capture, brainstorming, drafting, review — each with its own state, its own MCP tools, and its own UI panels. A session holds a set of active facets, not a current mode. Activate Voice and Draft together, and the LLM sees 10 tools. Activate Review alongside, and 13 more appear. Deactivate Voice, and 3 disappear. The ToolManager handles the registration lifecycle; MCP clients get `tools/list_changed` notifications automatically.

The naming took a detour. I started with "capability" — obvious choice. The adversarial review found six collisions across the platform dependency tree: `io.casehub.worker.api.Capability`, `io.casehub.eidos.api.AgentCapability`, `io.casehub.work.api.Capability`, `io.casehub.model.Capability`, `io.quarkus.deployment.Capability`, and `dev.langchain4j.model.chat.Capability`. "Facet" survived — semantically precise (a session has multiple facets simultaneously, like a gem), no existing collision, and naturally conveys composability.

The hardest design question was cross-facet interaction. When ReviewFacet accepts a finding about the draft, how does DraftFacet learn about it? Direct method calls would couple them. An event bus would create implicit dependencies through event types. We landed on something simpler: artifacts as the integration API. Facets communicate through files in a shared working directory. Voice writes `notes/accumulated.md`. Draft reads it as pipeline input. Review writes `findings/accepted.md`. Draft reads that when re-running. No facet holds a reference to another facet. The filesystem is the message bus, and the user controls when downstream stages re-run.

The voice-first drafting pipeline itself follows the same pattern. Raw audio goes to whisper.cpp via Java's Panama FFM API — Metal acceleration on Apple Silicon, no JNI framework overhead. The transcript flows through LLM cleanup (filler removal, punctuation, grammar), then into draft generation, then into revision incorporating review findings. Each stage is a file. The diff panel shows any two stages side-by-side. Hand-edit any stage, explicitly trigger "rerun from here." The LLM handles cleanup rather than a specialised model because it has document context — it knows the target style, what's already written, and the document's subject matter.

We built the foundation today: `Facet` interface, `ArtifactSpec` record, `DraftHouseSession` container, `DraftHouseSessionStore` SPI, registry, REST endpoints, and session-level MCP tools. The existing tests all pass — no regressions. The container is deliberately thin: identity, DocumentSet, working directory, and a ConcurrentHashMap of active facets. Everything else lives in the facets themselves, which arrive in subsequent plans.

What this opens up isn't just one new mode. It's the ability to add modes without modifying existing ones. A citation-lookup facet, a diagramming facet, a collaboration facet — each declares its artifact inputs and outputs, registers its tools on activation, and composes with whatever else is active. The architecture scales by addition, not by modification.
