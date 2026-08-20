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

**Choice:** Phase-filtered via platform WorkerFunctionProvider. DraftHouse registers all tools with the platform's dynamic tool discovery system, tagged by capability. Tools are registered when a capability is activated and deregistered when deactivated. LLM clients see only the tools for active capabilities (6-10 tools instead of 31).
**Alternatives:**
- Static but grouped — keep @Tool annotations, add naming prefixes (draft_*, review_*). All 31 tools visible, naming helps LLMs pick. Simple but doesn't reduce cognitive load.
- MCP notifications (tools/list_changed) — dynamically change tool list, send MCP notification. Most MCP-native but requires client support for dynamic tools.
**Rationale:** Platform already has WorkerFunctionProvider SPI with McpWorkerFunctionProvider for dynamic tool discovery. Leveraging this infrastructure means DraftHouse's tools are discoverable through the same mechanism as any other platform worker. Capability-scoped registration is natural — activate Voice, its 3 tools appear; deactivate, they disappear.
**Trade-offs:** Requires DraftHouse to integrate with platform's worker discovery rather than using Quarkus MCP's @Tool annotations directly. More setup, but aligns with platform direction.
**Sources:** Platform codebase: McpWorkerFunctionProvider, WorkerFunctionProviderRegistry, McpEndpointRegistry, McpClientRegistry, Capability record
**Exploration:** quick
**Status:** captured
