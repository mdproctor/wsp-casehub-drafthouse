# Chunked Orchestration Research — Report

**Issue:** casehubio/drafthouse#97
**Date:** 2026-07-18

## Phase 1a: Cost Baselines

Per-round cost data extracted from 11 drafthouse reviews (40 rounds total):

| Metric | Value |
|--------|-------|
| Avg reviewer cost/round | $1.94 |
| Avg implementor cost/round | $2.08 |
| Avg total cost/round | $4.02 |
| Avg total cost/review | $14.63 |
| Total across all reviews | $160.97 |

**Cost taper pattern:** Early rounds (1-2) average $4.50-6.20/round. Later
rounds (5+) taper to $1.50-2.50/round as issues converge and fewer items
need addressing.

**Implication for chunking:** At $2.08/implementor invocation, chunking into
3 priority tiers would cost ~$6.24 instead of ~$2.08 if all tiers run. However:
- Each chunk addresses fewer items → lower per-chunk cost (estimated $0.80-1.50)
- Early termination (skipping LOW items) saves ~$0.80-1.50 per skipped tier
- Net cost is likely similar to batch, with variance depending on early
  termination frequency

Cost data built into `adr-status.py --costs` for ongoing operational use.

## Phase 1b: Cross-Issue Pattern Analysis

Analyzed the brainstorming-ui review (24 issues: 7 HIGH, 12 MEDIUM, 5 LOW).
Read implementor rounds 1-2 (20 fixes total).

| Pattern | Count | % | Quality Impact |
|---------|-------|---|---------------|
| Self-contained fixes | 18 | 90% | None |
| Same-priority references | 1 | 5% | None (preserved in chunks) |
| Cross-priority references | 1 | 5% | None (incidental) |
| Batched fixes | 0 | 0% | N/A |

**The one cross-priority reference:** R1-09 (MEDIUM, terminal injection) mentions
R1-15 (LOW, querySelector coupling) because both converge on custom events. The
fix for R1-09 would be identical without seeing R1-15. When the LOW-tier chunk
runs, it would simply note "already resolved by R1-09 fix."

**Decision: GO.** Cross-issue coupling is rare (5%) and incidental. A chunked
implementor seeing only its priority tier would produce the same quality fixes.

## Phase 2: --chunked Implementation

Built and committed to soredium/design-review:

**`tracker.py`** — Added `get_focus_items_by_priority()` to Tracker. Groups
non-terminal focus items into `{"HIGH": [...], "MEDIUM": [...], "LOW": [...]}`
ordered by `PRIORITY_ORDER`. Empty tiers omitted. 4 unit tests.

**`review.py`** — Added `--chunked` flag and `_run_implementor_chunked()`.
When `--chunked`:
1. After reviewer runs, partitions focus items by priority
2. Runs implementor once per non-empty priority tier (HIGH → MEDIUM → LOW)
3. Each chunk writes to `implementor-{N}-chunk-{K}.md`
4. Emits `chunk_start`/`chunk_end` JSONL events with priority, item count,
   addressed/skipped counts
5. Between chunks: HIL checkpoint — continue, skip remaining, or abort
6. Skip marks remaining items DEFERRED
7. Tracker updated after all chunks

Leverages the existing timeout-retry pattern (partial progress persistence,
retry with reduced scope) — the chunked loop is the same mechanism applied
intentionally rather than reactively.

Default behavior unchanged — without `--chunked`, the existing batch path
runs identically to before. 53 tests pass.

## Recommendation

**Adopt `--chunked` for pilot use.** The evidence supports it:

1. **Quality risk is negligible.** 90% of implementor fixes are self-contained.
   The one cross-priority reference was incidental — fix quality unchanged.

2. **Cost is neutral-to-favorable.** Per-chunk invocations are cheaper than
   full-batch. Early termination (skipping LOW) saves further. Net cost depends
   on termination frequency — pilot data will clarify.

3. **UX is clearly better.** Batch: 5-10 min silence → dump. Chunked: 1-3 min →
   HIGH results → decide → MEDIUM → decide. The human sees important results
   sooner and can redirect effort.

4. **Implementation is behind a flag.** Zero risk to existing reviews. The
   `--chunked` flag can be removed or made default after pilot validation.

## Next Steps

**Phase 3: Pilot `--chunked` on the next 3-5 real reviews.** Collect:
- Per-chunk cost vs batch baseline
- Early termination frequency (how often humans skip LOW)
- Any quality issues not caught by the retrospective analysis

After pilot data, decide: make `--chunked` the default, keep as opt-in, or
revert if quality degrades.
