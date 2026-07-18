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

**Prior art:** The timeout-retry mechanism in `_invoke_claude()` / the batch
implementor loop already implements partial-batch retry — when the implementor
times out, the PM identifies addressed items, pre-applies them, and retries with
only the missing items. This demonstrates the orchestration plumbing works for
multiple invocations with reduced scope, but it is failure-driven (random scope
reduction after timeout) rather than design-driven (intentional priority ordering
with human control).

**Existing implementation:** A `--chunked` flag was added to review.py in commit
`4e7793a` (2026-07-18). The current implementation includes
`_run_implementor_chunked()`, priority grouping via
`Tracker.get_focus_items_by_priority()`, JSONL chunk events, and HIL checkpoints
between priority tiers. However, the implementation lacks parity with the batch
path's safeguards (see Phase 2). This spec's Phase 2 is therefore a validation and
hardening effort, not a greenfield build.

## Research Questions

1. **Cross-issue coupling (go/no-go gate):** Does the implementor reference issues
   across priority tiers? Would a chunked implementor seeing only its priority
   slice produce different (worse) fixes?
2. **Cost model:** What does each implementor invocation cost as a function of
   issue count and input/output token breakdown? Can we estimate chunked cost
   from existing batch data, accounting for context-loading multiplication?
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
| Input tokens per invocation | `claude -p` JSON output `input_tokens` |
| Output tokens per invocation | `claude -p` JSON output `output_tokens` |
| Wall-clock time per invocation | progress.log timestamps |
| Cost taper over rounds | cumulative cost / round number |
| Total review cost | final cumulative |
| Issue count vs cost | tracker.md issue count vs total cost |

The input/output token breakdown is critical for predicting chunked cost. Each
chunk invocation re-loads the full context (spec, codebase, protocols), so input
tokens are multiplied by chunk count while output tokens scale with issue count
per chunk. Without this breakdown, aggregate cost numbers cannot be decomposed
into the components that chunking changes.

Note: Anthropic's automatic prompt caching may reduce the effective
context-loading cost for consecutive chunk invocations that share a common prompt
prefix. The token counts should capture both cached and uncached reads to
understand the actual cost multiplier.

### Phase 1b — Cross-Issue Pattern Analysis (go/no-go gate)

**Prerequisite:** Issue #96 (structured priority metadata) is complete. Priority
metadata infrastructure is fully implemented: `parser.py` extracts
PRIORITY/LOCATION/DEPENDS, `tracker.py` stores priority on `TrackedIssue`, the
reviewer CLAUDE.md prompts for structured metadata.

Analyse all available reviews that have priority data. As of 2026-07-18, 60+
reviews include structured PRIORITY metadata (all reviews since ~2026-07-13).
Start with a representative sample across projects:

- **casehub-drafthouse/brainstorming-ui-decomposition** (7 HIGH, 12 MEDIUM)
- **casehub-engine/unified-execution-model** (35 issues with priority data)
- **casehub-blocks/hybrid-decomposition** (12 issues with priority data)
- Additional reviews selected to cover varying issue counts and priority
  distributions

For each review, read the implementor's round 1-2 responses and classify:

- **Cross-priority references:** Did the implementor reference MEDIUM/LOW issues
  when fixing HIGH issues?
- **Batched fixes:** Were any fixes applied to multiple issues simultaneously?
- **Pattern dependency:** Would a chunked implementor seeing only one priority
  tier have produced a different (worse) fix?

**Decision gate:** If cross-issue coupling is frequent and quality-affecting
across multiple reviews, chunking has a real quality risk — stop here and document
why batch is better. If coupling is rare or cosmetic, proceed to Phase 2. A
single-review analysis is insufficient for a go/no-go decision — the gate
requires evidence from at least 3 reviews with varying priority distributions.

### Phase 2 — Validate and Harden `--chunked` Mode (if Phase 1b green-lights)

The `--chunked` flag already exists in review.py (commit `4e7793a`). The current
implementation covers the core orchestration: priority grouping, per-chunk
invocation, JSONL events, HIL checkpoints, and early termination. Phase 2
validates this implementation and closes safeguard gaps before using it for pilot
data.

**Design choice — no cross-chunk context:** Each chunk invocation sees only its
priority slice. This is intentional: the simplest design directly tests whether
cross-issue context loss matters in practice. If the pilot reveals quality loss
from missing cross-chunk context, context-aware chunking becomes a follow-up
design (see issue filed below).

**Design choice — cold-start chunks:** Each chunk is a fresh `claude -p`
invocation with no session continuity from prior chunks. Within a single round,
chunks do not share session state. This isolates each priority tier's results and
avoids session-window complexity. Anthropic's automatic prompt caching may reduce
the effective context-loading cost for consecutive invocations sharing a common
prompt prefix — Phase 1a token data will quantify this.

**Safeguard parity checklist** — the existing chunked path is missing these
batch-path features:

- [ ] **Timeout-retry per chunk:** If a chunk times out, identify addressed
      items within the chunk, preserve partial progress, and retry with only the
      unaddressed items from that priority tier (batch equivalent: lines 814–858)
- [ ] **Session ID management:** Reuse sessions within a chunk retry to benefit
      from cached context (batch equivalent: lines 788–810)
- [ ] **Missing-file retry:** If a chunk invocation produces no output file,
      retry with HIL prompt before aborting (batch equivalent: lines 851–858)
- [ ] **Evidence verification:** For spec-review mode, verify FIXED responses
      against git diff (batch equivalent: lines 894–943)

**Test coverage prerequisite:** Before trusting pilot data, the chunked path
needs integration tests covering:

- [ ] Orchestration flow (3-tier priority grouping → sequential invocation)
- [ ] HIL checkpoint behavior (continue / skip / abort)
- [ ] Partial failure and recovery (chunk timeout midway through tiers)
- [ ] Tracker state transitions across chunks
- [ ] JSONL event ordering with chunk events interleaved
- [ ] Non-interactive mode defaults (all chunks auto-continue)

### Phase 3 — Pilot and Decide

**3a. Controlled comparison (same spec, both modes):** Run one review with both
batch and `--chunked` on the same spec to isolate the chunking variable from
spec-to-spec variance. This requires running the review twice — once batch, once
chunked — on a spec with a representative priority distribution (at least 3
HIGH, 5+ MEDIUM items).

**3b. Diverse pilots:** Run `--chunked` on the next 3-5 real reviews alongside
batch baselines from Phase 1a. The controlled comparison (3a) isolates the
variable; the diverse pilots test generalisability across different specs,
projects, and priority distributions.

Collect for both 3a and 3b:

| Metric | Batch (Phase 1a / 3a) | Chunked (pilot) |
|--------|------------------------|------------------|
| Total implementor cost | From baselines | Per-pilot |
| Input tokens per invocation | From Phase 1a | Per-chunk |
| Time to first update | Full batch latency | First chunk latency |
| Coverage | Issues addressed / total | Issues addressed / total |
| Early terminations | N/A | Times human skipped chunks |

**Quality metrics** (derivable from tracker data, no additional instrumentation):

| Quality metric | What it measures | Source |
|----------------|-----------------|--------|
| Reviewer contest rate | How often FIXED items are contested next round | tracker round-over-round status |
| Rounds to convergence | Total rounds before all items resolve | tracker terminal state |
| REJECTED rate | Implementor pushback frequency | tracker REJECTED count / total |
| Cross-chunk contradictions | Fixes in chunk N that contradict fixes in chunk N-1 | reviewer verification comments |
| Duplicate effort | Same spec section modified in multiple chunks | git diff per chunk commit |

**Measuring context loss:** The absence of explicit cross-references between
chunks does not prove context wasn't needed. Phase 3 should specifically check:
- Whether later-chunk fixes contradict or duplicate earlier-chunk fixes
- Whether the reviewer contests chunked FIXED items at a higher rate than batch
- Whether rounds to convergence increase with chunking

**Multi-round convergence:** If chunking causes the implementor to miss
cross-issue patterns, the reviewer will contest more items, adding rounds. More
rounds × more chunks per round could make total cost worse than batch. Phase 3
must track total rounds and total cost across the full review, not just per-round
metrics.

**Decision:** Chunk, keep batch, or hybrid — based on real-world data across
both the controlled comparison and diverse pilots.

**Open questions deferred to future work:**
- Whether the reviewer should be told about chunk boundaries (currently each
  chunk produces a separate response file; the reviewer sees all of them)
- Context-aware chunking: if basic no-context chunking shows quality loss,
  design a variant that passes prior chunk summaries to later chunks

## Output

A research report (markdown) containing:

1. Cost model tables (Phase 1a)
2. Cross-issue pattern assessment with go/no-go decision (Phase 1b)
3. If proceed: pilot data tables and comparison (Phase 3)
4. Recommendation: chunk, keep batch, or hybrid — with evidence

## Acceptance Criteria

- [ ] Cross-issue pattern analysis on at least 3 reviews with priority data
      (Phase 1b)
- [ ] Cost baselines with input/output token breakdown from existing reviews
      (Phase 1a)
- [ ] If Phase 1b green-lights: `--chunked` implementation validated with
      safeguard parity and test coverage (Phase 2)
- [ ] At least one controlled comparison: same spec reviewed with batch vs
      chunked orchestration (Phase 3a)
- [ ] If Phase 2 validated: pilot data from at least 3 diverse reviews (Phase 3b)
- [ ] Quality metrics collected alongside cost metrics (Phase 3)
- [ ] Written recommendation with evidence

## Non-goals

- Statistical significance testing
- Context-aware chunking (cross-chunk summaries) — deferred unless basic
  chunking shows quality loss
- DraftHouse UI changes
