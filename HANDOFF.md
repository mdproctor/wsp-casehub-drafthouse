# Handover — 2026-08-03

**Branch:** `main` (#109, #110, #101, #113 closed this session)

## What Happened This Session

Four issues closed across two branches:

**Branch 1 (issue-101):** Extracted all 9 DraftHouse Lit panels to
`@casehubio/blocks-ui-document-workbench` with showcase gallery. DraftHouse
migrated to consumer. Epic #93 closed (all 8 children done). #84 and #106
also closed (completed/superseded).

**Branch 2 (issue-109):** #109 investigated — chunk boundary awareness adds
no benefit (reviewer already assigns priorities that drive chunking). #110
implemented — WorkspaceWatcher now dispatches DEFERRED entries and evidence
MEMOs via tracker diffing, plus ROUND_SNAPSHOT when spec commit found.

Garden entry: GE-20260803-1f9860 (jsdom/ResizeObserver gotcha).

## Follow-up

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| #108 | Explore context-aware chunking | M | High | Was blocked by #109 (now closed) |
| #72 | Review pipeline orchestration | L | High | Sequential multi-perspective reviews |
| #71 | Claude-to-Claude conversation protocol | L | High | Multi-turn agent dialogue |
| #61 | GraalVM native image build | M | Med | Paused |
| #60 | Selection-scoped conversation channels | M | Med | Persistent per-selection threads |
| — | design-review skill reads decisions/ | M | Med | soredium change |

## References

| Context | Where |
|---------|-------|
| Blog (panels) | blog/2026-08-03-mdp31-panels-leave-home.md |
| blocks-ui package | casehubio/blocks-ui components/document-workbench/ |
| Design spec (#101) | docs/specs/issue-101-extract-document-panels/ |
