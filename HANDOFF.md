# Handover — 2026-07-06

**Branch:** `main` (clean)

## What Happened This Session

### 1. Build fixed
The qhorus/ledger dependency chain was broken (ledger HEAD moved `LedgerEntry` from `runtime.model` to `api.model`, qhorus hadn't absorbed the change). User rebuilt qhorus externally. DraftHouse build is now green with the stable qhorus.

### 2. E2E tests verified
All 52 section highlight and cross-panel E2E tests pass — `SectionHighlightE2ETest`, `SectionHighlightVisualTest`, `CrossPanelE2ETest`. This validates heading matching, LLM compound location normalization, toggle behaviour, re-render persistence, forward/backward clicking, and scroll-to-location. The heading matcher is solid.

### 3. Document Workbench vision established
A brainstorming conversation produced a major strategic direction: **DraftHouse is not just a diff viewer — it's a document workbench**. The conversation explored how DraftHouse and the `design-review` skill (headless adversarial debates between Claude agents) relate, and discovered they share the same fundamental abstraction (structured conversations about documents) but serve different phases of a document's lifecycle.

### 4. Epic #93 filed with 8 child issues
### 5. Exploration spec design-reviewed and promoted
The spec at `docs/superpowers/specs/2026-07-05-document-workbench-exploration.md` was adversarially reviewed (3 rounds, 18 issues, 17 verified, 1 accepted, $15.10). The review workspace at `~/adr/casehub-drafthouse/document-workbench-20260705-224726/` is the first test dataset for the replay adapter (#95).

## The Document Workbench Direction

### The core insight
Documents pass through phases: **genesis** (blank page → Q&A → document), **refinement** (discuss and improve), **adversarial review** (formal debate with evidence), **implementation** (spec → code), **re-review** (incremental). Today each phase is a disconnected tool (brainstorming skill, Claude Code conversation, design-review skill, work-start/TDD). DraftHouse becomes the **continuous surface** where the document lives through all phases, preserving the conversation spine — *why* each section says what it says.

### Two systems, one abstraction
- **design-review** — headless Python orchestrator. Reviewer + implementor Claude sessions debate a spec. File-based (responses/reviewer-N.md, implementor-N.md, tracker.md). Proven at scale (~20 reviews overnight). Stays headless — it works. Not being replaced.
- **DraftHouse** — Quarkus server with Web Component panels. Debate event stream via Qhorus channels + WebSocket push. Review point tracking. Section highlighting.

Both build on the **P5 conversation protocol** (`ConversationState`, `ConversationProjection` in casehub-blocks). The adapter between them produces `DebateStreamEntry` objects from workspace files; consumers fold entries into `ConversationState`.

### Key design decisions made
1. **The adapter is permanent infrastructure**, not a temporary bridge. design-review stays file-based; the adapter translates at the boundary.
2. **Separate sessions per phase**, linked by document path (canonical file path as identity). Human-initiated phase transitions.
3. **Document timeline** is an abstraction over multiple versioning backends — git commits (adversarial review), session snapshots (genesis), explicit saves (refinement).
4. **DraftHouse panels are vanilla Shadow DOM** (not Lit). Extraction to `@casehubio/` packages will need a technology decision.
5. **UC1 (single-file review) and UC2 (A/B diff review) are the same abstraction** at different timescales — each review round creates an implicit before/after pair via git commits.

### Threads to preserve
- **Chunked vs batch orchestration (#97):** Currently agents respond to ALL issues in one batch (5-10 min). Explored whether priority-ordered chunking (address HIGH first, then MEDIUM) would give better UX and HIL affordance. Research issue — not decided yet.
- **HIL unlock (#100):** Channel-based human participation enables concurrent intervention at the issue level (vs design-review's stop-the-world DECISION_NEEDED). This changes what's *possible*, not just what's *visible*.
- **blocks-ui alignment:** blocks-ui provides reusable work-item components (`<work-item-inbox>`, `<work-item-workbench>`). The document workbench follows the same pattern — composable Lit/vanilla Web Components, npm packages under `@casehubio/`, embeddable via pages `hostPanel()`.
- **Claudony integration:** DraftHouse panels embed in Claudony for document work. DraftHouse becomes the document layer across all CaseHub apps. Any app that needs "discuss and refine this artifact" gets it through composable panels.
- **Publisher workflow (UC8):** The workbench isn't just for specs — blog editing, contract negotiation, clinical reports, any structured document work. Reusable component within any app, part of a case workflow.
- **design-review improvements (#96):** The skill should add structured metadata (LOCATION, PRIORITY, DEPENDS per issue), JSONL sidecar files, and implementor EVIDENCE markers. Benefits design-review independently and makes the adapter simpler.

## Immediate Next Step

**Start #94 — Research: map design-review output to DraftHouse data model.** This is pure research, no code changes. Read both systems' data models thoroughly and produce a field-by-field mapping. The exploration spec (§4, §6) has an initial mapping — #94 validates, corrects, and completes it against actual code.

Key files to read:
- design-review parser: `~/.claude/skills/design-review/parser.py`
- design-review tracker: `~/.claude/skills/design-review/tracker.py`
- P5 conversation protocol: casehub-blocks `io.casehub.blocks.conversation` package
- DraftHouse debate model: `server/api/src/main/java/io/casehub/drafthouse/debate/`
- DraftHouse review tracker panel: `server/runtime/src/main/webui/src/panels/drafthouse-review-tracker.js`
- Example workspace: `~/adr/casehub-drafthouse/document-workbench-20260705-224726/`

## What's Left

| # | Title | Scale | Complexity | Blocked by | Notes |
|---|-------|-------|------------|------------|-------|
| #93 | Document workbench (epic) | XL | High | — | 8 child issues |
| #94 | Research: data model mapping | M | High | — | **Start here** |
| #95 | Replay adapter | L | Med | #94 | First concrete integration |
| #96 | design-review structured output | M | Med | #94, #95 | Changes to soredium skill |
| #97 | Chunked orchestration research | M | High | #96 | Cost/UX tradeoff study |
| #98 | Document timeline | L | Med | #95 | Version navigation |
| #99 | Live workspace watching | M | Med | #95 | Real-time monitoring |
| #100 | Channel-based HIL | L | High | #97, #99 | Concurrent human participation |
| #101 | Panel extraction | XL | High | all above | @casehubio components |
| #92 | Adopt pages-push typed protocol SDK | S | Low | — | Independent |
| #89 | AgentProvider migration | M | Med | — | Independent |
| #84 | Brainstorming UI (epic) | L | High | — | #53 still open |
| #72 | Review pipeline orchestration | L | High | — | Design issue |
| #71 | Claude-to-Claude conversation protocol | L | High | — | Design issue |
| #61 | GraalVM native image | M | Med | — | Paused |
| #60 | Selection-scoped conversation channels | M | Med | — | |

## Build Note

The qhorus/ledger dependency is now stable. If it breaks again: the issue is `LedgerEntry` moved from `io.casehub.ledger.runtime.model` to `io.casehub.ledger.api.model` in ledger HEAD. Qhorus needs to absorb this change. Use `-nsu` flag on DraftHouse builds to avoid pulling broken SNAPSHOTs.

## References

| Context | Where |
|---|---|
| Exploration spec | `docs/superpowers/specs/2026-07-05-document-workbench-exploration.md` |
| Review workspace | `~/adr/casehub-drafthouse/document-workbench-20260705-224726/` |
| Epic | casehubio/drafthouse#93 |
| Architecture record | `ARC42STORIES.MD` |
| design-review skill | `~/claude/hortora/soredium/design-review/` |
| P5 conversation protocol | casehub-blocks `io.casehub.blocks.conversation` |
| blocks-ui components | `~/claude/casehub/blocks-ui/` |
