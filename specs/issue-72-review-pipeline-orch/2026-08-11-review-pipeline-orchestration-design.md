# Review Pipeline Orchestration — Design Spec

**Issue:** casehubio/drafthouse#72
**Date:** 2026-08-11
**Status:** Design approved

## Problem

Dimensional reviews (coherence, structure, robustness + cross-cutting) run as
independent processes with progress visible only in the terminal. The human has
no visual dashboard showing which dimensions are active, what findings have
accumulated, when a HIL checkpoint is pending in the terminal, or what decisions
were validated during brainstorming.

The #108 spec (phased HIL for dimensional reviews) established a three-phase
pipeline with watchdog-driven HIL checkpoints, all orchestrated from the calling
terminal session. This issue builds the browser-side visualization for that
pipeline — observation only, not control.

## Solution

A new `review-pipeline` panel in DraftHouse that provides real-time visualization
of the review pipeline. Four deliverables:

1. **Pipeline progress** — phase indicator + per-dimension cards with status,
   rounds, cost, elapsed time
2. **Findings per stage** — accumulated findings feed, grouped by dimension,
   filterable by priority
3. **HIL checkpoint indicators** — read-only status showing when a checkpoint
   is pending in the terminal
4. **Decision validation** — collapsible section showing brainstorming decisions
   with their validation status

**Interaction model:** The browser is a dashboard. HIL decisions (approve, refuse,
discuss) stay in the terminal session as #108 designed. The pipeline panel shows
state, not controls.

## Architecture

```
Calling Session (Claude Code / design-review skill)
  │
  ├── start_pipeline(debateSessionId, dimensions, ordered, specPath)
  │     └── Server creates PipelineSession
  │           ├── WorkspaceWatcher per dimension (tails progress.log)
  │           └── Pushes pipeline-progress events via WebSocket
  │
  ├── update_pipeline(pipelineId, action, dimension?)
  │     └── HIL checkpoint state changes
  │         (checkpoint_reached, dimension_refused, dimension_accepted,
  │          crosscutting_started, pipeline_complete)
  │
  └── load_decisions(pipelineId, decisionsPath)
        └── Parses decisions.md, pushes pipeline-decisions events

Server (Quarkus)
  ├── PipelineSession (domain object — server/api/)
  │     ├── pipelineId: String
  │     ├── debateSessionId: String
  │     ├── dimensions: List<DimensionDescriptor>
  │     │     ├── name, workspacePath, status (PENDING/RUNNING/DONE/KILLED)
  │     │     ├── currentRound, totalRounds, degree
  │     │     ├── issueCount (by priority), cost, elapsed
  │     │     └── findings: List<PipelineFinding>
  │     ├── ordered: boolean
  │     ├── currentPhase: PipelinePhase
  │     ├── checkpointStatus: CheckpointStatus
  │     └── decisions: List<PipelineDecision>
  │
  ├── PipelineSessionRegistry (runtime, in-memory ConcurrentHashMap)
  │
  ├── PipelineMcpTools (runtime, 3 @Tool methods)
  │
  └── WebSocketEventBus
        ├── pipeline-progress topic (aggregate state)
        ├── pipeline-decisions topic (decision cards)
        └── workspace-progress topic (per-dimension detail, existing)

Browser (review-pipeline panel)
  ├── Decisions section (collapsible, top)
  ├── Pipeline header (phases)
  ├── Dimension cards (per-dimension status)
  └── Findings feed (accumulated, filterable)
```

## Server-Side Changes

### ProgressLogParser extensions

Four new event types to match review.py's existing JSONL events:

```java
public record DimensionStart(String dimension, String degree, int phase) implements ProgressEvent {}
public record RoundFindings(String dimension, int roundNumber, int issueCount) implements ProgressEvent {}
public record RoundEnd(String dimension, int roundNumber, int addressed, int contested) implements ProgressEvent {}
public record DimensionDone(String dimension, int totalRounds, double cost, int issues) implements ProgressEvent {}
```

These parse `EVENT: {...}` lines from progress.log. The `parse()` method gets
an early check for lines containing `EVENT:` — when detected, it extracts the
JSON payload and delegates to a `parseJsonEvent()` method that reads the `type`
field and constructs the appropriate record. Existing regex patterns for
plain-text lines are unchanged.

### PipelineSession domain model

Lives in `server/api/` (pure Java, no Quarkus deps) alongside `DebateSession`.

**`PipelineSession`:**
- `pipelineId: String` — unique identifier (UUID)
- `debateSessionId: String` — links to existing debate session for WebSocket routing
- `dimensions: List<DimensionDescriptor>` — ordered list of dimension states
- `ordered: boolean` — parallel vs sequential execution
- `specPath: String` — the spec being reviewed
- `currentPhase: PipelinePhase` — tracks pipeline progression
- `checkpointStatus: CheckpointStatus` — NONE, PENDING, or RESOLVED
- `decisions: List<PipelineDecision>` — parsed decisions from decisions.md

**`DimensionDescriptor`:**
- `name: String` — coherence, structure, robustness, crosscutting
- `workspacePath: String` — filesystem path to dimension's review workspace
- `status: DimensionStatus` — PENDING, RUNNING, DONE, KILLED
- `currentRound: int`, `totalRounds: int`, `degree: String`
- `issuesByPriority: Map<String, Integer>` — HIGH/MEDIUM/LOW counts
- `cost: double`, `elapsedSeconds: int`
- `findings: List<PipelineFinding>`

**`PipelinePhase`** enum:
`ROUND_1`, `HIL_CHECKPOINT_1`, `REMAINING_ROUNDS`, `HIL_CHECKPOINT_2`,
`CROSS_CUTTING`, `COMPLETE`

**`CheckpointStatus`** enum: `NONE`, `PENDING`, `RESOLVED`

**`DimensionStatus`** enum: `PENDING`, `RUNNING`, `DONE`, `KILLED`

**`PipelineDecision`:**
- `id: String` (D1, D2, ...)
- `title: String`
- `choice: String`
- `alternatives: List<String>` — one-line each
- `rationale: String`
- `tradeoffs: String`
- `status: String` — captured, revised, confirmed, challenged, rejected
- `explorationDepth: String` — quick, deep-analysis, multi-agent-debate
- `dependsOn: String` — optional, references another decision ID

**`PipelineFinding`:**
- `dimension: String`
- `issueId: String`
- `priority: String` — HIGH, MEDIUM, LOW
- `summary: String`
- `status: String` — open, verified, deferred
- `location: String` — optional, document location for cross-panel routing

### PipelineDecisionParser

Utility class in `server/api/` that parses the `decisions.md` markdown format
into a `List<PipelineDecision>`. Splits on `## D<N>:` headers, extracts
`**Choice:**`, `**Alternatives:**`, `**Rationale:**`, `**Trade-offs:**`,
`**Exploration:**`, `**Status:**`, and optional `**Depends on:**` fields.

### PipelineSessionRegistry

`@ApplicationScoped` in `server/runtime/`. In-memory
`ConcurrentHashMap<String, PipelineSession>`. No persistence — pipelines are
transient (tied to a running review, not historical).

Methods: `create(PipelineSession)`, `get(String pipelineId)`,
`remove(String pipelineId)`.

### PipelineMcpTools

New `@Tool` class in `server/runtime/` alongside `DebateMcpTools`.

**`start_pipeline`:**

```
@ToolArg description = "Debate session ID to link pipeline events to"
String debateSessionId

@ToolArg description = "Dimensions to review, as JSON array: [{name, workspacePath, degree}]"
String dimensions

@ToolArg description = "Sequential execution with cascading findings (true) or parallel (false)"
boolean ordered

@ToolArg description = "Path to the spec being reviewed"
String specPath
```

Creates a `PipelineSession`, starts a `WorkspaceWatcher` per dimension
(reusing the existing `WorkspaceWatcher` infra from `DraftHouseInstances`),
registers in `PipelineSessionRegistry`, pushes initial `pipeline-progress`
event. Returns pipeline state JSON.

Each dimension's `WorkspaceWatcher` pushes `workspace-progress` events as
today. `PipelineSession` registers a `ProgressListener` callback on each
`WorkspaceWatcher` — a simple `Consumer<ProgressEvent>` interface that
`WorkspaceWatcher.tailProgressLog()` calls before pushing to the
`WebSocketEventBus`. This gives PipelineSession server-side visibility
into dimension events without subscribing to its own WebSocket output.
When a `DimensionStart` event arrives, the corresponding
`DimensionDescriptor` transitions to `RUNNING`; when `DimensionDone`
arrives, it transitions to `DONE`.

Phase auto-advancement: when all non-cross-cutting dimensions report
`ROUND_COMPLETE` with `round: 1`, `currentPhase` advances to
`HIL_CHECKPOINT_1`. When all surviving dimensions report `REVIEW_TERMINAL`,
`currentPhase` advances to `HIL_CHECKPOINT_2`. The server detects these
conditions — the calling session does NOT need to call `update_pipeline`
for phase advancement that is derivable from events.

**`update_pipeline`:**

```
@ToolArg description = "Pipeline ID"
String pipelineId

@ToolArg description = "Action: checkpoint_reached, dimension_refused, dimension_accepted, crosscutting_started, pipeline_complete"
String action

@ToolArg description = "Dimension name (required for dimension_refused, dimension_accepted)"
String dimension
```

For state changes that can't be derived from progress.log — specifically,
HIL decisions made in the terminal:

- `checkpoint_reached` — sets `checkpointStatus = PENDING` (the calling
  session calls this when presenting the checkpoint to the human)
- `dimension_refused` — sets the named dimension's status to `KILLED`
- `dimension_accepted` — confirms the dimension continues (no-op for
  state but clears the pending checkpoint if all dimensions are resolved)
- `crosscutting_started` — sets `currentPhase = CROSS_CUTTING`
- `pipeline_complete` — sets `currentPhase = COMPLETE`, stops all watchers

Each action pushes an updated `pipeline-progress` event.

**`load_decisions`:**

```
@ToolArg description = "Pipeline ID"
String pipelineId

@ToolArg description = "Path to decisions.md file"
String decisionsPath
```

Reads the file, parses via `PipelineDecisionParser`, stores on the
`PipelineSession`, pushes a `pipeline-decisions` event. Can be called
multiple times as decisions are revised — each call replaces the
previous decision list.

### WebSocket event topics

Two new topics via the existing `WebSocketEventBus.pushMetadata()`:

**`pipeline-progress`** — full pipeline state snapshot:
```json
{
  "pipelineId": "...",
  "phase": "ROUND_1",
  "checkpointStatus": "NONE",
  "ordered": false,
  "dimensions": [
    {
      "name": "coherence",
      "status": "RUNNING",
      "currentRound": 2,
      "totalRounds": 3,
      "degree": "standard",
      "issuesByPriority": {"HIGH": 1, "MEDIUM": 2, "LOW": 0},
      "cost": 2.10,
      "elapsed": 145,
      "findingsCount": 3
    }
  ]
}
```

**`pipeline-decisions`** — decision list:
```json
{
  "pipelineId": "...",
  "decisions": [
    {
      "id": "D1",
      "title": "PipelineSession architecture",
      "choice": "Thin registry over existing WorkspaceWatchers",
      "alternatives": ["Heavy pipeline orchestrator", "Event aggregation middleware"],
      "rationale": "Preserves separation...",
      "tradeoffs": "Requires explicit MCP calls...",
      "status": "captured",
      "exploration": "quick"
    }
  ]
}
```

Both route through the debate session's channel ID.

### workspace-status enhancement

The existing `workspace-status` panel gains a handler for `pipeline-progress`
events. When a pipeline is active, it shows a compact summary:
`Pipeline: 2/3 dimensions running` or `Pipeline: checkpoint pending`. This
replaces the per-dimension `workspace-progress` display when a pipeline is
active — the pipeline panel owns the detailed view.

## Browser-Side Changes

### New panel: `review-pipeline`

A LitElement panel in `@casehubio/blocks-ui-document-workbench`, following
existing patterns (Shadow DOM, `configure(props)`, `onPagesEvent()`,
`_cleanups[]` teardown).

**Event subscriptions:**
- `pipeline-progress` — updates phase header, dimension cards, findings
- `pipeline-decisions` — updates decision section

**Layout (vertical stack):**

1. **Decisions section** — `<details>` element, collapsible.
   - Each decision: compact card with `D<N>: title` header + status badge
     (colour-coded: captured=grey, confirmed=green, challenged=amber,
     rejected=red, revised=blue)
   - Expanded: choice, alternatives (one-line each), rationale, trade-offs
   - Auto-collapses when all decisions reach terminal status

2. **Pipeline header** — horizontal row of phase segments:
   `Round 1` → `HIL` → `Rounds 2+` → `HIL` → `Cross-cutting`
   - Active: accent colour, bold text
   - Complete: success colour, checkmark icon
   - Pending: muted colour
   - HIL segments: warning badge when `checkpointStatus === 'PENDING'`

3. **Dimension cards** — one per dimension, ordered:
   - Status badge: running (pulsing dot, reuse workspace-status animation),
     done (green check), killed (red, strikethrough label)
   - Progress bar: thin bar showing `currentRound / totalRounds`
   - Stats line: issue count pills (HIGH=red, MEDIUM=amber, LOW=grey),
     cost (`$X.XX`), elapsed timer (`Nm Ns`)
   - Killed dimensions: "refused at round 1 checkpoint" label

4. **Findings feed** — scrollable list:
   - Each finding: dimension colour badge, priority pill, issue ID, summary
   - Grouped by dimension, most recent first within group
   - Filter bar: priority toggle buttons (HIGH / MEDIUM / LOW / all)
   - Click → dispatch `point-selected` event with document location
     (reuses existing cross-panel wiring from review-tracker)

**Empty state:** When no pipeline is active, the panel shows
"No active review pipeline" with muted text.

### Workbench integration (index.ts)

- Register: `registerPanel("review-pipeline", "review-pipeline")`
- Layout: add `withId("pipeline", hostPanel("review-pipeline", {}))` to the
  right-side vertical split
- Topbar: add toggle button `<button id="btn-pipeline">⚙ Pipeline</button>`
- Visibility: default hidden (`pipelineVisible = false`). Auto-shown when a
  `pipeline-progress` event arrives (via `pages-event` listener in index.ts).
  Toggle button controls manual show/hide.
- Split ratios when pipeline visible: debate 35%, pipeline 30%, threads 15%,
  review 20%
- Split ratios when pipeline hidden: debate 45%, threads 25%, review 30%
  (unchanged)

## Testing

### Java unit tests

- **ProgressLogParser** — `DimensionStart`, `RoundFindings`, `RoundEnd`,
  `DimensionDone` parse from `EVENT: {...}` lines. Existing plain-text
  event types unchanged. Null/blank/malformed input returns null.
- **PipelineSession** — dimension status transitions
  (PENDING→RUNNING→DONE, PENDING→RUNNING→KILLED), phase progression
  (ROUND_1→HIL_CHECKPOINT_1→REMAINING_ROUNDS→...), checkpoint status
  changes (NONE→PENDING→RESOLVED).
- **PipelineSessionRegistry** — create/get/remove, concurrent access.
- **PipelineDecisionParser** — parse decisions.md format, all fields
  including optional `Depends on:`, empty/malformed input.

### Integration tests

- **PipelineMcpTools** — `start_pipeline` creates session and returns
  state JSON, `update_pipeline` with each action pushes correct events,
  `load_decisions` parses and pushes decision data, unknown pipeline ID
  returns error.
- **WorkspaceWatcher dimension events** — watcher tails progress.log
  containing `EVENT:` lines, new event types parsed and pushed as
  `workspace-progress` metadata.

### E2E tests (Playwright)

- **Panel visibility** — hidden by default, topbar button toggles,
  auto-shown on pipeline-progress event.
- **Dimension cards** — mock pipeline-progress events via WebSocket,
  verify status badges, round progress, issue counts, cost.
- **Pipeline header** — active/complete/pending phase styling on
  state transitions.
- **Findings feed** — mock findings, verify rendering with dimension
  badges and priority pills. Click finding → `point-selected` dispatched.
- **Decision cards** — mock pipeline-decisions events, verify status
  badges. Auto-collapse when all confirmed.
- **HIL checkpoint** — mock checkpoint_reached, verify warning badge
  on HIL phase segment.

## Dependencies

- No new external dependencies
- Extends existing `ProgressLogParser`, `WorkspaceWatcher`, `WebSocketEventBus`
- New panel in `@casehubio/blocks-ui-document-workbench` (existing package)
- review.py already emits required JSONL events (soredium#198 shipped)

## Non-goals

- Browser-based HIL control (approve/refuse/discuss stays in the terminal)
- Persistence of pipeline state (transient, in-memory only)
- Changes to review.py or the design-review skill's orchestration logic
- Cross-dimension context flow during reviews (dimensions stay isolated)
- New event types in review.py (existing events are sufficient)
