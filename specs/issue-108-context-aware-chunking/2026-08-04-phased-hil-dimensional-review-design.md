# Phased HIL for Dimensional Reviews — Design Spec

**Issue:** casehubio/drafthouse#108
**Date:** 2026-08-04
**Status:** Design approved

## Problem

Dimensional reviews (coherence, structure, robustness + cross-cutting) run all
dimensions to completion before the human sees anything actionable. At standard
degree (2-3 rounds per dimension, 3 parallel dimensions + cross-cutting), the
human sits idle for ~20 minutes. The original chunking research (#97) aimed to
solve this by splitting the implementor's work by priority tier, but the
architecture has since moved to dimensional reviews — making priority-tier
chunking obsolete.

The root motivation remains: give the human meaningful engagement points during
the review, not just a progress bar.

## Solution

A three-phase pipeline with human decision gates between each phase. The key
insight: round 1 of each dimension is the cheapest moment to learn whether that
dimension has anything useful to say. Killing an unproductive dimension after
round 1 saves all remaining rounds at that degree.

## Architecture

```
Degree ──► Phase 1 (round 1) ──► HIL ──► Phase 2 (remaining rounds) ──► HIL ──► Phase 3 (cross-cutting) ──► HIL
              │                    │           │                          │           │                       │
              ├─ coherence r1      │           ├─ coherence r2..N         │           └─ cross-cutting        └─ final
              ├─ structure r1      │           └─ robustness r2..N        │              reads trackers          results
              └─ robustness r1     │                                      │
                                   │                                      │
                              accept/refuse/                         go/skip
                              discuss per                            cross-cutting
                              dimension
```

Dimensions launch at the selected degree and run their full round loop — exactly
as before. A single watchdog cron monitors progress and provides two HIL
checkpoints:

1. **Round 1 checkpoint** — after all dimensions complete round 1, the watchdog
   presents findings and the human can kill dimensions wasting time. Surviving
   dimensions keep running — they were never stopped.
2. **Completion checkpoint** — after all survivors finish, the human decides
   whether to run cross-cutting.

The dimensions are not stopped and restarted. No `--max-rounds`, no workspace
resume, no degree changes. The existing review.py round loop is untouched.

### Degree Behavior

| Degree | Rounds | Round 1 checkpoint | Completion checkpoint |
|--------|--------|-------------------|----------------------|
| Light | 1 round, reviewer-only | Findings shown, dimensions already done | Cross-cutting go/skip |
| Standard | 2-3 rounds | Findings shown, kill/accept/discuss | Cross-cutting go/skip |
| Adversarial | 4-6 rounds | Findings shown, kill/accept/discuss | Cross-cutting go/skip |
| Deep | 8-10 rounds + ultrathink | Findings shown, kill/accept/discuss | Cross-cutting go/skip |

At light degree, dimensions exit after round 1, so the round 1 checkpoint
and completion checkpoint collapse — the human sees findings and decides
on cross-cutting in one step.

## Phase Orchestration

### Launch

```bash
python3 review.py --spec X --title Y-coherence --type coherence --degree Z --source-dirs ...
python3 review.py --spec X --title Y-structure --type structure --degree Z --source-dirs ...
python3 review.py --spec X --title Y-robustness --type robustness --degree Z --source-dirs ...
```

Three parallel background launches at the selected degree. Identical to the
existing launch commands — no changes.

### Round 1 Checkpoint (watchdog-driven)

A watchdog cron monitors progress.log for `round_end` events with
`round_number: 1`. When all three dimensions have completed round 1, the
watchdog reads each tracker.md and presents findings.

At light degree, dimensions exit after round 1 — the checkpoint fires when
the processes complete. At other degrees, dimensions keep running through
round 2+ while the human reviews round 1 findings. The human can kill
unproductive dimensions; survivors continue uninterrupted.

### Completion Checkpoint (watchdog-driven)

When all surviving dimensions show `REVIEW DONE`, the watchdog presents
full results and offers a cross-cutting go/skip decision. Cross-cutting
launches only if approved.

### No New review.py Flags

No changes to review.py's CLI interface. The round loop, degree presets,
session management, and workspace structure are all untouched. The only
review.py change is JSONL event emission (Task 1) — the watchdog uses
these events for round-level visibility.

## HIL Interaction Model

The calling session (the Claude that launched the reviews) handles all human
interaction. The human never talks directly to review.py.

### After Phase 1 — Round 1 Findings

The watchdog reads each dimension's tracker.md and presents a summary:

```
Round 1 findings:

  Coherence:  3 issues (1 HIGH, 2 MEDIUM)
  Structure:  0 issues — nothing found
  Robustness: 5 issues (2 HIGH, 1 MEDIUM, 2 LOW)
```

Dimensions with zero issues are flagged — they found nothing useful.

Four actions available (at non-light degrees — dimensions are still running):
- **Accept all** — all dimensions continue running (no action taken)
- **Refuse all** — kill all dimension processes, skip to completion checkpoint
- **Refuse subset** — kill selected dimension processes (survivors keep running
  — they were never stopped)
- **Discuss** — human asks about specific findings before deciding. The calling
  session reads the relevant tracker entries and discusses. After discussion,
  the same four options re-present.

At light degree, dimensions are already done (1 round) — the round 1 checkpoint
and completion checkpoint collapse into one.

### After Phase 2 — Pre-cross-cutting

Summary with full round data:

```
Dimension results:

  Coherence:  3 rounds, 4 issues (2 verified, 1 accepted, 1 deferred) — $4.20
  Robustness: 4 rounds, 6 issues (3 verified, 2 accepted, 1 contested) — $5.80
  Structure:  killed after round 1
```

Two actions:
- **Run cross-cutting** — launch phase 3 with surviving dimension trackers
- **Skip** — done, present final results from dimensions only

### After Phase 3 — Final Results

Unified results table across all dimensions plus cross-cutting. No action
options — the human reads and decides what to act on in the spec.

## Live Progress Events

Between HIL checkpoints, the human sees real-time progress via DraftHouse's
workspace-status panel. review.py emits JSONL events that WorkspaceWatcher
consumes.

### New/Enhanced Events

| Event | When | Payload |
|-------|------|---------|
| `dimension_start` | Process launch | dimension, degree, phase (1 or 2) |
| `round_start` | Each round begins | dimension, round_number |
| `round_findings` | Reviewer completes | dimension, round_number, issue_count by priority |
| `round_end` | Implementor completes (or skipped) | dimension, round_number, addressed/contested counts |
| `dimension_done` | Process exits normally | dimension, total_rounds, cost, issue_summary |
| `crosscutting_start` | Phase 3 launch | arch_files list |
| `crosscutting_done` | Phase 3 complete | findings_count, cost |

Phase transitions (paused/resumed/killed) are managed by the calling session,
not by review.py events. The calling session tracks which dimensions were
accepted or refused at HIL and only launches phase 2 for accepted ones.

The workspace-status panel renders these as:

```
coherence   ██████░░░░  round 2/3  2 issues  $2.10
structure   ████████░░  round 3/4  5 issues  $3.40
robustness  ██████████  done       6 issues  $5.80  ✓
```

No new UI component needed — workspace-status consumes the richer event data.

## Session Continuity

No session continuity mechanism is needed. Dimensions launch once and run
their full round loop at the selected degree — they are never stopped and
restarted. The watchdog monitors progress.log for round-level events and
presents HIL checkpoints while dimensions continue running.

The only "interruption" is killing a dimension process when the human
refuses it at the round 1 checkpoint. This is a clean termination, not a
pause-and-resume.

## Changes Summary

| Component | What changes |
|-----------|-------------|
| **SKILL.md** | Watchdog cron enhanced with two checkpoints (round 1 HIL + pre-cross-cutting gate). Launch commands and step structure preserved. |
| **review.py** | New JSONL events for dimension lifecycle (`dimension_start`, `round_findings`, `round_end`, `dimension_done`). No CLI changes. |
| **setup.py** | No changes. |
| **prompts.py** | No changes. |
| **tracker.py** | No changes. |
| **parser.py** | No changes. |

No new CLI flags. No changes to review.py's round loop, degree presets,
session management, or workspace structure. The entire change is: events
in review.py + smarter watchdog in SKILL.md.

### Not Changing

- The launch commands (same degree, same flags)
- The reviewer-implementor loop within a round
- Dimension-specific prompts and briefs
- The cross-cutting mechanism (reads tracker.md files via --arch-files)
- Degree presets (round counts, budgets, ultrathink)
- The workspace directory structure

## Prior Art

- **#97 research:** Chunked orchestration research established that cross-issue
  coupling is rare (5%, incidental) and that narrower scope produces deeper
  analysis. The phased model builds on this finding — round 1 isolation is safe
  because issues rarely depend on each other across priority tiers or dimensions.
- **Existing `--chunked` flag:** Priority-tier chunking within a single
  implementor round. Superseded by the phased dimensional model for the HIL
  use case. Can be removed in a follow-up cleanup.
- **WorkspaceWatcher:** Already watches progress.log and emits workspace-progress
  events to the browser. The new JSONL events extend this existing capability.

## Non-goals

- Changing the reviewer-implementor conversation mechanics within a round
- Adding new dimensions beyond the existing four
- Modifying degree presets or cost model
- Building a new UI component (workspace-status already exists)
- Cross-dimension context flow during review (dimensions stay isolated until
  cross-cutting)
