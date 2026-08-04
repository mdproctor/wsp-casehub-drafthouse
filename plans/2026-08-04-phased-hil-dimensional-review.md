# Phased HIL for Dimensional Reviews — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #108 — Explore context-aware chunking if basic chunking shows quality loss
**Issue group:** #108

**Goal:** Replace the single-pass dimensional review orchestration in
SKILL.md with a three-phase pipeline that gives the human meaningful
engagement points after ~2-3 minutes instead of ~20 minutes.

**Architecture:** Phase 1 runs all dimensions at light degree
(reviewer-only). The calling session captures workspace paths, presents
findings, and the human decides which dimensions to continue. Phase 2
resumes surviving dimensions at the selected degree via existing
`--workspace` resume. Phase 3 optionally runs cross-cutting. All phase
transitions are driven by background task completion notifications.

**Tech Stack:** Python (review.py), SKILL.md (Claude skill orchestration),
JSONL events

## Global Constraints

- No new CLI flags in review.py — use existing `--degree`, `--workspace`,
  `--max-rounds`
- review.py's session management internals stay internal — the calling
  session only passes workspace paths and degree
- The reviewer-implementor loop within a round is unchanged
- Dimension-specific prompts and briefs are unchanged
- Changes are in soredium (`~/claude/hortora/soredium/design-review/`),
  not in drafthouse

---

### Task 1: Add dimension lifecycle JSONL events to review.py

The review.py orchestrator needs to emit structured events that
WorkspaceWatcher can consume for live progress display. These are
written to progress.log alongside existing log lines.

**Files:**
- Modify: `~/claude/hortora/soredium/design-review/review.py`
- Test: `~/claude/hortora/soredium/design-review/tests/test_review.py` (add to existing)

**Interfaces:**
- Consumes: existing `_log()` function, `_write_jsonl()` function
- Produces: `_emit_event(ws, event_type, payload)` function that writes
  JSON lines to `progress.log` prefixed with `EVENT:`. Format:
  `EVENT: {"type": "<event_type>", ...payload}`

- [ ] **Step 1: Write failing tests for event emission**

Add tests to the existing `tests/test_review.py` file. Follow the
existing import pattern (`sys.path.insert` + direct module import):

```python
# Add to tests/test_review.py — new test class at the end

class TestEmitEvent:

    def test_emit_event_writes_json_line(self, tmp_path):
        """_emit_event writes a parseable JSON line prefixed with EVENT: to progress.log."""
        import review
        review._LOG_FILE = tmp_path / "progress.log"
        review._LOG_FILE.touch()

        _emit_event(tmp_path, "dimension_start", {"dimension": "coherence", "degree": "light", "phase": 1})

        lines = review._LOG_FILE.read_text().splitlines()
        event_lines = [l for l in lines if "EVENT:" in l]
        assert len(event_lines) == 1
        json_str = event_lines[0].split("EVENT: ", 1)[1]
        payload = json.loads(json_str)
        assert payload["type"] == "dimension_start"
        assert payload["dimension"] == "coherence"
        assert payload["degree"] == "light"
        assert payload["phase"] == 1

    def test_emit_event_round_findings(self, tmp_path):
        """round_findings event includes issue counts by priority."""
        import review
        review._LOG_FILE = tmp_path / "progress.log"
        review._LOG_FILE.touch()

        _emit_event(tmp_path, "round_findings", {
            "dimension": "robustness",
            "round_number": 1,
            "issues": {"HIGH": 2, "MEDIUM": 3, "LOW": 1},
        })

        lines = review._LOG_FILE.read_text().splitlines()
        event_lines = [l for l in lines if "EVENT:" in l]
        json_str = event_lines[0].split("EVENT: ", 1)[1]
        payload = json.loads(json_str)
        assert payload["issues"]["HIGH"] == 2
        assert payload["issues"]["MEDIUM"] == 3
```

Also add `_emit_event` to the import block at the top of test_review.py:
```python
from review import (
    _apply_resume_degree,
    _build_reviewer_events,
    _build_implementor_events,
    _build_chunk_start_event,
    _build_chunk_end_event,
    _detect_last_round,
    _emit_event,
    _load_degree,
    _write_jsonl,
    parse_args,
)
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest ~/claude/hortora/soredium/design-review/tests/test_review.py::TestEmitEvent -v`
Expected: FAIL — `_emit_event` not defined

- [ ] **Step 3: Implement `_emit_event` in review.py**

Add after the existing `_write_jsonl` function (around line 90):

```python
def _emit_event(ws: Path, event_type: str, payload: dict) -> None:
    """Write a structured event to progress.log for external consumers."""
    event = {"type": event_type, **payload}
    _log(f"EVENT: {json.dumps(event)}")
```

- [ ] **Step 4: Wire event emission into the round loop**

Add calls at the appropriate points in `main()`:

After workspace setup (line ~485):
```python
_emit_event(ws, "dimension_start", {
    "dimension": args.review_type,
    "degree": resolved_depth,
    "phase": 1 if start_round <= 1 else 2,
})
```

After `_log(f"  {len(new_issues)} new issue(s) raised")` (line ~680):
```python
priority_counts = {}
for issue in new_issues:
    p = getattr(issue, 'priority', 'UNSPECIFIED')
    priority_counts[p] = priority_counts.get(p, 0) + 1
_emit_event(ws, "round_findings", {
    "dimension": args.review_type,
    "round_number": round_num,
    "issues": priority_counts,
})
```

After round complete log (line ~965):
```python
_emit_event(ws, "round_end", {
    "dimension": args.review_type,
    "round_number": round_num,
    "cost": cost_per_round,
})
```

Before `REVIEW DONE` (line ~1756):
```python
_emit_event(_REVIEW_WS, "dimension_done", {
    "dimension": args.review_type if args else "unknown",
    "total_rounds": round_num if 'round_num' in dir() else 0,
    "cost": cumulative_cost if 'cumulative_cost' in dir() else 0,
})
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `python3 -m pytest ~/claude/hortora/soredium/design-review/tests/test_review.py::TestEmitEvent -v`
Expected: PASS

- [ ] **Step 6: Run full test suite**

Run: `python3 -m pytest ~/claude/hortora/soredium/design-review/tests/ -v`
Expected: All existing tests still pass

- [ ] **Step 7: Commit**

```bash
git -C ~/claude/hortora/soredium add design-review/review.py design-review/tests/test_events.py
git commit -m "feat(design-review): add dimension lifecycle JSONL events to review.py

Refs casehubio/drafthouse#108"
```

---

### Task 2: Rewrite SKILL.md multi-dimension orchestration to three-phase pipeline

This is the core change. Replace the current "launch all dimensions at
selected degree → single watchdog → cross-cutting" flow with: phase 1
(all at light) → HIL → phase 2 (surviving at selected degree) → HIL →
phase 3 (cross-cutting) → final results.

**Files:**
- Modify: `~/claude/hortora/soredium/design-review/SKILL.md`

**Interfaces:**
- Consumes: review.py CLI (`--spec`, `--title`, `--type`, `--degree`,
  `--workspace`, `--source-dirs`, `--arch-files`, `--stage`)
- Produces: updated SKILL.md that the calling Claude session follows

- [ ] **Step 1: Replace Step 4 (Launch dimension reviews)**

Replace the current Step 4 content (lines ~142-189) with the phased
launch. The key changes:

**Phase 1 announcement:**
```markdown
## Step 4 — Launch Phase 1 (round 1 exploration)

**IMPORTANT: These are long-running processes. Do NOT run inline — use
`run_in_background: true` on all Bash tool calls.**

### Single-type mode (backward compat)

If the user provided `--type X`, launch a single review.py with that type
at the selected degree. Skip cross-cutting. Use the old behavior exactly.
The phased model only applies to multi-dimension mode.

### Multi-dimension mode (default for post-spec)

Tell the user BEFORE running:
> Starting post-spec review of **{title}** at **{degree}** depth.
>
> **Phase 1:** Launching 3 dimension reviewers (round 1 only, ~2-3 min).
> You'll see findings and decide which dimensions to pursue further.
>
> - Coherence (completeness, consistency, gaps)
> - Structure (decomposition, boundaries, dependencies)
> - Robustness (failure modes, edge cases, error paths)

Launch all three with `run_in_background: true`. **Always use
`--degree light`** for phase 1 regardless of the user's selected degree:

\`\`\`bash
python3 ~/.claude/skills/design-review/review.py \
  --spec {spec_path} --title {title}-coherence \
  --type coherence --degree light \
  --stage {maturity_stage} --source-dirs {dirs}
\`\`\`

\`\`\`bash
python3 ~/.claude/skills/design-review/review.py \
  --spec {spec_path} --title {title}-structure \
  --type structure --degree light \
  --stage {maturity_stage} --source-dirs {dirs}
\`\`\`

\`\`\`bash
python3 ~/.claude/skills/design-review/review.py \
  --spec {spec_path} --title {title}-robustness \
  --type robustness --degree light \
  --stage {maturity_stage} --source-dirs {dirs}
\`\`\`

**Capture workspace paths:** As each background task completes, read
the first line of its progress.log to extract the workspace path:
`Review (<type>): <workspace-path>`. Store these — they are needed
for phase 2.
```

- [ ] **Step 2: Replace Step 5 (watchdog) with Phase 1 HIL gate**

Replace the current Step 5 (unified watchdog + cross-cutting auto-launch)
with the phase 1 HIL interaction:

```markdown
## Step 5 — Phase 1 HIL gate

When all three background tasks complete (via task completion
notifications), read each dimension's tracker.md from the captured
workspace paths. Present the round 1 summary:

\`\`\`
Round 1 findings:

  Coherence:  {N} issues ({priority breakdown})
  Structure:  {N} issues ({priority breakdown})
  Robustness: {N} issues ({priority breakdown})
\`\`\`

Dimensions with zero issues are candidates for exclusion — they found
nothing to iterate on.

**If selected degree is light:** phase 1 IS the complete review. Skip
the four-action prompt. Proceed directly to Step 7 (pre-cross-cutting
gate).

**If selected degree is standard/adversarial/deep:** present the
four-action prompt:

Use `AskUserQuestion` with four options:
- **Accept all** — all dimensions with findings continue to phase 2
- **Refuse all** — stop everything, proceed to Step 7 (pre-cross-cutting)
- **Refuse subset** — present a multi-select of dimensions to stop
- **Discuss** — human asks about specific findings. Read the relevant
  tracker entries and discuss. After discussion, re-present the prompt.

Record which dimensions survived (accepted) and their workspace paths.
```

- [ ] **Step 3: Add Step 6 (Phase 2 — depth pursuit)**

Insert a new Step 6 for phase 2:

```markdown
## Step 6 — Phase 2: depth pursuit (surviving dimensions)

**Only runs if the selected degree is not light AND at least one
dimension survived the Phase 1 HIL gate.**

Launch surviving dimensions as background tasks using `--workspace`
to resume from phase 1:

\`\`\`bash
python3 ~/.claude/skills/design-review/review.py \
  --workspace {coherence-workspace-path} \
  --degree {selected_degree} \
  --source-dirs {dirs}
\`\`\`

Repeat for each surviving dimension. **Always pass `--degree`
explicitly** — without it, review.py falls back to the saved "light"
from phase 1.

Set up a fallback watchdog cron (5-minute interval) for stall/failure
detection only:

> Check progress of the depth pursuit for {title}. Read the last 10
> lines of each surviving dimension's progress.log.
> Report status for each dimension.
> If no update for 10+ min — warn about stall.
> If REVIEW FAILED/CRASHED — report error.
> This cron does NOT launch cross-cutting. Delete when all dimensions
> report REVIEW DONE.

When all surviving dimensions complete (via background task
notifications), proceed to Step 7.
```

- [ ] **Step 4: Add Step 7 (pre-cross-cutting gate)**

```markdown
## Step 7 — Pre-cross-cutting gate

Read each surviving dimension's tracker.md and present the full results:

\`\`\`
Dimension results:

  Coherence:  {N} rounds, {M} issues ({V} verified, {A} accepted, {D} deferred) — ${C}
  Robustness: {N} rounds, {M} issues ({V} verified, {A} accepted, {D} deferred) — ${C}
  Structure:  killed after round 1
\`\`\`

Two options:
- **Run cross-cutting** — launch phase 3 with surviving dimension
  trackers as `--arch-files`
- **Skip** — present final results from dimensions only (Step 9)

If cross-cutting selected, proceed to Step 8.
If skip, proceed to Step 9.
```

- [ ] **Step 5: Add Step 8 (Phase 3 — cross-cutting)**

```markdown
## Step 8 — Phase 3: cross-cutting (if approved)

Launch as a background task:

\`\`\`bash
python3 ~/.claude/skills/design-review/review.py \
  --spec {spec_path} --title {title}-crosscutting \
  --type crosscutting --degree {selected_degree} \
  --stage {maturity_stage} --source-dirs {dirs} \
  --arch-files {coherence-tracker} {structure-tracker} {robustness-tracker}
\`\`\`

Only pass tracker.md paths for surviving dimensions (not killed ones).

When the background task completes, proceed to Step 9.
```

- [ ] **Step 6: Renumber remaining steps and update Step 9 (results)**

Renumber existing Steps 6-8 to align. Update the results presentation
(old Step 8, now Step 9) to include phase information in the summary.

- [ ] **Step 7: Verify SKILL.md is internally consistent**

Read through the full updated SKILL.md. Check:
- Step numbers are sequential with no gaps
- All references to workspace paths use the captured paths from phase 1
- The watchdog cron in phase 2 does NOT auto-launch cross-cutting
- Single-type mode is unaffected by phased changes
- Light degree flow correctly skips phase 2

- [ ] **Step 8: Commit**

```bash
git -C ~/claude/hortora/soredium add design-review/SKILL.md
git commit -m "feat(design-review): three-phase HIL orchestration in SKILL.md

Phase 1 runs all dimensions at light (reviewer-only, ~2-3 min).
Human sees findings, kills unproductive dimensions.
Phase 2 resumes survivors at selected degree.
Phase 3 runs cross-cutting if approved.

Refs casehubio/drafthouse#108"
```

---

### Task 3: Manual validation — run a phased review

Validate the phased flow end-to-end by running it against the spec
itself.

**Files:**
- No code changes — this is a validation task

- [ ] **Step 1: Run a light phased review**

Use the phased HIL spec as the test target. Follow the updated SKILL.md
flow:
1. Launch phase 1 (3 dimensions at light)
2. Wait for all 3 to complete
3. Read tracker.md files from captured workspace paths
4. Present round 1 findings
5. Verify the HIL prompt appears correctly

- [ ] **Step 2: Verify workspace resume works**

Take one of the phase 1 workspace paths and run phase 2 manually:
```bash
python3 ~/.claude/skills/design-review/review.py \
  --workspace <phase1-workspace-path> \
  --degree standard \
  --source-dirs <dirs>
```

Verify:
- review.py detects round 1 from tracker.md
- Resumes at round 2 with standard degree's round count (3) and budget ($5.00)
- The reviewer has context from round 1's findings

- [ ] **Step 3: Verify JSONL events are emitted**

Check the progress.log for EVENT: lines:
```bash
grep "EVENT:" <workspace-path>/progress.log
```

Verify `dimension_start`, `round_findings`, `round_end`, and
`dimension_done` events are present and parseable as JSON.

- [ ] **Step 4: Commit any fixes**

If validation surfaced issues, fix them and commit.
