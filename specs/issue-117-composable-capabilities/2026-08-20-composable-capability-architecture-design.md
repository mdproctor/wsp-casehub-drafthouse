# Composable Facet Architecture with Voice-First Drafting

**Issue:** casehubio/drafthouse#117
**Date:** 2026-08-20
**Decisions:** [decisions.md](decisions.md) (D1–D7)

## Problem

DraftHouse has three independent session types (ReviewSession, DebateSession, BrainstormSession) with separate registries, 37 always-registered MCP tools, and fixed UI layouts selected at URL load time. This creates friction:

- LLM clients see all 37 tools regardless of context
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
  ├── workingDirectory: Path                ← artifact space (unifies DebateSession.workspacePath)
  └── activeFacets: Map<String, Facet>
```

`DraftHouseSession` is a pure domain type in the api module. It does not hold runtime infrastructure references (`ToolManager`, `WebSocketEventBus`) — those are injected into facet implementations by the CDI container in the runtime module.

**Qhorus channel model:** DraftHouseSession does not own a Qhorus channel. Qhorus channels are created by facet-specific domain entry points (e.g., `start_debate` creates a channel for the debate session), not by `activate()`. ReviewFacet's entry points (`start_debate`, `start_review`, `load_workspace`) each create a Qhorus channel as part of domain initialization. VoiceFacet, DraftFacet, and BrainstormFacet do not use Qhorus channels — they work with files and in-memory state. On `deactivate()`, ReviewFacet cleans up any active channel.

**Working directory:** `DraftHouseSession.workingDirectory` replaces `DebateSession.workspacePath`. When ReviewFacet activates with a DebateSession, it passes the session's workingDirectory to `DebateSession.setWorkspacePath()`. All facet artifact I/O is scoped to this directory.

**Session-level MCP tools** (always available regardless of active facets):
- `create_session` — creates a new DraftHouseSession, optionally with a working directory. Returns session ID. Preserves MCP-first workflow (no REST required).
- `set_working_directory` — sets or changes the session's working directory (artifact space). Available without activating any facet.
- `add_document`, `remove_document`, `list_documents`, `set_comparison` (lifted from DebateMcpTools)
- `list_facets`, `activate_facet`, `deactivate_facet`
- `list_reviewers`, `get_reviewer_instructions`

**Session-level REST endpoints** (always available):
- `GET /api/sessions` — active session list (replaces `GET /api/debate/sessions`)
- `POST /api/sessions` — create session
- `DELETE /api/sessions/{id}` — end session

**Persistence:** `DraftHouseSessionStore` SPI (new, in api module) persists session metadata, active facet set, and working directory path. Each facet with durable state persists through its existing mechanism: ReviewFacet continues to use `DebateSessionStore` for `DebateSessionSnapshot`. Artifact files are inherently persistent (filesystem). On restore, DraftHouseSession reactivates its recorded facets, and each facet restores from its own store.

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

```java
public record ArtifactSpec(String pathPattern, String description, boolean required) {
    public ArtifactSpec(String pathPattern, String description) {
        this(pathPattern, description, false);
    }
}
```

`ArtifactSpec` defines the artifact contract for a facet. `pathPattern` is a glob pattern relative to the working directory (e.g., `notes/accumulated.md`, `stages/*.md`). `description` documents the artifact's purpose. `required` indicates whether the facet can function without this input — defaults to `false` (optional). Facets generate from whatever inputs are available, gracefully degrading when optional inputs are absent.

**Two-phase facet lifecycle:**

1. **Activation** (`activate()`) — registers MCP tools via `ToolManager.newTool()` and sets up UI panels. The facet is "available" — its tools are visible to MCP clients. No domain-specific state is created yet (no DebateSession, no Qhorus channel, no BrainstormSession).

2. **Domain initialization** (via entry-point tools like `start_debate`, `start_brainstorm`) — creates domain-specific state: DebateSession + Qhorus channel, BrainstormSession, etc. The facet is now "active" with an initialized domain session.

3. **Deactivation** (`deactivate()`) — if a domain session is active, cleans it up (ends the debate, destroys the Qhorus channel, etc.), then deregisters tools. Connected MCP clients receive automatic `tools/list_changed` notifications on each transition.

Between activation and domain initialization, tools that require a domain session (e.g., `raise_point` without an active debate) return a clear error directing the user to call the entry-point tool first. Tools that don't require domain state (e.g., `list_reviewers`) work immediately.

Facet implementations are CDI beans in the runtime module — they receive `ToolManager` and `WebSocketEventBus` via `@Inject`.

The `Facet` interface and `ArtifactSpec` live in the api module (pure Java). Facet implementations live in the runtime module.

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

**STT integration:** Consumes the new `SpeechToTextService` SPI from `casehub-blocks-api` (see §7 and §8 — both the SPI and its whisper.cpp implementation in `casehub-blocks-stt` are new artifacts that must be created before VoiceFacet can be built). Browser captures audio via MediaRecorder API, streams PCM to Quarkus server endpoint, server transcribes and writes to working directory.

#### BrainstormFacet

Option exploration and selection. Wraps existing `BrainstormSession` as internal state.

| Aspect | Detail |
|--------|--------|
| State | BrainstormSession (options list, ACTIVE/CONVERGED/ABANDONED) |
| Artifacts | Writes: `brainstorm/options.json`, `brainstorm/selected.md` |
| MCP tools | `start_brainstorm`, `present_options`, `update_option`, `set_recommendation`, `mark_eliminated`, `mark_selected`, `end_brainstorm` |
| UI panels | `<brainstorm-options>` (existing panel, extracted to blocks-ui) |

**Artifact production:** BrainstormSession is currently in-memory with no file I/O. BrainstormFacet adds file-writing around it:
- `brainstorm/options.json` — written on each state mutation (`present_options`, `update_option`, `set_recommendation`, `mark_eliminated`). Contains the full options list with status, description, and tradeoffs.
- `brainstorm/selected.md` — written on convergence (`mark_selected`). Contains the selected option's title, description, and rationale. Consumed by DraftFacet as the drafting brief.

#### DraftFacet

Immutable stage-based processing, document editing, preview rendering.

| Aspect | Detail |
|--------|--------|
| State | Processing stages, current editor content, preview state |
| Artifacts | Reads (all optional): `notes/accumulated.md`, `brainstorm/selected.md`, `findings/accepted.md`. Writes: `stages/01-raw-notes.md` through `stages/N-*.md` |
| MCP tools | `generate_draft`, `edit_stage`, `rerun_from_stage`, `get_stage_status` |
| UI panels | Dual-screen: editable markdown editor (left), rendered preview (right), stage navigator |
| REST | `GET /api/sessions/{id}/stages` — list stages with staleness indicators |

**Processing stages** (the term "stages" is used to avoid collision with the existing review pipeline subsystem — `PipelineSession`, `PipelineOrchestrator`, `PipelineMcpTools` — which manages multi-dimension review orchestration with phases, checkpoints, and dimension tracking):

```
notes/accumulated.md  ──read──→  stages/01-raw-notes.md
                                        │
                                        ↓ (LLM cleanup: filler removal, punctuation, grammar)
                                 stages/02-cleaned-notes.md
                                        │
                                        ↓ (LLM drafting: notes → document prose)
                                 stages/03-draft.md
                                        │
                                        ↓ (LLM revision: incorporate accepted findings)
                                 stages/04-revised-draft.md
```

Each stage is a file. The diff panel shows any two stages side-by-side. Hand-edit any stage, then explicitly trigger "rerun from here" to regenerate downstream stages. No auto-triggering.

**Staleness detection:** Each stage output records the hash of its inputs as YAML frontmatter:

```markdown
---
generated-from-hash: sha256:abc123def456...
generated-at: 2026-08-20T10:30:00Z
---
# Stage content follows...
```

When inputs change, the UI shows a staleness indicator — "inputs changed since last run." The user decides when to re-run. Frontmatter survives session restarts, is human-readable, and is compatible with diff viewing. Hand-editing a stage file preserves the hash metadata (indicating what it was generated from, which is now stale relative to the edit).

**Optional inputs and composability:** All three DraftFacet inputs are optional (`required=false`). `generate_draft` uses whatever inputs are available:
- Voice + Draft (no Brainstorm): generates from `notes/accumulated.md` only
- Brainstorm + Draft (no Voice): generates from `brainstorm/selected.md` only
- Draft + Review (no Voice): generates from `findings/accepted.md` plus any existing draft
- All three: combines notes, brainstorm brief, and accepted findings
- No inputs at all: returns an error — at least one input is needed

This enables free composition of facets without requiring all upstream facets to be active.

#### ReviewFacet

Debate/critique with agents, threads, points, and review pipeline orchestration. Wraps existing `DebateSession` as internal state, plus `PipelineSession` for multi-dimension review orchestration.

| Aspect | Detail |
|--------|--------|
| State | DebateSession (channel, participants, rounds, threads, orchestrator), PipelineSession (review pipeline orchestration with phases and checkpoints) |
| Artifacts | Reads: current draft (via DocumentSet). Writes: `findings/review-{id}.md`, `findings/accepted.md` |
| MCP tools | See complete tool mapping table below |
| UI panels | `<debate-feed>`, `<review-tracker>`, `<selection-threads>`, `<review-pipeline>`, `<document-timeline>` (all existing panels) |

**ReviewSession absorption:** ReviewSession is structurally a restricted DebateSession — a single-reviewer, channel-based review with inline document content. In the facet model, ReviewFacet provides both entry points:
- `start_review` (from DraftHouseMcpTools) — creates a single-reviewer DebateSession, adds documents from file paths to the session-level DocumentSet, registers the reviewer instance. Preserves the simple 1:1 review workflow.
- `start_debate` — creates the full multi-agent DebateSession with configurable participants. The existing multi-agent debate workflow.

Both entry points produce a DebateSession as ReviewFacet's internal state. `start_review` is a convenience path that configures a single-reviewer debate — not a structurally distinct session type.

**Artifact production:** ReviewFacet produces `findings/accepted.md` via:
1. The `export_debate_summary` tool — user explicitly exports debate output to a file path (existing capability).
2. On `end_debate` — if the working directory is set, ReviewFacet automatically writes `findings/accepted.md` with all accepted/resolved findings from the debate (new behaviour).

Individual review findings are written to `findings/review-{id}.md` as they are raised and accepted during the debate.

**DocumentSet:** DebateSession receives the session-level DocumentSet via its existing 5-argument constructor: `DebateSession(channelId, sessionId, channelName, agentId, documentSet)`. DebateSession's document delegation methods (`addDocument`, `removeDocument`, `setComparison`, etc.) operate on this shared instance — no delegation changes required. `branchFrom()` continues to copy the DocumentSet for branched debates (independent document state for branched sessions). `snapshot()` captures the shared DocumentSet's state unchanged.

**REST endpoints under ReviewFacet** (available when ReviewFacet is active — migrated from `DebateEventResource` at `/api/debate`):
- `POST /api/sessions/{id}/review/selection` — update selection (was `POST /api/debate/{id}/selection`)
- `DELETE /api/sessions/{id}/review/selection` — clear selection
- `GET /api/sessions/{id}/review/documents` — document listing (was `GET /api/debate/{id}/documents`)
- `POST /api/sessions/{id}/review/comparison` — set comparison pair
- `GET /api/sessions/{id}/review/snapshot/{index}` — snapshot content

### Complete Tool-to-Facet Mapping

All 37 existing tools mapped to their target:

| Current class | Tool | Target |
|---------------|------|--------|
| **DraftHouseMcpTools** | `start_review` | ReviewFacet |
| | `update_selection` | ReviewFacet |
| | `query_review` | ReviewFacet |
| | `end_review` | ReviewFacet |
| | `list_reviewers` | Session-level |
| | `get_reviewer_instructions` | Session-level |
| **DebateMcpTools** | `start_debate` | ReviewFacet |
| | `raise_point` | ReviewFacet |
| | `respond_to` | ReviewFacet |
| | `flag_human` | ReviewFacet |
| | `get_debate_summary` | ReviewFacet |
| | `end_debate` | ReviewFacet |
| | `post_memo` | ReviewFacet |
| | `request_subagent` | ReviewFacet |
| | `get_debate_summary_at_round` | ReviewFacet |
| | `restart_from_round` | ReviewFacet |
| | `report_context` | ReviewFacet |
| | `export_debate_summary` | ReviewFacet |
| | `load_workspace` | ReviewFacet |
| | `add_document` | Session-level |
| | `remove_document` | Session-level |
| | `list_documents` | Session-level |
| | `set_comparison` | Session-level |
| **ThreadMcpTools** | `start_thread` | ReviewFacet |
| | `reply_to_thread` | ReviewFacet |
| | `resolve_thread` | ReviewFacet |
| | `get_thread_summary` | ReviewFacet |
| **BrainstormMcpTools** | `start_brainstorm` | BrainstormFacet |
| | `present_options` | BrainstormFacet |
| | `update_option` | BrainstormFacet |
| | `set_recommendation` | BrainstormFacet |
| | `mark_eliminated` | BrainstormFacet |
| | `mark_selected` | BrainstormFacet |
| | `end_brainstorm` | BrainstormFacet |
| **PipelineMcpTools** | `start_pipeline` | ReviewFacet |
| | `update_pipeline` | ReviewFacet |
| | `load_decisions` | ReviewFacet |

**New tools** (introduced by this spec):
- Session-level: `create_session`, `set_working_directory`, `list_facets`, `activate_facet`, `deactivate_facet`
- VoiceFacet: `start_recording`, `stop_recording`, `list_voice_notes`
- DraftFacet: `generate_draft`, `edit_stage`, `rerun_from_stage`, `get_stage_status`

### 4. Cross-Facet Integration

Three integration layers, each with a clear responsibility:

| Layer | Responsibility | Not used for |
|-------|---------------|--------------|
| **Artifacts** (files in working directory) | All data flow between facets | Real-time notifications |
| **DocumentSet** (session-level) | Current document identity, comparison pair | Data content (that's in files) |
| **WebSocket events** | UI notification of artifact changes and staleness | Facet-to-facet signalling |

**Cross-facet data flow is always file-mediated.** When a facet produces an artifact, it writes a file and emits a WebSocket file-change event. The UI shows a staleness indicator on downstream facets. The user decides when to re-run downstream stages. There is no direct facet-to-facet notification — the user is always in the loop.

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
  reads (all optional): notes/accumulated.md, brainstorm/selected.md, findings/accepted.md
  writes: stages/01-*.md through stages/N-*.md

ReviewFacet:
  reads:  current draft (via DocumentSet)
  writes: findings/review-*.md, findings/accepted.md
```

**ArtefactRef for audit/provenance (within channel-bearing facets only):** Facets with active Qhorus channels (currently only ReviewFacet) attach `ArtefactRef` records to channel messages when producing file artifacts. This provides an audit trail linking review deliberation to its file outputs — connecting the debate that produced a finding to the `findings/review-{id}.md` file. ArtefactRef is NOT a cross-facet notification mechanism — it records artifact production on the channel for observability. The integration uses:
- `ArtefactType.DOCUMENT` for file-based artifacts
- `MessageDispatch.builder().artefactRefs(List.of(ref)).build()` to attach refs to channel messages
- The `scope` field uses the Qhorus `SelectionScope` (not DraftHouse's `SelectionScope` — distinct types in different packages)

### 5. Dynamic Tool Registration

Each facet registers its MCP tools on activation via Quarkus MCP's `ToolManager`:

```java
// On facet activation — handler signature: Function<ToolArguments, ToolResponse>
toolManager.newTool("start_recording")
    .setDescription("Begin voice recording")
    .addArgument("mode", "Recording mode: push-to-talk, continuous, review",
                 true, String.class)
    .setHandler(args -> {
        String mode = (String) args.args().get("mode");
        return handleStartRecording(mode);
    })
    .register();

// On facet deactivation
toolManager.removeTool("start_recording");
```

Connected MCP clients receive `tools/list_changed` notifications automatically. An LLM client that connects when Voice + Draft facets are active sees 18 tools (11 session-level + 3 voice + 4 draft). Activating Review adds 24 more. Deactivating Voice removes 3.

Session-level tools (document management, facet lifecycle, reviewer lookup) are registered at session creation and never removed.

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
            { name: 'stage-navigator', position: 'bottom' }
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

**Event routing:** WebSocket subscriptions transition from mode-scoped (`debate:{id}`, `brainstorm:{id}`) to session-scoped (`session:{id}`). All facet events route through a single session subscription. The event payload includes a `facet` field so the UI can dispatch to the correct panel handlers. File-change watching (`file:{path}`) remains path-scoped and is managed by the active facet set — activating DraftFacet subscribes to `stages/` changes, deactivating it unsubscribes. Panel toggle state, keyboard shortcuts, and diff navigation continue to work as today — they are panel-level concerns independent of the session model.

### 7. Voice Pipeline Detail

**Audio capture:** Browser MediaRecorder API captures audio as PCM/WAV chunks. WebSocket endpoint `/api/voice` receives audio data and routes to the STT service.

**Transcription:** `SpeechToTextService` SPI — a new interface to be declared in `casehub-blocks-api` (does not exist yet):

```java
public interface SpeechToTextService {
    TranscriptionResult transcribe(byte[] audioData, TranscriptionOptions options);
}
```

Implementation in `casehub-blocks-stt` (a new optional submodule to be created) uses whisper.cpp via Java FFM (Panama). Metal acceleration on Apple Silicon is automatic (whisper.cpp default on macOS). Both the SPI declaration and the implementation module are prerequisites for VoiceFacet — they do not exist yet (see D4 and D7 for rationale).

**Cleanup stages (DraftHouse-owned):** The raw transcript goes through DraftFacet's processing stages. The LLM cleanup stage:
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
casehub-blocks-api (pure Java) — NEW SPI DECLARATION
  └── SpeechToTextService SPI (new — does not exist yet)

casehub-blocks-stt (new optional submodule, native deps) — NEW MODULE
  └── WhisperSpeechToTextService (FFM/Panama + whisper.cpp)

casehub-drafthouse/server/api (pure Java — no Quarkus, no JPA)
  └── DraftHouseSession (domain state only: id, documentSet, workingDirectory, metadata, activeFacets)
  └── Facet interface, ArtifactSpec record
  └── DraftHouseSessionStore SPI (new)

casehub-drafthouse/server/runtime
  └── VoiceFacet, BrainstormFacet, DraftFacet, ReviewFacet (CDI beans, @Inject ToolManager/EventBus)
  └── Dynamic ToolManager registration
  └── WebSocket event bus, REST endpoints
  └── JpaDraftHouseSessionStore (new)
  └── Consumes casehub-blocks-stt

casehub-drafthouse/server/runtime/src/main/webui
  └── Workbench layout composition from active facets
  └── New panels: document-editor, document-preview, stage-navigator, voice-controls
```

### 9. Migration Path

The existing mode architecture transitions to facets incrementally:

1. **Create DraftHouseSession container** with facet activation/deactivation lifecycle. Define `DraftHouseSessionStore` SPI in api and `JpaDraftHouseSessionStore` implementation in runtime.
2. **Wrap DebateSession in ReviewFacet** — move MCP tools from DebateMcpTools, ThreadMcpTools, and PipelineMcpTools to ReviewFacet's activate/deactivate using ToolManager.
3. **Wrap BrainstormSession in BrainstormFacet** — move tools from BrainstormMcpTools. Add file-writing for `brainstorm/options.json` and `brainstorm/selected.md`.
4. **Absorb ReviewSession into ReviewFacet** — move DraftHouseMcpTools review tools (`start_review`, `update_selection`, `query_review`, `end_review`) into ReviewFacet. `start_review` creates a single-reviewer DebateSession. `list_reviewers` and `get_reviewer_instructions` become session-level tools.
5. **Lift DocumentSet and document tools** — move `add_document`, `remove_document`, `list_documents`, `set_comparison` from DebateMcpTools to session-level. DebateSession receives the session-level DocumentSet via its existing 5-argument constructor.
6. **Migrate REST endpoints** — `DebateEventResource` endpoints transition to session-scoped URLs under ReviewFacet: `/api/debate/{id}/selection` → `/api/sessions/{id}/review/selection`, `/api/debate/{id}/documents` → `/api/sessions/{id}/review/documents`, `/api/debate/{id}/comparison` → `/api/sessions/{id}/review/comparison`, `/api/debate/{id}/snapshot/{index}` → `/api/sessions/{id}/review/snapshot/{index}`. `GET /api/debate/sessions` → `GET /api/sessions`.
7. **Create SpeechToTextService SPI** in `casehub-blocks-api` and `WhisperSpeechToTextService` in new `casehub-blocks-stt` module. These are prerequisites for step 8.
8. **Create VoiceFacet** — new code: audio capture, Whisper integration via casehub-blocks-stt.
9. **Create DraftFacet** — new code: stage processing, editor panel, preview panel.
10. **Runtime layout switching** — replace fixed mode-based layout with facet-driven composition. Transition WebSocket subscriptions from mode-scoped to session-scoped.
11. **Remove old session registries** — ReviewSessionRegistry, DebateSessionRegistryImpl, BrainstormSessionRegistry consolidated into DraftHouseSession management.

Steps 1-6 are refactoring (no new features, existing tests still pass). Step 7 is new infrastructure. Steps 8-9 are new capability. Steps 10-11 are cleanup.

## Testing Strategy

**Unit tests per facet:** Each facet tested in isolation with fixture files in the working directory. Write `notes/accumulated.md` → activate DraftFacet → assert stages produced correctly. No need to stand up other facets.

**Integration tests:** Facet activation/deactivation lifecycle, tool registration/deregistration, layout switching. Verify `tools/list_changed` notifications fire correctly.

**E2E tests:** Voice recording flow (mock Whisper for CI), stage progression and diff display, review-during-drafting workflow.

**Existing tests:** All existing debate, brainstorm, and review E2E tests must continue to pass after wrapping in facets. The facet wrapper adds lifecycle management; the internal behaviour is unchanged. Pre-existing disabled assertions in `DebatePanelE2ETest` (3 TODOs at lines 254, 358, 381, related to casehub-ledger TENANCY_ID migration) are orthogonal to the facet architecture — they are tracked as separate ledger migration debt.

## References

- [decisions.md](decisions.md) — D1-D7 validated decisions
- `server/api/src/main/java/io/casehub/drafthouse/DebateSession.java` — existing debate session (becomes ReviewFacet state)
- `server/api/src/main/java/io/casehub/drafthouse/BrainstormSession.java` — existing brainstorm session (becomes BrainstormFacet state)
- `server/api/src/main/java/io/casehub/drafthouse/ReviewSession.java` — absorbed into ReviewFacet
- `server/api/src/main/java/io/casehub/drafthouse/debate/PipelineSession.java` — review pipeline orchestration (ReviewFacet state)
- `server/runtime/src/main/java/io/casehub/drafthouse/PipelineMcpTools.java` — review pipeline tools (mapped to ReviewFacet)
- `server/runtime/src/main/java/io/casehub/drafthouse/DebateEventResource.java` — existing REST endpoints (migrated under facet model)
- `server/runtime/src/main/webui/src/index.ts` — current layout system (to be replaced with facet-driven composition)
- [Quarkus + FFM Whisper tutorial](https://www.the-main-thread.com/p/java-speech-to-text-quarkus-whisper-ffm) — STT integration reference
- [whisper.cpp](https://github.com/ggml-org/whisper.cpp) — Metal acceleration on Apple Silicon
- `io.casehub.engine.mcp.McpWorkerFunctionProvider` — platform MCP discovery (NOT used for tool scoping — see D5)
- `io.quarkiverse.mcp.server.ToolManager` — dynamic tool registration mechanism
- `io.casehub.qhorus.api.message.ArtefactRef` — artifact reference notification pattern
- `io.casehub.qhorus.api.message.MessageDispatch` — message builder with `artefactRefs()` for ArtefactRef integration
