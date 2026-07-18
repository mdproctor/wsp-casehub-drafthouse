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

## Current State

The `--chunked` flag and `_run_implementor_chunked()` already exist in review.py
(lines 250-375, flag at line 1595, dispatch at line 764). The implementation
partitions focus items by priority, loops over chunks, writes per-chunk markdown
files, emits JSONL events, and includes a human-in-the-loop checkpoint between
priority tiers.

Known issues in the current implementation:

1. **Resume broken:** `_rebuild_tracker()` (line 1299) reads only
   `implementor-{rn}.md` (line 1325). Chunked mode writes
   `implementor-{rn}-chunk-{idx}.md` files (line 283). No combined file is
   synthesized, so resume after a chunked round loses all implementor responses.

2. **No evidence verification:** Batch mode verifies FIXED claims against git
   diffs via `_verify_evidence()` in Step 4 (lines 894-960). Chunked mode skips
   verification entirely — the "Fall through to termination checks" comment at
   line 777 bypasses Step 4.

3. **No timeout-retry:** Batch mode retries on timeout with only missing items
   (lines 814-848). Chunked mode aborts the entire run on any chunk failure
   (line 293).

4. **Silent failure on mid-chunk crash:** Items in a failed chunk stay OPEN with
   no indication they were attempted. No DEFERRED marking, no failure context in
   the tracker.

5. **DraftHouse parser gap:** `WorkspaceParser.parseRoundFromJsonl` handles core
   events (`issue_fixed`, `issue_rejected`, etc.) correctly because `_write_jsonl`
   (line 770) combines all chunk events into the standard `implementor-{rn}.jsonl`.
   However: the markdown fallback path (`parseRoundFromMarkdown`) reads
   `implementor-{rn}.md` which doesn't exist in chunked mode, and
   `chunk_start`/`chunk_end` JSONL events are emitted but silently ignored by the
   parser's switch statement.

6. **No test coverage:** The soredium test suite (121 tests) has zero tests for
   `_run_implementor_chunked`, the `--chunked` flag, or any chunked-mode edge
   cases.

Phase 2 of this research validates and hardens the existing implementation rather
than building from scratch.

## Research Questions

1. **Cross-issue coupling (go/no-go gate):** Does the implementor reference issues
   across priority tiers? Would a chunked implementor seeing only its priority
   slice produce different (worse) fixes?
2. **Cost model:** What does each implementor invocation cost as a function of
   issue count? Can we estimate chunked cost from existing batch data?
3. **Orchestration complexity:** What review.py changes are needed to harden the
   existing chunked implementation, and how much of the existing timeout-retry
   path can be reused?
4. **Existing implementation correctness:** Does the current `--chunked`
   implementation produce correct results despite the known bugs? Which bugs
   must be fixed before pilot data collection?

## Alternatives Considered

### Priority ordering within a single invocation

Order `focus_items` by priority within `build_implementor_prompt` so HIGH items
are listed first, MEDIUM second, LOW last. Combined with the existing
timeout-retry mechanism, this gives priority ordering and timeout-aware priority
without multi-invocation complexity.

**Why this is insufficient:** This addresses priority ordering but not mid-process
human control — the primary motivation for chunked orchestration. The human still
cannot see HIGH results before the full batch completes, decide after seeing HIGH
fixes whether to skip MEDIUM/LOW, or terminate early to save cost.

It also relies on the agent addressing items in presentation order, which is not
guaranteed. The existing timeout-retry mechanism catches timeouts, but scope
reduction is random (whatever was addressed first), not priority-aware.

This alternative is worth noting as a zero-cost improvement that can be applied
independently of chunking — priority-ordered prompt construction is good practice
regardless. But it does not solve the control problem that motivates this research.

## Research Structure

### Phase 1a — Cost Baselines (operational tooling, no API cost)

Per-round cost and timing data already exists in `progress.log` files as
`Reviewer done ($N)` / `Implementor done ($N)` lines. Phase 1a parses and
aggregates this existing data into `adr-status.py` as a cost reporting feature.

| Metric | Source | Status |
|--------|--------|--------|
| Reviewer cost per round | progress.log `Reviewer done ($N)` | Exists in logs, needs parsing |
| Implementor cost per round | progress.log `Implementor done ($N)` | Exists in logs, needs parsing |
| Cost taper over rounds | Computed from above | Needs aggregation |
| Total review cost | Computed from above | Needs aggregation |
| Issue count vs cost | tracker.md issue count vs total cost | Needs correlation |

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

### Phase 2 — Validate and Harden `--chunked` Mode (if Phase 1b green-lights)

The `--chunked` flag exists but has known bugs (see Current State). Phase 2
validates the existing implementation and fixes the identified issues:

**Resume synthesis:** After all chunks complete, synthesize a combined
`implementor-{rn}.md` from the per-chunk files. This makes chunked rounds
produce the same file structure as batch rounds, keeping `_rebuild_tracker()`
and the `WorkspaceParser` markdown fallback working unchanged.

**Evidence verification:** Run the same evidence verification (Step 4) on chunked
output that batch mode runs. After synthesizing the combined file, the existing
verification path can process it identically to a batch response.

**Per-chunk timeout-retry:** If a chunk's `_invoke_claude` call returns None
(timeout), retry with only the unaddressed items from that chunk before aborting.
Reuse the existing timeout-retry logic from batch mode (lines 814-848).

**Mid-chunk failure handling:** On chunk failure, mark remaining unprocessed items
in the chunk as DEFERRED with note `"chunk failure (priority {p})"`. These appear
in the tracker with explicit failure context rather than silently staying OPEN.

**DraftHouse integration:** The JSONL path already works correctly for essential
review data (issue_fixed, issue_rejected, etc.). Add `chunk_start` and `chunk_end`
case handling to `WorkspaceParser.parseRoundFromJsonl` to capture chunk boundary
metadata. The synthesized combined `implementor-{rn}.md` (above) fixes the
markdown fallback path.

**Reviewer chunk awareness:** In multi-round reviews, the reviewer sees mixed
confirmation statuses (some ADDRESSED from chunk 1, some still OPEN from chunk 2).
Add a note to the reviewer prompt when chunked mode was used: "The implementor
addressed items in priority-ordered chunks (HIGH → MEDIUM → LOW). Items may have
been addressed in separate invocations without cross-chunk context."

**Test requirements:** Add tests for:
- Chunk failure mid-round (items properly marked DEFERRED)
- Resume after chunked round (synthesized file parsed correctly)
- Single-priority-tier reviews (one chunk, no checkpoint)
- Human skip at checkpoint (remaining items DEFERRED)
- Evidence verification on chunked output

### Phase 3 — Pilot and Decide

Run `--chunked` on the next 3-5 real reviews alongside batch baselines from
Phase 1a. Include at least one controlled comparison: the same spec reviewed
with both batch and chunked orchestration, to isolate the effect of chunking
from spec-specific variance.

Collect:

| Metric | Batch (Phase 1a) | Chunked (pilot) |
|--------|-------------------|------------------|
| Total implementor cost | From baselines | Per-pilot |
| Time to first update | Full batch latency | First chunk latency |
| Cross-issue connections | N/A | Cross-references in later chunks |
| Coverage | Issues addressed / total | Issues addressed / total |
| Early terminations | N/A | Times human skipped chunks |
| Rounds to convergence | From baselines | Per-pilot |

**Session context design decision:** Each chunk runs as a fresh `claude -p`
session with no cross-chunk context. This is the simplest design and directly
tests whether cross-issue context loss matters in practice. If Phase 3 data
shows quality degradation from context loss, session reuse across chunks becomes
a follow-up concern.

**Decision:** Chunk, keep batch, or hybrid — based on real-world data across
diverse specs and priority distributions.

## Output

A research report (markdown) containing:

1. Cost model tables (Phase 1a)
2. Cross-issue pattern assessment with go/no-go decision (Phase 1b)
3. If proceed: pilot data tables and comparison (Phase 3)
4. Recommendation: chunk, keep batch, or hybrid — with evidence

## Acceptance Criteria

- [ ] Cost baselines parsed and aggregated from existing progress.log files
      (Phase 1a — data exists, needs parsing)
- [ ] Cross-issue pattern analysis on at least one review with priority data
      (Phase 1b — new analysis)
- [ ] If Phase 1b green-lights: existing `--chunked` bugs fixed — resume
      synthesis, evidence verification, timeout-retry, failure handling
      (Phase 2 — hardening existing implementation)
- [ ] If Phase 2 complete: test coverage for chunked mode edge cases
      (Phase 2 — new tests)
- [ ] If Phase 2 complete: at least one controlled same-spec comparison plus
      pilot data from 2-4 additional reviews (Phase 3 — new data collection)
- [ ] Written recommendation with evidence

## Non-goals

- Statistical significance testing (pilot observations suffice for a
  go/no-go decision)
- DraftHouse UI changes for chunk visualization (the JSONL path works for
  data; rendering improvements are a separate concern)
