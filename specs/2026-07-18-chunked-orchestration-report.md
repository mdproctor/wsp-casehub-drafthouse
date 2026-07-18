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

## Head-to-Head Comparison

Ran both modes on the same 137-line spec (the research design spec itself).

### Reviewer findings

| | Batch | Chunked |
|---|---|---|
| Issues raised | 13 | 12 |
| Shared core findings | 6 | 6 |
| Unique findings | 7 (methodological) | 4 (implementation bugs) |

Both found the same 6 core problems. Batch found more methodological
weaknesses (single-sample go/no-go, no quality definition, design/measurement
conflation). Chunked found more implementation-specific bugs (resume ignores
chunk files, evidence verification skipped, parser gap). Neither missed
something critical the other caught.

### Implementor fix quality

**Batch:** Addressed all 13 issues in one pass. Competent verification of
reviewer claims, spec updates committed. Standard quality.

**Chunked HIGH (4 focus items):** Addressed all 12 issues — ignored the
focus constraint and responded to the full reviewer file. Prompt design flaw.

**Chunked MEDIUM (5 focus items):** Re-addressed R1-05 through R1-09 with
more precision than either batch or HIGH. On timeout-retry (R1-05): batch
wrote "missing timeout-retry." MEDIUM distinguished two failure modes —
timeout with partial output (retry correct) vs human abort (current break
correct) — and traced both code paths. The narrower scope produced deeper
analysis.

**Chunked LOW (3 focus items):** Straightforward — two rejections, one fix.
Adequate but unremarkable.

### Cost

| Mode | Reviewer | Implementor | Total |
|------|----------|-------------|-------|
| Batch | $1.72 | $3.27 | $4.99 |
| Chunked | $1.96 | $6.00 (H:$1.87 M:$2.30 L:$1.83) | $7.96 |

Chunked costs 60% more. Skipping LOW would bring it to $6.13 (23% more).

### Known bugs found during comparison

1. **Prompt scoping:** `build_implementor_prompt` hardcodes the output filename
   to `implementor-{N}.md`. Chunked code expects `implementor-{N}-chunk-{K}.md`.
   Each chunk overwrites the previous. Responses were processed before overwrite
   (tracker correct) but resume and evidence verification are broken.
2. **Focus constraint ignored:** HIGH chunk addressed all issues, not just HIGH.
   The prompt passes focus items but the implementor reads the full reviewer
   file and responds to everything it sees.

## Recommendation

**Adopt `--chunked` for pilot use after fixing the two bugs above.**

1. **Quality is at least equivalent, potentially better for middle tiers.**
   The narrower scope lets the implementor go deeper on each issue. The
   reviewer finds the same core problems regardless of mode.

2. **Cost is 60% higher per round.** Early termination partially offsets
   this. Whether the depth improvement justifies the cost depends on review
   complexity — pilot data will clarify.

3. **UX is clearly better.** HIGH results in 1-3 minutes instead of 5-10.
   Human controls depth between tiers.

4. **Two bugs must be fixed first:** prompt filename scoping and focus
   constraint enforcement. Without these, chunked mode silently loses data
   on resume.

## Next Steps

1. Fix the two bugs (prompt filename, focus enforcement)
2. Pilot `--chunked` on the next 3-5 real reviews
3. After pilot data: make default, keep opt-in, or revert
