# Chunked Orchestration Research — Design Spec

**Issue:** casehubio/drafthouse#97
**Date:** 2026-07-18
**Status:** Research design

## Problem

Design-review orchestration is batch: the reviewer raises all issues in one pass,
then the implementor addresses all of them in one response. This treats every issue
equally — HIGH and LOW items get processed in the same invocation, with no
opportunity for human control between priority tiers.

The human cannot:
- See HIGH-priority results before the full batch completes (5-10 minutes)
- Decide after seeing HIGH fixes whether MEDIUM/LOW items are worth the compute
- Skip lower-priority items to save cost or redirect effort

These are not three independent problems with independent solutions — they share a
root cause: batch processing ignores priority ordering and offers no mid-process
control point.

**Hypothesis:** Priority-ordered chunked invocations (HIGH → MEDIUM → LOW) would
give the human mid-process control: see important results sooner, intervene between
priority tiers, and optionally skip lower-priority work.

**Prior art:** The timeout-retry mechanism (review.py lines 644-680) already
implements partial-batch retry — when the implementor times out, the PM identifies
addressed items, pre-applies them, and retries with only the missing items. This
demonstrates the orchestration plumbing works for multiple invocations with reduced
scope, but it is failure-driven (random scope reduction after timeout) rather than
design-driven (intentional priority ordering with human control).

## Research Questions

1. **Cross-issue coupling (go/no-go gate):** Does the implementor reference issues
   across priority tiers? Would a chunked implementor seeing only its priority
   slice produce different (worse) fixes?
2. **Cost model:** What does each implementor invocation cost as a function of
   issue count? Can we estimate chunked cost from existing batch data?
3. **Orchestration complexity:** What review.py changes are needed, and how much
   of the existing timeout-retry path can be reused for intentional chunking?

## Research Structure

### Phase 1a — Cost Baselines (operational tooling, no API cost)

Extract per-round reviewer cost, implementor cost, and timing from all
`~/adr/casehub-drafthouse/*/progress.log` files. Build into `adr-status.py`
as a cost reporting feature — useful regardless of the chunking decision.

| Metric | Source |
|--------|--------|
| Reviewer cost per round | progress.log `Reviewer done ($N)` |
| Implementor cost per round | progress.log `Implementor done ($N)` |
| Cost taper over rounds | cumulative cost / round number |
| Total review cost | final cumulative |
| Issue count vs cost | tracker.md issue count vs total cost |

### Phase 1b — Cross-Issue Pattern Analysis (go/no-go gate)

The brainstorming-ui review has priority data: 7 HIGH, 12 MEDIUM. Read the
implementor's round 1-2 responses and classify:

- **Cross-priority references:** Did the implementor reference MEDIUM issues when
  fixing HIGH issues?
- **Batched fixes:** Were any fixes applied to multiple issues simultaneously?
- **Pattern dependency:** Would a chunked implementor seeing only HIGH items have
  produced a different (worse) fix?

**Decision gate:** If cross-issue coupling is frequent and quality-affecting,
chunking has a real quality risk — stop here and document why batch is better.
If coupling is rare or cosmetic, proceed to Phase 2.

### Phase 2 — Build `--chunked` Mode (if Phase 1b green-lights)

Build chunking behind a `--chunked` flag in review.py:

- After reviewer runs, partition `tracker.get_focus_items()` by priority
  (HIGH → MEDIUM → LOW)
- Replace the single implementor invocation with a loop over priority chunks
- Each chunk invocation uses `build_implementor_prompt()` with filtered focus
  items (no cross-chunk context — the simplest design, and the one that directly
  tests whether cross-issue context loss matters in practice)
- Between chunks: write JSONL events for DraftHouse incremental progress
- Optional HIL checkpoint between chunks (configurable)
- Early termination: human can skip remaining chunks → mark items DEFERRED

Leverage the existing timeout-retry mechanism (lines 644-680) which already
handles partial progress persistence and retry with reduced scope.

### Phase 3 — Pilot and Decide

Run `--chunked` on the next 3-5 real reviews alongside batch baselines from
Phase 1a. Collect:

| Metric | Batch (Phase 1a) | Chunked (pilot) |
|--------|-------------------|------------------|
| Total implementor cost | From baselines | Per-pilot |
| Time to first update | Full batch latency | First chunk latency |
| Cross-issue connections | N/A | Cross-references in later chunks |
| Coverage | Issues addressed / total | Issues addressed / total |
| Early terminations | N/A | Times human skipped chunks |

**Decision:** Chunk, keep batch, or hybrid — based on real-world data across
diverse specs and priority distributions.

**Open questions deferred to future work:**
- How chunking interacts with session windows (reset boundaries)
- Whether the reviewer should be told about chunk boundaries
- Multi-round convergence effects

## Output

A research report (markdown) containing:

1. Cost model tables (Phase 1a)
2. Cross-issue pattern assessment with go/no-go decision (Phase 1b)
3. If proceed: pilot data tables and comparison (Phase 3)
4. Recommendation: chunk, keep batch, or hybrid — with evidence

## Acceptance Criteria

- [ ] Cross-issue pattern analysis on at least one review with priority data
      (Phase 1b)
- [ ] Cost baselines from existing reviews (Phase 1a)
- [ ] If Phase 1b green-lights: working `--chunked` flag in review.py (Phase 2)
- [ ] If Phase 2 built: pilot data from at least 3 real reviews (Phase 3)
- [ ] Written recommendation with evidence

## Non-goals

- Controlled side-by-side experiment on a single spec (pilot data across
  diverse specs is more informative)
- Statistical significance testing
- Multi-round convergence testing
- DraftHouse UI changes
