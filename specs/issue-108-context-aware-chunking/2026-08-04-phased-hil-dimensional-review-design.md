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

Each phase follows the same pattern: launch N review.py processes → watchdog
monitors progress.log → all complete → present results → human decides → next
phase.

Live progress (workspace-status) runs continuously across all phases.

### Degree Degradation

| Degree | Phase 1 | Phase 2 | Phase 3 |
|--------|---------|---------|---------|
| Light (1 round, no implementor) | Round 1 only | Skipped — nothing to continue | Optional cross-cutting |
| Standard (2-3 rounds) | Round 1 | Rounds 2-3 | Cross-cutting |
| Adversarial (4-6 rounds) | Round 1 | Rounds 2-6 | Cross-cutting |
| Deep (8-10 rounds) | Round 1 | Rounds 2-10 | Cross-cutting |

Light naturally collapses to two HIL points (round 1 results → cross-cutting
decision). No special-casing needed.

## Phase Orchestration

### Phase 1 — Round 1 Exploration

```bash
python3 review.py --spec X --title Y-coherence --type coherence --degree Z --max-rounds 1 --source-dirs ...
python3 review.py --spec X --title Y-structure --type structure --degree Z --max-rounds 1 --source-dirs ...
python3 review.py --spec X --title Y-robustness --type robustness --degree Z --max-rounds 1 --source-dirs ...
```

Three parallel background launches. Watchdog cron monitors for `ROUND 1 DONE`
(or `REVIEW DONE` for light degree) in each progress.log. When all three fire,
the watchdog presents results and pauses for HIL.

### Phase 2 — Depth Pursuit (surviving dimensions only)

```bash
python3 review.py --spec X --title Y-coherence --type coherence --degree Z --start-round 2 --resume-session <id> --source-dirs ...
python3 review.py --spec X --title Y-robustness --type robustness --degree Z --start-round 2 --resume-session <id> --source-dirs ...
```

Only dimensions the human accepted. `--start-round 2` and `--resume-session`
pick up from where phase 1 left off, reusing the same workspace and session.
Watchdog monitors for `REVIEW DONE`.

### Phase 3 — Cross-cutting (if approved)

```bash
python3 review.py --spec X --title Y-crosscutting --type crosscutting --degree Z --arch-files <trackers...> --source-dirs ...
```

Same as today. Launched only if the human says go at HIL point 3.

### New review.py Flags

- `--start-round N` — skip rounds 1 through N-1, begin the loop at round N
- `--resume-session <id>` — reuse the claude session from phase 1 for
  conversation continuity

The workspace directory, tracker.md, and response files persist across phases —
phase 2 reads phase 1's tracker and continues from that state.

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

Four actions available:
- **Accept all** — all dimensions continue to phase 2
- **Refuse all** — stop everything, skip to cross-cutting decision (or end)
- **Refuse subset** — kill selected dimensions (multi-select of dimensions to
  stop; others continue)
- **Discuss** — human asks about specific findings before deciding. The calling
  session reads the relevant tracker entries and discusses. After discussion,
  the same four options re-present.

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
| `dimension_start` | Phase 1 launch | dimension, degree, max_rounds |
| `round_start` | Each round begins | dimension, round_number, phase |
| `round_findings` | Reviewer completes | dimension, round_number, issue_count by priority |
| `round_end` | Implementor completes (or skipped) | dimension, round_number, addressed/contested counts |
| `dimension_paused` | Phase 1 round 1 complete | dimension, awaiting_hil: true |
| `dimension_resumed` | Phase 2 start | dimension, remaining_rounds |
| `dimension_killed` | Human refused | dimension, reason: "hil_refused" |
| `dimension_done` | All rounds complete | dimension, total_rounds, cost, issue_summary |
| `crosscutting_start` | Phase 3 launch | arch_files list |
| `crosscutting_done` | Phase 3 complete | findings_count, cost |

The workspace-status panel renders these as:

```
coherence   ██████░░░░  round 2/3  2 issues  $2.10
structure   ████████░░  round 3/4  5 issues  $3.40
robustness  ██████████  done       6 issues  $5.80  ✓
```

No new UI component needed — workspace-status consumes the richer event data.

## Session Continuity

### What Persists Across Phases

- The claude session (conversation history with reviewer/implementor agents)
- The workspace directory (tracker.md, response files, context.md)
- The issue state machine (which issues are OPEN, ADDRESSED, etc.)

### Mechanism

Phase 1 completes with `--max-rounds 1`. The session ID is written to a
`.session` file in the workspace. Phase 2 reads it back via
`--resume-session <id>`.

The workspace directory is the same across both phases — review.py is given the
same `--title`, so it resolves to the same
`~/reviews/{project}/{title}-{dimension}-{timestamp}/` path. tracker.md,
response files, and context.md are all in place when phase 2 starts.

`--start-round 2` tells review.py to skip the initial reviewer invocation for
round 1 (already done) and begin the loop at round 2. The tracker already has
round 1 data, so the round 2 reviewer sees the prior state naturally.

### Session Expiry Fallback

If the human takes a long time at the HIL checkpoint, the claude session may
expire. `--resume-session` would fail. Fallback: review.py detects the failure,
generates a handover from tracker.md + round 1 responses, and starts a fresh
session with that context. Same mechanism as the existing session-window
boundary, triggered by resume failure rather than round count.

## Changes Summary

| Component | What changes |
|-----------|-------------|
| **SKILL.md** | New three-phase orchestration flow. Three watchdog cycles instead of one. HIL prompt logic between phases. |
| **review.py** | Two new CLI flags: `--start-round N`, `--resume-session <id>`. Session ID written to `.session` file. Resume failure fallback. New JSONL events. |
| **setup.py** | No changes. |
| **prompts.py** | No changes. |
| **tracker.py** | No changes. |
| **parser.py** | No changes. |

### Not Changing

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
  implementor round. Superseded by the dimensional model for the HIL use case,
  but the flag remains available for backward compatibility.
- **WorkspaceWatcher:** Already watches progress.log and emits workspace-progress
  events to the browser. The new JSONL events extend this existing capability.

## Non-goals

- Changing the reviewer-implementor conversation mechanics within a round
- Adding new dimensions beyond the existing four
- Modifying degree presets or cost model
- Building a new UI component (workspace-status already exists)
- Cross-dimension context flow during review (dimensions stay isolated until
  cross-cutting)
