## D1: PipelineSession architecture

**Choice:** Thin registry over existing WorkspaceWatchers
**Alternatives:**
- Heavy pipeline orchestrator — server launches review.py directly, duplicates skill orchestration
- Event aggregation middleware — stateless, can't track HIL checkpoint state server-side
**Rationale:** Preserves separation where the skill orchestrates and the server visualizes. PipelineSession is a coordinator that groups dimension WorkspaceWatchers, tracks aggregate state, detects HIL checkpoint conditions, and pushes unified pipeline-progress events. The calling session creates pipelines via MCP tools.
**Trade-offs:** Requires the calling session to explicitly create the pipeline via MCP (extra tool calls), but avoids duplicating orchestration logic.
**Exploration:** quick
**Status:** captured

## D2: Pipeline event model

**Choice:** Extend ProgressLogParser + PipelineSession aggregates
**Alternatives:**
- New JSONL parser bypass — parallel JSON parser for structured EVENT lines, breaks single-parser contract
- Server polls workspace directories — periodic tracker.md reads, introduces polling latency
**Rationale:** Incremental extension of existing patterns. Add dimension-level event types to ProgressLogParser (dimension_start, round_findings, round_end, dimension_done). WorkspaceWatcher tails them as today. PipelineSession subscribes to per-dimension workspace-progress events and emits unified pipeline-progress topic with aggregate state.
**Trade-offs:** More parser types to maintain, but they're mechanical regex additions following the established pattern.
**Exploration:** quick
**Status:** captured

## D3: Review-pipeline panel layout

**Choice:** Vertical pipeline tracker — pipeline header + dimension cards + findings feed
**Alternatives:**
- Tabbed dimension view — per-dimension tabs, loses unified findings view
- Dashboard grid — 2×2 grid, cramped once findings accumulate
**Rationale:** Unified view shows the whole pipeline at a glance. Pipeline header gives phase awareness, dimension cards give per-dimension status, findings feed gives cross-dimension comparison. Scrolling is the only cost.
**Trade-offs:** Vertical scrolling needed when findings accumulate, but this is the natural browsing model for a feed-style view.
**Exploration:** quick
**Status:** captured

## D4: MCP tool surface

**Choice:** Three focused tools — start_pipeline, update_pipeline, load_decisions
**Alternatives:**
- Single tool with subcommands — fewer registrations but complex action enum, harder for LLM discovery
- Five granular tools — most explicit but clutters MCP tool list
**Rationale:** Clean separation: lifecycle (start_pipeline), state updates (update_pipeline), decisions (load_decisions). Three tools is manageable and each has a clear responsibility. update_pipeline is how HIL checkpoint state reaches the browser — the watchdog cron calls it when presenting checkpoints to the human.
**Trade-offs:** Three new MCP tools to maintain and document. But each maps to a distinct concern.
**Exploration:** quick
**Status:** captured

## D5: Workbench integration

**Choice:** Add to right-side split with toggle — fourth panel, topbar button, auto-shown on pipeline-created event
**Alternatives:**
- Replace review-tracker conditionally — overloads review-tracker's responsibility, can't coexist with per-point tracking
- Separate layout mode — cleanest layout but requires URL switch mid-workflow
**Rationale:** Incremental, uses existing panel toggle pattern. Pipeline panel appears when start_pipeline is called. Split ratios adjust dynamically. Default hidden when no pipeline is active.
**Trade-offs:** 4 panels in the right split gets tight on smaller screens.
**Exploration:** quick
**Status:** captured

## D6: Decision validation display

**Choice:** Collapsible section at top of pipeline panel — decision cards above pipeline header
**Alternatives:**
- Separate decisions sub-panel — full isolation but fragments the pipeline view
- Inline in findings feed — simplest but decisions get lost in dimension findings noise
**Rationale:** Decisions and dimensions share the same panel but are visually separated. Decision cards show ID, title, choice, status (captured/revised/confirmed/challenged/rejected). Expandable for alternatives, rationale, trade-offs. Auto-collapses after all decisions are confirmed to keep focus on dimension pipeline.
**Trade-offs:** Decision section takes vertical space at top when expanded. But auto-collapse after validation mitigates this.
**Exploration:** quick
**Depends on:** D3 (panel layout)
**Status:** captured
