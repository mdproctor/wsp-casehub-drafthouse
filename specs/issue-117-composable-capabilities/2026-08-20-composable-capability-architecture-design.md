# Composable Facet Architecture with Voice-First Drafting

**Issue:** casehubio/drafthouse#117
**Date:** 2026-08-20
**Decisions:** [decisions.md](decisions.md) (D1–D7)

## Problem

DraftHouse has three independent session types (ReviewSession, DebateSession, BrainstormSession) with separate registries, 31 always-registered MCP tools, and fixed UI layouts selected at URL load time. This creates friction:

- LLM clients see all 31 tools regardless of context
- No runtime mode switching — page reload required
- Review and Debate share a layout but have separate registries
- Adding a new mode (voice-first drafting) would mean a fourth parallel session type

The architecture needs to move toward composability — independent building blocks that can be mixed, layered, and extended without modifying existing code.

## Design

### 1. Unified Session Container

A thin `DraftHouseSession` replaces the three top-level session types. It owns shared state and manages independently activatable facets.

```
DraftHouseSession
  ├── id: String
  ├── created: Instant
  ├── metadata: Map<String, Object>
  ├── documentSet: DocumentSet              ← shared across all facets
  ├── workingDirectory: Path                ← artifact space
  ├── activeFacets: Map<String, Facet>
  ├── toolManager: ToolManager              ← Quarkus MCP dynamic tool registration
  └── eventBus: WebSocketEventBus           ← UI notifications
```

**Session-level MCP tools** (always available regardless of active facets):
- `add_document`, `remove_document`, `list_documents`, `set_comparison`
- `list_facets`, `activate_facet`, `deactivate_facet`
- `list_reviewers`

**Session-level REST endpoints** (always available):
- `GET /api/sessions` — active session list
- `POST /api/sessions` — create session
- `DELETE /api/sessions/{id}` — end session

### 2. Facet Interface

Each facet is a self-contained module with its own state, tools, UI panels, and artifact contract.

```java
public interface Facet {
    String name();
    void activate(DraftHouseSession session);
    void deactivate(DraftHouseSession session);
    List<ArtifactSpec> inputs();
    List<ArtifactSpec> outputs();
}
```

`activate()` registers MCP tools via `ToolManager.newTool()` and initialises facet-specific state.
`deactivate()` deregisters tools via `ToolManager.removeTool()` and cleans up.
Connected MCP clients receive automatic `tools/list_changed` notifications on each transition.

### 3. Four Facets

#### VoiceFacet

Records audio, transcribes via whisper.cpp (FFM/Panama), accumulates notes.

| Aspect | Detail |
|--------|--------|
| State | Recording state, accumulated notes, Whisper model reference |
| Artifacts | Writes: `notes/{timestamp}.md`, `notes/accumulated.md` |
| MCP tools | `start_recording`, `stop_recording`, `list_voice_notes` |
| UI panels | Microphone controls (topbar widget), voice note list |
| Input modes | Push-to-talk, continuous with pauses, record-then-review |

**STT integration:** Consumes `SpeechToTextService` SPI from `casehub-blocks-api`. The whisper.cpp FFM implementation lives in `casehub-blocks-stt` (optional submodule). Browser captures audio via MediaRecorder API, streams PCM to Quarkus server endpoint, server transcribes and writes to working directory.

#### BrainstormFacet

Option exploration and selection. Wraps existing `BrainstormSession` as internal state.

| Aspect | Detail |
|--------|--------|
| State | BrainstormSession (options list, ACTIVE/CONVERGED/ABANDONED) |
| Artifacts | Writes: `brainstorm/options.json`, `brainstorm/selected.md` |
| MCP tools | `present_options`, `update_option`, `set_recommendation`, `mark_eliminated`, `mark_selected`, `end_brainstorm` |
| UI panels | `<brainstorm-options>` (existing panel, extracted to blocks-ui) |

On convergence (option selected), writes `brainstorm/selected.md` — consumed by DraftFacet as the drafting brief.

#### DraftFacet

Immutable pipeline processing, document editing, preview rendering.

| Aspect | Detail |
|--------|--------|
| State | Pipeline stages, current editor content, preview state |
| Artifacts | Reads: `notes/accumulated.md`, `brainstorm/selected.md`, `findings/accepted.md`. Writes: `pipeline/01-raw-notes.md` through `pipeline/N-*.md` |
| MCP tools | `generate_draft`, `edit_stage`, `rerun_pipeline`, `get_pipeline_status` |
| UI panels | Dual-screen: editable markdown editor (left), rendered preview (right), pipeline stage navigator |
| REST | `GET /api/session/{id}/pipeline/stages` — list stages with staleness indicators |

**Pipeline stages:**

```
notes/accumulated.md  ──read──→  pipeline/01-raw-notes.md
                                        │
                                        ↓ (LLM cleanup: filler removal, punctuation, grammar)
                                 pipeline/02-cleaned-notes.md
                                        │
                                        ↓ (LLM drafting: notes → document prose)
                                 pipeline/03-draft.md
                                        │
                                        ↓ (LLM revision: incorporate accepted findings)
                                 pipeline/04-revised-draft.md
```

Each stage is a file. The diff panel shows any two stages side-by-side. Hand-edit any stage, then explicitly trigger "rerun from here" to regenerate downstream stages. No auto-triggering.

**Staleness detection:** Each stage output records the hash of its inputs at generation time. When inputs change, the UI shows a staleness indicator — "inputs changed since last run." The user decides when to re-run.

#### ReviewFacet

Debate/critique with agents, threads, points. Wraps existing `DebateSession` as internal state.

| Aspect | Detail |
|--------|--------|
| State | DebateSession (channel, participants, rounds, threads, orchestrator) |
| Artifacts | Reads: current draft (via DocumentSet). Writes: `findings/review-{id}.md`, `findings/accepted.md` |
| MCP tools | `start_debate`, `raise_point`, `respond_to`, `flag_human`, `get_debate_summary`, `end_debate`, `report_context`, `start_thread`, `reply_to_thread`, `resolve_thread`, `get_thread_summary`, `load_workspace`, `export_debate_summary` |
| UI panels | `<debate-feed>`, `<review-tracker>`, `<selection-threads>`, `<review-pipeline>`, `<document-timeline>` (all existing panels) |

ReviewSession is absorbed into ReviewFacet — it was always a lightweight DebateSession.

### 4. Cross-Facet Integration

Four integration layers, each with a clear responsibility:

| Layer | Responsibility | Not used for |
|-------|---------------|--------------|
| **Artifacts** (files in working directory) | All data flow between facets | Real-time notifications |
| **DocumentSet** (session-level) | Current document identity, comparison pair | Data content (that's in files) |
| **ArtefactRef** (Qhorus channel messages) | Facet-to-facet notification of artifact availability | Data flow (artifacts are files, not messages) |
| **WebSocket events** | UI panel refresh | Facet-to-facet signalling |

**What is explicitly absent:**
- No facet-to-facet method calls
- No event subscriptions between facets
- No shared mutable state between facets (DocumentSet is session-level)
- No mediator routing commands between facets

**Artifact contract per facet:**

```
VoiceFacet:
  reads:  nothing
  writes: notes/accumulated.md

BrainstormFacet:
  reads:  nothing
  writes: brainstorm/options.json, brainstorm/selected.md

DraftFacet:
  reads:  notes/accumulated.md, brainstorm/selected.md, findings/accepted.md
  writes: pipeline/01-*.md through pipeline/N-*.md

ReviewFacet:
  reads:  current draft (via DocumentSet)
  writes: findings/review-*.md, findings/accepted.md
```

**ArtefactRef notification flow:** When a facet produces an artifact (e.g., VoiceFacet writes `notes/accumulated.md`), it posts a Qhorus channel message with an `ArtefactRef` pointing to the output file. Other facets can observe these messages to know when new input is available — without polling or direct coupling. The `ArtefactRef` references the artifact; it doesn't replace it.

### 5. Dynamic Tool Registration

Each facet registers its MCP tools on activation via Quarkus MCP's `ToolManager`:

```java
// On facet activation
toolManager.newTool("start_recording")
    .setDescription("Begin voice recording")
    .setInputSchema(schema)
    .setHandler(this::handleStartRecording)
    .register();

// On facet deactivation
toolManager.removeTool("start_recording");
```

Connected MCP clients receive `tools/list_changed` notifications automatically. An LLM client that connects when Voice + Draft facets are active sees ~10 tools. Activating Review adds ~13 more. Deactivating Voice removes 3.

Session-level tools (document management, facet lifecycle) are registered at session creation and never removed.

### 6. Runtime Layout Switching

The browser workbench rebuilds its panel layout when the active facet set changes.

```typescript
function buildLayout(activeFacets: Set<string>): LayoutNode {
    const panels: PanelConfig[] = [];

    if (activeFacets.has('voice'))
        panels.push({ name: 'voice-controls', position: 'topbar' });
    if (activeFacets.has('brainstorm'))
        panels.push({ name: 'brainstorm-options', position: 'right' });
    if (activeFacets.has('draft'))
        panels.push(
            { name: 'document-editor', position: 'left' },
            { name: 'document-preview', position: 'right' },
            { name: 'pipeline-navigator', position: 'bottom' }
        );
    if (activeFacets.has('review'))
        panels.push(
            { name: 'debate-feed', position: 'right' },
            { name: 'review-tracker', position: 'right' },
            { name: 'selection-threads', position: 'right' }
        );

    return composeLayout(panels);
}
```

The WebSocket `facet-activated` / `facet-deactivated` events trigger layout rebuilds. The pages layout system (`rows()`, `split()`, `hostPanel()`) is already declarative — rebuilding from a panel set is a natural extension.

### 7. Voice Pipeline Detail

**Audio capture:** Browser MediaRecorder API captures audio as PCM/WAV chunks. WebSocket endpoint `/api/voice` receives audio data and routes to the STT service.

**Transcription:** `SpeechToTextService` SPI (from `casehub-blocks-api`):

```java
public interface SpeechToTextService {
    TranscriptionResult transcribe(byte[] audioData, TranscriptionOptions options);
}
```

Implementation in `casehub-blocks-stt` uses whisper.cpp via Java FFM (Panama). Metal acceleration on Apple Silicon is automatic (whisper.cpp default on macOS).

**Cleanup pipeline (DraftHouse-owned):** The raw transcript goes through DraftFacet's pipeline. The LLM cleanup stage:
1. Removes filler words (um, uh, like, you know)
2. Removes repeated words and false starts
3. Adds punctuation and capitalisation
4. Fixes grammar
5. Preserves verbatim paragraphs marked by the speaker (e.g., "verbatim: ...")

The LLM has document context — it knows the target style, what's already written, and the document's subject matter. This context-awareness produces better cleanup than a generic filler-removal model.

**Three input modes:**

| Mode | Behaviour | Use case |
|------|-----------|----------|
| Push-to-talk | Hold button/key to record, release to transcribe | Discrete instructions, quick additions |
| Continuous | Always listening, pauses delimit notes | Extended dictation, stream of consciousness |
| Record then review | Record a stretch, see cleaned transcript, edit before submitting | Careful input where accuracy matters |

### 8. Module Boundaries

```
casehub-blocks-api (pure Java)
  └── SpeechToTextService SPI

casehub-blocks-stt (optional submodule, native deps)
  └── WhisperSpeechToTextService (FFM/Panama + whisper.cpp)

casehub-drafthouse/server/api
  └── DraftHouseSession, Facet interface, artifact contracts

casehub-drafthouse/server/runtime
  └── VoiceFacet, BrainstormFacet, DraftFacet, ReviewFacet
  └── Dynamic ToolManager registration
  └── WebSocket event bus, REST endpoints
  └── Consumes casehub-blocks-stt

casehub-drafthouse/server/runtime/src/main/webui
  └── Workbench layout composition from active facets
  └── New panels: document-editor, document-preview, pipeline-navigator, voice-controls
```

### 9. Migration Path

The existing mode architecture transitions to facets incrementally:

1. **Create DraftHouseSession container** with facet activation/deactivation lifecycle
2. **Wrap DebateSession in ReviewFacet** — move MCP tools from DebateMcpTools/ThreadMcpTools to ReviewFacet's activate/deactivate using ToolManager
3. **Wrap BrainstormSession in BrainstormFacet** — move tools from BrainstormMcpTools
4. **Absorb ReviewSession into ReviewFacet** — ReviewSession was always a lightweight DebateSession
5. **Create DraftFacet** — new code: pipeline processing, editor panel, preview panel
6. **Create VoiceFacet** — new code: audio capture, Whisper integration via casehub-blocks-stt
7. **Runtime layout switching** — replace fixed mode-based layout with facet-driven composition
8. **Remove old session registries** — ReviewSessionRegistry, DebateSessionRegistryImpl, BrainstormSessionRegistry consolidated into DraftHouseSession management

Steps 1-4 are refactoring (no new features, existing tests still pass). Steps 5-6 are new capability. Step 7-8 are cleanup.

## Testing Strategy

**Unit tests per facet:** Each facet tested in isolation with fixture files in the working directory. Write `notes/accumulated.md` → activate DraftFacet → assert pipeline stages produced correctly. No need to stand up other facets.

**Integration tests:** Facet activation/deactivation lifecycle, tool registration/deregistration, layout switching. Verify `tools/list_changed` notifications fire correctly.

**E2E tests:** Voice recording flow (mock Whisper for CI), pipeline stage progression and diff display, review-during-drafting workflow.

**Existing tests:** All existing debate, brainstorm, and review E2E tests must continue to pass after wrapping in facets. The facet wrapper adds lifecycle management; the internal behaviour is unchanged.

## References

- [decisions.md](decisions.md) — D1-D7 validated decisions
- `server/api/src/main/java/io/casehub/drafthouse/DebateSession.java` — existing debate session (becomes ReviewFacet state)
- `server/api/src/main/java/io/casehub/drafthouse/BrainstormSession.java` — existing brainstorm session (becomes BrainstormFacet state)
- `server/api/src/main/java/io/casehub/drafthouse/ReviewSession.java` — absorbed into ReviewFacet
- `server/runtime/src/main/webui/src/index.ts` — current layout system (to be replaced with facet-driven composition)
- [Quarkus + FFM Whisper tutorial](https://www.the-main-thread.com/p/java-speech-to-text-quarkus-whisper-ffm) — STT integration reference
- [whisper.cpp](https://github.com/ggml-org/whisper.cpp) — Metal acceleration on Apple Silicon
- `io.casehub.engine.mcp.McpWorkerFunctionProvider` — platform MCP discovery (NOT used for tool scoping — see D5)
- `io.quarkiverse.mcp.server.ToolManager` — dynamic tool registration mechanism
- `io.casehub.qhorus.api.message.ArtefactRef` — artifact reference notification pattern
