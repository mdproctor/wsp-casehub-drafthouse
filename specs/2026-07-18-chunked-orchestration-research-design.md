# Chunked Orchestration Research — Design Spec

**Issue:** casehubio/drafthouse#97
**Date:** 2026-07-18
**Status:** Research design

## Problem

Design-review orchestration is batch: the reviewer raises all issues in one pass,
then the implementor addresses all of them in one response. Each turn takes 5-10
minutes. The human sees nothing until the full implementor response arrives and
cannot intervene mid-batch.

**Hypothesis:** Priority-ordered chunked responses would give better UX (shorter,
more frequent updates), better HIL (intervene before low-priority issues are
addressed), and potentially lower cost (early termination when high-impact issues
are resolved).

## Research Questions

1. **Cost:** Does chunking cost more (multiple invocations) or less (smaller per-invocation, early termination)?
2. **Quality:** Does the implementor lose cross-issue pattern recognition when it only sees a priority slice?
3. **Reviewer efficiency:** Does chunking affect the reviewer's verification workload?
4. **Orchestration complexity:** What changes to review.py?
5. **UX:** How does batch arrival compare to chunked arrival in DraftHouse?

## Research Structure

Three phases, each building on the previous:

### Phase 1 — Retrospective Analysis (no API cost)

**1a. Cost baselines from all 10 existing drafthouse reviews.**

Extract per-round reviewer cost, implementor cost, and timing from all
`~/adr/casehub-drafthouse/*/progress.log` files. Build a cost model:

| Metric | Source |
|--------|--------|
| Reviewer cost per round | progress.log `Reviewer done ($N)` |
| Implementor cost per round | progress.log `Implementor done ($N)` |
| Cost taper over rounds | cumulative cost / round number |
| Total review cost | final cumulative |
| Issue count vs cost | tracker.md issue count vs total cost |

Expected baseline from initial data: ~$2/reviewer, ~$2-3/implementor in early
rounds, tapering to ~$1 each as issues converge.

**1b. Cross-issue pattern analysis from brainstorming-ui review.**

This review has priority data: 7 HIGH, 12 MEDIUM. Read the implementor's
round 1-2 responses and classify:

- **Cross-priority references:** Did the implementor reference MEDIUM issues when
  fixing HIGH issues?
- **Batched fixes:** Were any fixes applied to multiple issues simultaneously?
- **Pattern dependency:** Would a chunked implementor seeing only HIGH items have
  produced a different (worse) fix?

This tells us whether cross-issue pattern loss is a real risk or a theoretical
concern before spending money on the experiment.

### Phase 2 — Fork-after-reviewer Experiment (~$10-15 API cost)

**Spec selection:** An unreviewed spec with enough surface area to generate 15+
issues across priority levels. Candidates: an open drafthouse spec (e.g., #99
live workspace watching), or a purpose-built spec for the experiment.

**Experiment protocol:**

1. Run the reviewer once on the chosen spec (standard review.py, round 1 only)
   → produces issue list with priorities
2. Save spec state and reviewer output as common starting point (git tag)
3. **Batch run:** invoke implementor with ALL focus items (standard mode)
   → capture: cost, wall-clock time, response content
4. **Reset spec** to tagged state (git checkout)
5. **Chunked run:** invoke implementor 2-3 times — HIGH items first, then
   MEDIUM, then LOW — using same reviewer output and tracker
   → capture: cost per chunk, wall-clock time per chunk, response content

**Measurements:**

| Metric | Batch | Chunked |
|--------|-------|---------|
| Total implementor cost | Single invocation | Sum of chunks |
| Time to first update | Full batch latency | First chunk latency |
| Cross-issue connections | Count of cross-references | Count of cross-references |
| Redundant work | N/A | Fixes repeated across chunks |
| Coverage | Issues addressed / total | Issues addressed / total |
| Quality assessment | Qualitative read | Qualitative read |

**Controls:**
- Same reviewer output for both runs (fork after reviewer)
- Same spec version (git tag)
- Same model, same budget, same effort level
- One round only — multi-round convergence is out of scope

### Phase 3 — Orchestration Design Sketch

Based on Phase 1+2 findings, produce:

**If chunking is recommended:**

Concrete sketch of review.py changes:

- After reviewer runs, partition `tracker.get_focus_items()` by priority
  (HIGH → MEDIUM → LOW)
- Replace the single implementor invocation (review.py lines 611-691) with a
  loop over priority chunks
- Each chunk invocation uses `build_implementor_prompt()` with filtered focus
  items plus a note that the full tracker is available for cross-issue context
- Between chunks: write JSONL events for DraftHouse incremental progress
- Optional HIL checkpoint between chunks (configurable flag)
- Early termination: human can skip remaining LOW items → mark DEFERRED

DraftHouse integration assessment:
- WorkspaceParser: already handles per-round JSONL with priority metadata
- WorkspaceReplayAdapter: already passes priority to channel message meta
- Channel-feed panel: already groups by round — chunks appear as incremental
  updates within the same round group
- Expected DraftHouse changes: zero

Tracker changes: none — `TrackedIssue.priority` already exists,
`get_focus_items()` returns all non-terminal items, chunking logic filters
externally.

**If batch is recommended:** document why, with the evidence.

**If hybrid is recommended:** specify which parts chunk and which don't.

**Open questions deferred to future work:**
- How chunking interacts with session windows (reset boundaries)
- Whether the reviewer should be told about chunk boundaries
- Multi-round convergence effects (would need a longer experiment)

## Output

A research report (markdown) containing:

1. Cost model tables (Phase 1a)
2. Cross-issue pattern assessment (Phase 1b)
3. Experiment data tables and qualitative comparison (Phase 2)
4. Recommendation: chunk, keep batch, or hybrid — with specific rules
5. If chunking recommended: orchestration change sketch (Phase 3)

## Acceptance Criteria

From the issue:

- [ ] At least one side-by-side comparison: same spec reviewed with batch vs
      chunked orchestration (Phase 2)
- [ ] Cost analysis: total tokens, total cost, total time for each approach
      (Phase 1a + Phase 2)
- [ ] Quality analysis: did chunking miss cross-issue patterns? Did batch waste
      time on trivial issues? (Phase 1b + Phase 2)
- [ ] Written recommendation: chunk, keep batch, or hybrid (Phase 3)
- [ ] If chunking recommended: sketch of orchestration changes to review.py
      (Phase 3)

## Non-goals

- Actually implementing the chunked orchestration in review.py (that's a
  separate issue if recommended)
- Running multiple experiments for statistical significance
- Multi-round convergence testing
- DraftHouse UI changes
