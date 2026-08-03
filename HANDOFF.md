# Handover — 2026-08-03

**Branch:** `main` (#101 closed, #113 closed)

## What Happened This Session

Fixed #113 (stale ledger CDI bean exclusion — workaround removed, root cause
already fixed in ledger's reactive tier retirement). Filed and closed
casehubio/ledger#182.

Implemented #101 — extracted all 9 DraftHouse Lit panels to
`@casehubio/blocks-ui-document-workbench`. Convergence analysis confirmed
neither `blocks-channel-feed` nor `blocks-timeline` overlap with the
drafthouse panels. Panels adapted with pages CSS tokens, `apiBaseUrl`
property, `channel-feed` → `debate-feed` rename. Showcase gallery added
to blocks-ui examples. DraftHouse migrated to consumer via esbuild alias.
578 E2E tests green, 31 vitest tests in blocks-ui.

Garden entry: GE-20260803-1f9860 — jsdom/ResizeObserver gotcha for Lit
component testing in vitest.

## Follow-up

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| #93 | Document workbench (epic) | XL | High | All 8 children now closed — epic ready to close |
| — | design-review skill reads decisions/ | M | Med | soredium change — agents read human decisions at round start |
| — | Unrecovered specs on 14 closed branches | S | Low | Hygiene scan flagged `research.md` files; cherry-pick if needed |

## References

| Context | Where |
|---------|-------|
| Blog entry | blog/2026-08-03-mdp31-panels-leave-home.md |
| Design spec | docs/specs/issue-101-extract-document-panels/2026-07-30-document-workbench-extraction-design.md |
| blocks-ui commits | casehubio/blocks-ui main (5 commits, document-workbench package) |
