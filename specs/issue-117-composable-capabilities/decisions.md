# Decisions — Composable Capability Architecture (#117)

## D1: Session architecture model

**Choice:** Composable capability model — a thin `DraftHouseSession` container with independently activatable capabilities, replacing the current three separate session types (ReviewSession, DebateSession, BrainstormSession).
**Alternatives:**
- Phase state machine — single session with exclusive phases (BRAINSTORM | DRAFT | REVIEW | REVISE), one active at a time. Simpler model but fights composability — can't review while drafting.
- Single flat session — god object with all fields from all modes, phase determines which are active. Simplest mental model but violates SRP and makes invariants hard to enforce.
- Event-sourced session — session as event stream, state derived from projections. Most flexible but massive architectural shift for current scale.
**Rationale:** Composability is the stated architectural direction. Capabilities must mix freely (draft + review simultaneously, voice during any mode). Existing session classes are preserved as internal capability state — minimal rewrite. The container is thin: identity, DocumentSet, working directory, event bus.
**Trade-offs:** Two layers of abstraction (container + capability). Capability interaction contracts must be explicitly designed. More interfaces than a flat model.
**Sources:** Existing codebase (DebateSession.java, BrainstormSession.java, ReviewSession.java), exploration of current mode structure
**Exploration:** deep-analysis
**Status:** captured

## D2: Cross-capability interaction model

**Choice:** Artifacts as the integration API — capabilities communicate through files in a shared working directory. No direct references, no event subscriptions between capabilities. Three integration layers: (1) artifacts for data flow, (2) session-level DocumentSet for document identity, (3) WebSocket events for UI panel refresh only.
**Alternatives:**
- Event bus between capabilities — typed events, subscribe/react. Introduces coupling through event type definitions and ordering concerns.
- Mediator pattern — session routes commands between capabilities. Centralises routing logic, becomes a god object over time.
- Direct references — capabilities call each other's methods. Violates composability entirely.
**Rationale:** File-based integration gives zero coupling between capabilities. Each capability can be tested in isolation with fixture files. Adding new capabilities means declaring artifact inputs/outputs — nothing else changes. Matches the pipeline model (each stage is a file, diffable, hand-editable). Stale input detection is simple timestamp comparison.
**Trade-offs:** Not suitable for real-time inter-capability communication — but live interactions stay internal to their capability (debate rounds inside Review, option state machine inside Brainstorm). Only committed outputs cross boundaries.
**Sources:** Unix philosophy (programs communicate through files), existing pipeline model discussion
**Exploration:** deep-analysis
**Status:** captured

## D3: Voice-to-document pipeline architecture

**Choice:** Immutable pipeline with explicit re-run. Raw transcript → LLM cleanup (filler removal, punctuation, grammar) → draft generation → reviewed draft. Each stage produces a file. User can see diffs between stages, hand-edit any intermediate, and explicitly trigger downstream regeneration. LLM handles cleanup (not a separate model) because it has document context.
**Alternatives:**
- CrisperWhisper 2.0 for verbatim + intended transcription — highest quality filler detection but requires Python sidecar, adds model management complexity.
- Auto-triggering pipeline — saving an edit auto-re-runs downstream. Faster iteration but risks overwriting manual edits in later stages.
- Replacement notes model — each voice note replaces the previous. Clean but requires re-stating everything each time.
**Rationale:** The LLM already has document context — it knows the style, what's been written, what the document is about. A generic cleanup model doesn't. Explicit re-run prevents surprise overwrites. Appending notes model lets the user build up instructions incrementally across a session.
**Trade-offs:** LLM cleanup is not as specialised as CrisperWhisper for filler detection, but the context-awareness compensates. Explicit re-run is slower than auto but predictable.
**Sources:** Web research on Whisper Java bindings, CrisperWhisper, erm CLI, LanguageTool
**Exploration:** deep-analysis
**Status:** captured

## D4: Speech-to-text integration

**Choice:** Local Whisper via whisper.cpp + Quarkus FFM (Panama Foreign Function API). DraftHouse owns the STT pipeline. Three voice input modes: push-to-talk, continuous with pauses, record-then-review. Browser captures audio via MediaRecorder, streams to Quarkus server, server transcribes via whisper.cpp with Metal acceleration on Apple Silicon.
**Alternatives:**
- whisper-jni — JNI wrapper, Maven Central, but ships Linux/Win binaries only; needs custom macOS build.
- WhisperJET — pure Java, no native bridge. Avoids all native complexity but CPU-only, slower.
- Browser Web Speech API — free, built-in, but quality varies by browser (Chrome-only realistically).
- Cloud STT service (Deepgram, AssemblyAI) — highest quality but adds dependency, cost, network requirement.
**Rationale:** Already on Quarkus 3.34 with Java 21+. Panama FFM is the modern way to call native code — no JNI framework overhead. whisper.cpp with Metal gives native Apple Silicon acceleration. Jan 2026 tutorial shows this exact stack (Quarkus + FFM + whisper.cpp). No external dependencies.
**Trade-offs:** Requires jextract tooling setup. Native library build step. Only works on platforms where whisper.cpp builds (covers all our targets). Model download (~1-3GB for large-v3-turbo).
**Sources:** [Quarkus + FFM tutorial](https://www.the-main-thread.com/p/java-speech-to-text-quarkus-whisper-ffm), [whisper-jni](https://github.com/GiviMAD/whisper-jni), [whisper.cpp](https://github.com/ggml-org/whisper.cpp), [WhisperJET](https://github.com/eix128/WhisperJET), [Metal benchmarks](https://www.promptquorum.com/local-llms/apple-silicon-whisper-metal-benchmark)
**Exploration:** deep-analysis
**Status:** captured

## D5: MCP tool scoping mechanism

**Choice:** Dynamic registration via Quarkus MCP `ToolManager`. Each facet registers its tools when activated via `toolManager.newTool(name)` and deregisters via `toolManager.removeTool(name)`. Connected MCP clients receive automatic `tools/list_changed` notifications. LLM clients see only the tools for active facets (6–10 tools instead of 31).
**Alternatives:**
- Static but grouped — keep @Tool annotations, add naming prefixes (draft_*, review_*). All 31 tools visible, naming helps LLMs pick. Simple but doesn't reduce cognitive load.
- Phase-filtered via platform WorkerFunctionProvider — register tools through engine's worker dispatch SPI. Architecturally incoherent: `WorkerFunctionProvider` handles worker dispatch from case definitions (`JsonNode` → `WorkerFunction`), not MCP tool surface management. Would also introduce premature `casehub-engine-api` dependency (APPLICATIONS.md: DraftHouse depends on qhorus initially, engine added later).
**Rationale:** Quarkus MCP Server 1.11.1 (already in DraftHouse's dependency tree) provides `ToolManager` with exactly the dynamic tool lifecycle needed: `newTool()` for programmatic registration with full schema control, `removeTool()` for deregistration, and automatic `tools/list_changed` notifications to connected clients via `ToolDefinition.register()`. No new dependencies. MCP-native. Claude Code supports `tools/list_changed`.
**Trade-offs:** Requires programmatic tool registration instead of declarative `@Tool` annotations. More setup code per tool, but aligns with the composable facet model — tools belong to their facet, not to a static class.
**Sources:** Quarkus MCP Server: `io.quarkiverse.mcp.server.ToolManager`, `io.quarkiverse.mcp.server.FeatureManager.FeatureDefinition.register()` javadoc
**Exploration:** quick → revised via adversarial review
**Status:** revised (R1-02/R1-03/R1-04/R1-05: replaced phantom WorkerFunctionProvider-based mechanism with Quarkus MCP ToolManager; eliminated premature casehub-engine-api dependency)

## D6: Naming for composable session concept

**Choice:** "Facet" — `VoiceFacet`, `DraftFacet`, `ReviewFacet`, `BrainstormFacet`. The `DraftHouseSession` container holds independently activatable facets.
**Alternatives:**
- Capability — collides with `io.casehub.worker.api.Capability` (worker dispatch), `io.casehub.eidos.api.AgentCapability`/`CapabilityHealth` (agent probing), `io.casehub.work.api.Capability`, `io.casehub.model.Capability` (engine schema), `io.quarkus.deployment.Capability` (build-time detection), and `dev.langchain4j.model.chat.Capability`. Six collisions across the dependency tree.
- Mode — implies exclusivity (one mode at a time), which contradicts the composability goal. DraftHouse's design allows simultaneous voice + review + draft.
- Feature — collides with Quarkus MCP's `FeatureManager<T>` which `ToolManager` extends. Also too generic.
- Aspect — AOP connotations in the Java ecosystem would mislead.
**Rationale:** "Facet" is semantically precise — a session has multiple facets simultaneously, like a gem. No existing collision in the platform or dependency tree. Naturally conveys composability: activating a facet reveals one face of the session without hiding others.
**Sources:** Platform codebase: `io.casehub.worker.api.Capability`, `io.casehub.eidos.api.AgentCapability`, `io.casehub.work.api.Capability`, `io.casehub.model.Capability`, `io.quarkus.deployment.Capability`, `io.quarkiverse.mcp.server.FeatureManager`
**Exploration:** surfaced via adversarial review (R1-11)
**Status:** captured

## D7: Voice infrastructure scoping — DraftHouse vs casehub-blocks

**Choice:** Two-layer design: domain-agnostic STT infrastructure (whisper.cpp FFM bindings, audio capture, input modes, raw transcript management) in `casehub-blocks`; document-specific pipeline (transcript cleanup, draft generation, style adaptation) in DraftHouse.
**Alternatives:**
- DraftHouse-only — entire voice stack scoped to DraftHouse. Faster initial delivery but prevents reuse by other applications (devtown: voice PR review, clinical: voice dictation, life: voice commands).
- Full blocks extraction — entire pipeline including document-aware cleanup in casehub-blocks. Over-generalises: document-context-aware cleanup IS domain-specific.
**Rationale:** The STT pipeline (whisper.cpp + FFM + audio processing) takes audio in and produces text out — no DraftHouse-specific logic. APPLICATIONS.md boundary rules: "If a capability is useful across multiple applications, it belongs in the foundation." Voice input is clearly reusable across applications. casehub-blocks already exists as the foundation module for cross-application building blocks. Clean boundary: blocks provides `SpeechToTextService` (audio → transcript), DraftHouse consumes it for its document pipeline.
**Sources:** APPLICATIONS.md boundary rules, casehub-blocks module structure
**Exploration:** surfaced via adversarial review (R1-12)
**Status:** captured
