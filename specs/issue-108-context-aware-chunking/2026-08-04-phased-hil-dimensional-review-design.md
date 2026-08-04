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

Each phase follows the same pattern: launch N review.py processes as background
tasks → calling session receives completion notifications → present results →
human decides → next phase.

A cron watchdog runs as a fallback monitor for stall detection and failure
handling, but phase transitions are driven by background task completion
notifications in the calling session — not by the cron.

Live progress (workspace-status) runs continuously across all phases.

### Degree Degradation

Phase 1 runs at the selected degree with `--max-rounds 1`. This preserves the
full budget and ultrathink settings — round 1 is where issues are found, so it
gets the quality the user asked for. At non-light degrees, the implementor runs
too, giving the human richer context at the HIL gate ("5 found, 3 addressed, 2
contested" vs just "5 found"). Phase 2 resumes at the same degree for remaining
rounds.

| Degree | Phase 1 | Phase 2 | Phase 3 |
|--------|---------|---------|---------|
| Light (1 round, no implementor) | Round 1 reviewer-only | Skipped — nothing to continue | Optional cross-cutting |
| Standard (2-3 rounds) | Round 1 (reviewer + implementor) | Resume, rounds 2-3 | Cross-cutting |
| Adversarial (4-6 rounds) | Round 1 (reviewer + implementor) | Resume, rounds 2-6 | Cross-cutting |
| Deep (8-10 rounds) | Round 1 (reviewer + implementor, ultrathink) | Resume, rounds 2-10 | Cross-cutting |

Light naturally collapses to two HIL points (round 1 results → cross-cutting
decision). No special-casing needed — phase 1 IS the entire review at light.

## Phase Orchestration

### Phase 1 — Round 1 Exploration

```bash
python3 review.py --spec X --title Y-coherence --type coherence --degree Z --max-rounds 1 --source-dirs ...
python3 review.py --spec X --title Y-structure --type structure --degree Z --max-rounds 1 --source-dirs ...
python3 review.py --spec X --title Y-robustness --type robustness --degree Z --max-rounds 1 --source-dirs ...
```

Three parallel background launches. Uses the selected degree (`Z`) with
`--max-rounds 1` to cap at one round while preserving the full budget and
ultrathink settings. At light degree, the implementor is skipped (existing
behavior). At other degrees, the implementor runs after the reviewer,
giving richer findings context at the HIL gate.

The calling session receives background task completion notifications. When all
three have completed, it reads each workspace's tracker.md and presents results
for HIL.

The calling session captures the workspace path from each process's progress.log
(printed at startup as `Review (<type>): <workspace-path>`). These paths are
passed explicitly to phase 2 — no title-based resolution needed.

### Phase 2 — Depth Pursuit (surviving dimensions only)

```bash
python3 review.py --workspace <coherence-workspace-path> --degree Z --source-dirs ...
python3 review.py --workspace <robustness-workspace-path> --degree Z --source-dirs ...
```

Only dimensions the human accepted. Uses `--workspace <path>` to resume from
phase 1's workspace — review.py's existing resume mechanism detects the
completed round from tracker state and continues from round 2. The degree
is already saved from phase 1 (which used the selected degree), so resume
picks it up automatically.

Session continuity is handled internally by review.py — the calling session
does not manage session IDs. If the prior session expired (human took a long
break at the HIL checkpoint), review.py's existing session-window mechanism
generates a handover and starts a fresh session.

### Phase 3 — Cross-cutting (if approved)

```bash
python3 review.py --spec X --title Y-crosscutting --type crosscutting --degree Z --arch-files <trackers...> --source-dirs ...
```

Same as today. Launched only if the human says go at HIL point 3. The
`--arch-files` paths are the tracker.md files from each surviving dimension's
workspace (captured in phase 1, filtered by HIL decisions).

### No New review.py Flags

The phased model requires no new CLI flags. It uses existing mechanisms:

- `--degree light` — phase 1 runs reviewer-only (already exists)
- `--workspace <path>` — phase 2 resumes from phase 1's workspace (already exists)
- `--degree Z` on resume — overrides the degree for remaining rounds

The workspace directory, tracker.md, and response files persist across phases —
phase 2 reads phase 1's tracker and continues from that state. review.py manages
reviewer and implementor session IDs internally.

## HIL Interaction Model

The calling session (the Claude that launched the reviews) handles all human
interaction. The human never talks directly to review.py.

### After Phase 1 — Round 1 Findings

The calling session reads each dimension's tracker.md and presents a summary:

```
Round 1 findings:

  Coherence:  3 issues (1 HIGH, 2 MEDIUM)
  Structure:  0 issues — nothing found
  Robustness: 5 issues (2 HIGH, 1 MEDIUM, 2 LOW)
```

Dimensions with zero issues are flagged — they found nothing to iterate on and
are candidates for automatic exclusion from phase 2.

Four actions available (only if selected degree is not light — light has no
phase 2):
- **Accept all** — all dimensions with findings continue to phase 2
- **Refuse all** — stop everything, skip to cross-cutting decision (or end)
- **Refuse subset** — kill selected dimensions (multi-select of dimensions to
  stop; others continue)
- **Discuss** — human asks about specific findings before deciding. The calling
  session reads the relevant tracker entries and discusses. After discussion,
  the same four options re-present.

At light degree, phase 1 IS the complete review — the HIL presents findings
and proceeds directly to the pre-cross-cutting gate.

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

### What Persists Across Phases

- The workspace directory (tracker.md, response files, context.md)
- The issue state machine (which issues are OPEN, ADDRESSED, etc.)
- Claude session IDs (managed internally by review.py, not by the calling session)

### Mechanism

The calling session captures each dimension's workspace path from phase 1's
progress.log output. Phase 2 passes this path via `--workspace <path>` —
no title-based resolution or timestamp matching needed.

review.py's existing `--workspace` resume mechanism:
1. Reads tracker.md to determine the last completed round
2. Recovers reviewer and implementor session IDs from internal state
3. Continues the round loop from the next round
4. If a session has expired, falls back to the existing session-window
   handover mechanism (generates handover from tracker + responses, starts
   fresh)

The calling session never manages session IDs. It only passes workspace paths
and the selected degree.

## Changes Summary

| Component | What changes |
|-----------|-------------|
| **SKILL.md** | New three-phase orchestration flow. Phase transitions driven by background task notifications. Single fallback cron for stall/failure monitoring. HIL prompt logic between phases. |
| **review.py** | New JSONL events for dimension lifecycle. No degree override needed — phase 1 uses the selected degree with `--max-rounds 1`, so the saved `.depth` already matches. |
| **setup.py** | No changes. |
| **prompts.py** | No changes. |
| **tracker.py** | No changes. |
| **parser.py** | No changes. |

No new CLI flags are introduced. The phased model uses existing `--degree`,
`--workspace`, and `--max-rounds` flags.

### Not Changing

- The reviewer-implementor loop within a round
- Dimension-specific prompts and briefs
- The cross-cutting mechanism (reads tracker.md files via --arch-files)
- Degree presets (round counts, budgets, ultrathink)
- The workspace directory structure
- Session management internals in review.py

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
