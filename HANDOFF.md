# Handover — 2026-07-19

**Branch:** `main` (#97 closed)

## What Happened This Session

Research: chunked orchestration vs batch for design-review (#97). Built
`--chunked` flag in soredium/design-review (review.py, tracker.py). Ran
head-to-head comparison on same spec — both modes find same core issues,
chunked produces deeper middle-tier fixes, costs 60% more per round.

Key soredium commits (on soredium main):
- `get_focus_items_by_priority()` in tracker.py (4 tests)
- `--chunked` flag + `_run_implementor_chunked()` in review.py (2 tests)

Also: `adr-status.py --costs` for per-round cost reporting (~/adr repo).

Two bugs found during comparison, not yet fixed:
1. Prompt filename scoping — implementor writes to standard filename, chunked code expects chunk-specific filename
2. Focus constraint ignored — HIGH chunk addresses all issues, not just HIGH

## Immediate Next Step

Pick from the backlog — all items are independent. The two chunked-mode bugs
should be fixed before piloting `--chunked` on real reviews.

## What's Next

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| #53 | Brainstorming UI slices 3-6 | S-M each | Low-Med | Slice 3: read-only panel, Slice 4: interactive injection, Slice 5: skill integration, Slice 6: convergence view |
| #99 | Live workspace watching | M | Med | Can consume JSONL events directly |
| #93 | Document workbench (epic) | XL | High | 5 remaining child issues |
| #100 | Channel-based HIL | L | High | Blocked by #99 |
| #101 | Panel extraction | XL | High | Blocked by all above |
| — | Fix chunked-mode bugs (soredium) | S | Low | Prompt filename + focus constraint — prerequisite for pilot |

## References

| Context | Where |
|---------|-------|
| Research report | specs/2026-07-18-chunked-orchestration-report.md |
| Cross-issue analysis | specs/2026-07-18-cross-issue-analysis.md |
| Blog entry | blog/2026-07-18-mdp27-chunking-the-implementor.md |
| Comparison workspaces | ~/adr/casehub-drafthouse/batch-comparison-*/, chunked-comparison-*/ |
| Soredium commits | ~/claude/hortora/soredium (design-review/review.py, tracker.py) |
