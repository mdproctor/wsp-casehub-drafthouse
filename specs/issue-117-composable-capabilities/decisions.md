# Decisions — Composable Capability Architecture (#117)

## D1: Session architecture model

**Choice:** Composable facet model — a thin `DraftHouseSession` container with independently activatable facets (see D6 for naming rationale), replacing the current three separate session types (ReviewSession, DebateSession, BrainstormSession).
**Alternatives:**
- Phase state machine — single session with exclusive phases (BRAINSTORM | DRAFT | REVIEW | REVISE), one active at a time. Simpler model but fights composability — can't review while drafting.
- Single flat session — god object with all fields from all modes, phase determines which are active. Simplest mental model but violates SRP and makes invariants hard to enforce.
- Event-sourced session — session as event stream, state derived from projections. Most flexible but massive architectural shift for current scale.
**Rationale:** Composability is the stated architectural direction. Facets must mix freely (draft + review simultaneously, voice during any mode). Existing session classes are preserved as internal facet state — minimal rewrite. The container is thin: identity, DocumentSet, working directory, event bus.
**Trade-offs:** Two layers of abstraction (container + facet). Facet interaction contracts must be explicitly designed. More interfaces than a flat model.
**Sources:** Existing codebase (DebateSession.java, BrainstormSession.java, ReviewSession.java), exploration of current mode structure
**Exploration:** deep-analysis
**Status:** revised (R2-03: updated "capability" → "facet" terminology per D6)

## D2: Cross-facet interaction model

**Choice:** Artifacts as the integration API — facets communicate through files in a shared working directory. No direct references, no event subscriptions between facets. Four integration layers: (1) artifacts for data flow (pipeline stages as files — diffable, hand-editable, persist across session restarts), (2) session-level DocumentSet for document identity, (3) Qhorus channel messages with `ArtefactRef` for facet-to-facet notification of artifact availability, (4) WebSocket events for UI panel refresh.
**Alternatives:**
- Event bus between facets — typed events, subscribe/react. Introduces coupling through event type definitions and ordering concerns.
- Mediator pattern — session routes commands between facets. Centralises routing logic, becomes a god object over time.
- Direct references — facets call each other's methods. Violates composability entirely.
- Qhorus channels as sole integration layer — use channel messages for both coordination and data flow. Fails for pipeline data: channel messages are communication records (speech acts, deliberation), not diffable/editable pipeline artifacts. `ArtefactRef` references artifacts, it doesn't replace them.
**Rationale:** File-based integration gives zero coupling between facets. Each facet can be tested in isolation with fixture files. Adding new facets means declaring artifact inputs/outputs — nothing else changes. Matches the pipeline model (each stage is a file, diffable, hand-editable). Stale input detection is simple timestamp comparison. Pipeline stages must persist as files independently of any communication channel — they need to survive session restarts, be hand-editable, and support diff-based review. Qhorus `ArtefactRef` provides the complementary notification path: when a facet produces an artifact, it posts a channel message with an `ArtefactRef` pointing to the output file, signalling availability to other facets without conflating data flow with communication.
**Trade-offs:** Not suitable for real-time inter-facet communication — but live interactions stay internal to their facet (debate rounds inside Review, option state machine inside Brainstorm). Only committed outputs cross boundaries.
**Sources:** Unix philosophy (programs communicate through files), existing pipeline model discussion, Qhorus `ArtefactRef` record (`io.casehub.qhorus.api.message.ArtefactRef`)
**Exploration:** deep-analysis
**Status:** revised (R2-06: made file-vs-channel justification explicit, added ArtefactRef notification as fourth integration layer, updated "capability" → "facet" per D6)

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

**Choice:** Local Whisper via whisper.cpp + Quarkus FFM (Panama Foreign Function API), implemented as `casehub-blocks-stt` (see D7 for scoping rationale). Three voice input modes: push-to-talk, continuous with pauses, record-then-review. Browser captures audio via MediaRecorder, streams to Quarkus server, server transcribes via whisper.cpp with Metal acceleration on Apple Silicon. DraftHouse consumes the `SpeechToTextService` SPI from `casehub-blocks-api`.
**Alternatives:**
- whisper-jni — JNI wrapper, Maven Central, but ships Linux/Win binaries only; needs custom macOS build.
- WhisperJET — pure Java, no native bridge. Avoids all native complexity but CPU-only, slower.
- Browser Web Speech API — free, built-in, but quality varies by browser (Chrome-only realistically).
- Cloud STT service (Deepgram, AssemblyAI) — highest quality but adds dependency, cost, network requirement.
**Rationale:** Already on Quarkus 3.34 with Java 21+. Panama FFM is the modern way to call native code — no JNI framework overhead. whisper.cpp with Metal gives native Apple Silicon acceleration. Jan 2026 tutorial shows this exact stack (Quarkus + FFM + whisper.cpp). No external dependencies. The STT infrastructure is domain-agnostic (audio in → text out), so it lives in `casehub-blocks-stt` as an optional submodule (see D7). DraftHouse consumes the `SpeechToTextService` SPI for its document-specific pipeline (D3).
**Trade-offs:** Requires jextract tooling setup. Native library build step. Only works on platforms where whisper.cpp builds (covers all our targets). Model download (~1-3GB for large-v3-turbo).
**Sources:** [Quarkus + FFM tutorial](https://www.the-main-thread.com/p/java-speech-to-text-quarkus-whisper-ffm), [whisper-jni](https://github.com/GiviMAD/whisper-jni), [whisper.cpp](https://github.com/ggml-org/whisper.cpp), [WhisperJET](https://github.com/eix128/WhisperJET), [Metal benchmarks](https://www.promptquorum.com/local-llms/apple-silicon-whisper-metal-benchmark)
**Exploration:** deep-analysis
**Status:** revised (R2-04: updated ownership from "DraftHouse owns the STT pipeline" to casehub-blocks-stt per D7)

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

**Choice:** Two-layer design with optional submodule. The `SpeechToTextService` SPI interface lives in `casehub-blocks-api` (pure Java, no native dependencies). The whisper.cpp FFM implementation lives in `casehub-blocks-stt` — an optional submodule activated by adding it as a compile dependency. DraftHouse depends on `casehub-blocks-stt`; other applications that don't need STT are unaffected. Document-specific pipeline (transcript cleanup, draft generation, style adaptation) stays in DraftHouse.
**Alternatives:**
- DraftHouse-only — entire voice stack scoped to DraftHouse. Faster initial delivery but prevents reuse by other applications (devtown: voice PR review, clinical: voice dictation, life: voice commands).
- Full blocks extraction — entire pipeline including document-aware cleanup in casehub-blocks. Over-generalises: document-context-aware cleanup IS domain-specific.
- STT in blocks-core — SPI and implementation in the main blocks module. Contaminates every blocks consumer with native whisper.cpp build requirements even if they never use STT. casehub-blocks is currently pure Java (orchestration, conversation, trust routing, summarisation, channel dispatch — zero native dependencies). Adding native deps to core violates the principle that optional heavy dependencies use optional submodules (see: casehub-engine-ai, casehub-work-ai, casehub-neocortex/memory-qdrant, casehub-qhorus-postgres-broadcaster).
**Rationale:** The STT pipeline (whisper.cpp + FFM + audio processing) takes audio in and produces text out — no DraftHouse-specific logic. APPLICATIONS.md boundary rules: "If a capability is useful across multiple applications, it belongs in the foundation." Voice input is clearly reusable across applications. casehub-blocks already exists as the foundation module for cross-application building blocks (`casehub-blocks-parent` with `blocks` and `engine-adapter` submodules). Clean boundary: `casehub-blocks-api` declares `SpeechToTextService` (audio → transcript); `casehub-blocks-stt` provides the whisper.cpp FFM implementation; DraftHouse adds `casehub-blocks-stt` as a compile dependency and consumes the SPI for its document pipeline.
**Sources:** APPLICATIONS.md boundary rules, casehub-blocks module structure (pom.xml: blocks, engine-adapter), platform optional-module pattern (casehub-engine-ai, casehub-work-ai)
**Exploration:** surfaced via adversarial review (R1-12), refined via R2-05
**Status:** revised (R2-05: added optional submodule scoping — SPI in blocks-api, implementation in blocks-stt — to prevent native dependency contamination of pure-Java consumers)
