# Chunked Orchestration Research — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #97 — Research: chunked orchestration vs batch
**Issue group:** #97

**Goal:** Determine whether design-review's implementor should address
issues in priority-ordered chunks instead of all at once, and build
the mechanism if the evidence supports it.

**Architecture:** Three phases — (1a) cost baselines from existing
reviews as operational tooling, (1b) cross-issue pattern analysis as
go/no-go gate, (2) build `--chunked` flag in review.py behind a flag.
Phase 3 (pilot on real reviews) happens after this plan completes.

**Tech Stack:** Python 3.12, pytest, existing design-review skill in
`~/claude/hortora/soredium/design-review/`

## Global Constraints

- Code changes are in soredium (skills repo), not drafthouse
- `review.py`, `tracker.py`, `prompts.py` are the target files
- Tests run with: `python3 -m pytest ~/claude/hortora/soredium/design-review/tests/ -v`
- The `--chunked` flag must be backward-compatible — default behavior unchanged
- DraftHouse integration requires zero changes (WorkspaceParser already handles priority metadata)

---

### Task 1: Cost Reporting in adr-status.py

**Files:**
- Modify: `~/adr/adr-status.py`

**Interfaces:**
- Consumes: progress.log files in `~/adr/*/` workspaces
- Produces: `--costs` CLI flag that prints per-round cost breakdown table

Phase 1a of the research — build cost extraction as operational tooling
useful regardless of chunking decision.

- [ ] **Step 1: Add `--costs` flag to argparse**

Add argument parsing to adr-status.py (currently has no argparse). The
`--costs` flag triggers detailed per-round cost output instead of the
default status table.

```python
import argparse

parser = argparse.ArgumentParser(description="Design review status report")
parser.add_argument("--costs", action="store_true",
                    help="Show per-round cost breakdown for all reviews")
parser.add_argument("filter", nargs="?", default=None,
                    help="Filter reviews by project or title substring")
cli_args = parser.parse_args()
```

Move the existing script body into `def main():` so argparse runs first.

- [ ] **Step 2: Implement cost extraction function**

```python
def extract_round_costs(log_path: Path) -> list[dict]:
    """Parse progress.log for per-round reviewer/implementor costs.

    Returns list of dicts: [{round: 1, reviewer: 1.96, implementor: 2.93, cumulative: 4.90}, ...]
    """
    lines = log_path.read_text().splitlines()
    rounds = []
    current_round = 0
    reviewer_cost = 0.0
    implementor_cost = 0.0

    for line in lines:
        if "ROUND " in line and "====" not in line:
            try:
                current_round = int(line.split("ROUND")[1].strip().split()[0])
            except (ValueError, IndexError):
                pass

        m = re.search(r'Reviewer done \(\$(\d+\.\d+)\)', line)
        if m:
            reviewer_cost = float(m.group(1))

        m = re.search(r'Implementor done \(\$(\d+\.\d+)\)', line)
        if m:
            implementor_cost = float(m.group(1))

        m = re.search(r'Round \d+ complete .* \$(\d+\.\d+) cumulative', line)
        if m:
            rounds.append({
                "round": current_round,
                "reviewer": reviewer_cost,
                "implementor": implementor_cost,
                "cumulative": float(m.group(1)),
            })
            reviewer_cost = 0.0
            implementor_cost = 0.0

    return rounds
```

- [ ] **Step 3: Implement cost summary output**

When `--costs` is passed, iterate all reviews and print cost tables:

```python
def print_cost_report(reviews: list[Path]) -> None:
    for ws in reviews:
        log = ws / "progress.log"
        if not log.exists():
            continue
        rounds = extract_round_costs(log)
        if not rounds:
            continue

        display = f"{ws.parent.name}/{ws.name}"
        total = rounds[-1]["cumulative"] if rounds else 0.0
        avg = total / len(rounds) if rounds else 0.0

        print(f"\n{display}")
        print(f"  {'Round':>5}  {'Rev':>6}  {'Imp':>6}  {'Total':>7}")
        print(f"  {'─'*5}  {'─'*6}  {'─'*6}  {'─'*7}")
        for r in rounds:
            print(f"  {r['round']:5d}  ${r['reviewer']:5.2f}  ${r['implementor']:5.2f}  ${r['cumulative']:6.2f}")
        print(f"  {'─'*5}  {'─'*6}  {'─'*6}  {'─'*7}")
        print(f"  Total: ${total:.2f}  Avg/round: ${avg:.2f}  Rounds: {len(rounds)}")
```

- [ ] **Step 4: Test manually**

Run: `python3 ~/adr/adr-status.py --costs`

Verify: each review with a progress.log shows a per-round cost table.
Verify: running without `--costs` produces the original status output.

- [ ] **Step 5: Commit**

```bash
git -C ~/adr add adr-status.py
git -C ~/adr commit -m "feat: add --costs flag for per-round cost breakdown

Refs casehubio/drafthouse#97"
```

---

### Task 2: Cross-Issue Pattern Analysis (go/no-go gate)

**Files:**
- Read: `~/adr/casehub-drafthouse/brainstorming-ui-decomposition-20260714-041508/responses/implementor-*.md`
- Read: `~/adr/casehub-drafthouse/brainstorming-ui-decomposition-20260714-041508/tracker.md`
- Create: `~/claude/public/casehub/drafthouse/specs/2026-07-18-cross-issue-analysis.md`

**Interfaces:**
- Consumes: existing review workspace data
- Produces: go/no-go document that determines whether Task 3 proceeds

This is analytical work, not code. Read the brainstorming-ui review
(7 HIGH, 12 MEDIUM) and classify cross-issue patterns.

- [ ] **Step 1: Read the tracker to identify priority distribution**

Read `tracker.md`. List all issues by priority tier. Note which issues
the implementor addressed vs rejected.

- [ ] **Step 2: Read implementor round 1-2 responses**

For each implementor response, classify:
1. **Self-contained fixes:** Issue addressed without referencing other issues
2. **Same-priority references:** References another issue in the same tier
3. **Cross-priority references:** References an issue in a different tier
4. **Batched fixes:** Single fix explicitly resolves multiple issues

For each cross-priority reference, assess:
- Would the fix have been different if the implementor only saw its tier?
- Is the cross-reference essential (fix depends on it) or incidental (mentions it)?

- [ ] **Step 3: Write the go/no-go assessment**

Write to `specs/2026-07-18-cross-issue-analysis.md`:

```markdown
# Cross-Issue Pattern Analysis — Phase 1b

## Data Source
brainstorming-ui-decomposition review (24 issues: 7 HIGH, 12 MEDIUM)

## Findings

| Pattern | Count | Quality Impact |
|---------|-------|---------------|
| Self-contained fixes | N | None |
| Same-priority references | N | None (preserved in chunks) |
| Cross-priority references | N | [assess each] |
| Batched fixes | N | [assess each] |

## Cross-Priority References (detail)

[For each cross-priority reference, describe the fix and whether
chunking would have degraded it]

## Decision

**GO / NO-GO**: [decision with rationale]
```

- [ ] **Step 4: Commit the analysis**

```bash
git -C ~/claude/public/casehub/drafthouse add specs/2026-07-18-cross-issue-analysis.md
git -C ~/claude/public/casehub/drafthouse commit -m "docs: cross-issue pattern analysis (Phase 1b)

Refs #97"
```

- [ ] **Step 5: Gate check**

If NO-GO: stop here. Write the recommendation document explaining why
batch is better. Close #97 with the finding.

If GO: proceed to Task 3.

---

### Task 3: Priority Grouping in Tracker

**Files:**
- Modify: `~/claude/hortora/soredium/design-review/tracker.py`
- Modify: `~/claude/hortora/soredium/design-review/tests/test_tracker.py`

**Interfaces:**
- Consumes: `TrackedIssue.priority` field (already exists)
- Produces: `Tracker.get_focus_items_by_priority() -> dict[str, list[str]]`
  used by Task 4's chunk loop

- [ ] **Step 1: Write failing test for priority grouping**

Add to `tests/test_tracker.py`:

```python
class TestPriorityGrouping:

    def test_groups_focus_items_by_priority(self):
        t = Tracker(project_name="test")
        t.add_issue("R1-01", "High issue", round_raised=1, priority="HIGH")
        t.add_issue("R1-02", "Medium issue", round_raised=1, priority="MEDIUM")
        t.add_issue("R1-03", "Low issue", round_raised=1, priority="LOW")
        t.add_issue("R1-04", "Another high", round_raised=1, priority="HIGH")

        grouped = t.get_focus_items_by_priority()

        assert grouped == {
            "HIGH": ["R1-01", "R1-04"],
            "MEDIUM": ["R1-02"],
            "LOW": ["R1-03"],
        }

    def test_excludes_terminal_items(self):
        t = Tracker(project_name="test")
        t.add_issue("R1-01", "Verified", round_raised=1, priority="HIGH")
        t.add_issue("R1-02", "Open", round_raised=1, priority="HIGH")
        t.mark_addressed("R1-01", section_ref="4.1", commit_hash="abc", rationale="fixed")
        t.mark_verified("R1-01")

        grouped = t.get_focus_items_by_priority()

        assert grouped == {"HIGH": ["R1-02"]}

    def test_empty_tiers_omitted(self):
        t = Tracker(project_name="test")
        t.add_issue("R1-01", "High only", round_raised=1, priority="HIGH")

        grouped = t.get_focus_items_by_priority()

        assert grouped == {"HIGH": ["R1-01"]}
        assert "MEDIUM" not in grouped
        assert "LOW" not in grouped

    def test_preserves_insertion_order_within_tier(self):
        t = Tracker(project_name="test")
        t.add_issue("R1-03", "Third", round_raised=1, priority="HIGH")
        t.add_issue("R1-01", "First", round_raised=1, priority="HIGH")

        grouped = t.get_focus_items_by_priority()

        assert grouped["HIGH"] == ["R1-03", "R1-01"]
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest ~/claude/hortora/soredium/design-review/tests/test_tracker.py::TestPriorityGrouping -v`
Expected: FAIL with `AttributeError: 'Tracker' object has no attribute 'get_focus_items_by_priority'`

- [ ] **Step 3: Implement `get_focus_items_by_priority`**

Add to `Tracker` class in `tracker.py`, after `get_focus_items`:

```python
PRIORITY_ORDER: Final = ("HIGH", "MEDIUM", "LOW")

# ... inside Tracker class:

def get_focus_items_by_priority(self) -> dict[str, list[str]]:
    focus = self.get_focus_items()
    grouped: dict[str, list[str]] = {}
    for iid in focus:
        priority = self._issues[iid].priority
        grouped.setdefault(priority, []).append(iid)
    return {p: grouped[p] for p in PRIORITY_ORDER if p in grouped}
```

Note: `PRIORITY_ORDER` goes at module level, after the existing `_TERMINAL` constant.

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest ~/claude/hortora/soredium/design-review/tests/test_tracker.py::TestPriorityGrouping -v`
Expected: all 4 tests PASS

- [ ] **Step 5: Run full test suite**

Run: `python3 -m pytest ~/claude/hortora/soredium/design-review/tests/ -v`
Expected: all existing tests still pass

- [ ] **Step 6: Commit**

```bash
git -C ~/claude/hortora/soredium add design-review/tracker.py design-review/tests/test_tracker.py
git -C ~/claude/hortora/soredium commit -m "feat(design-review): add get_focus_items_by_priority to Tracker

Groups non-terminal focus items by priority tier (HIGH → MEDIUM → LOW).
Empty tiers are omitted. Preserves insertion order within each tier.

Refs casehubio/drafthouse#97"
```

---

### Task 4: Chunked Implementor Loop in review.py

**Files:**
- Modify: `~/claude/hortora/soredium/design-review/review.py`
- Modify: `~/claude/hortora/soredium/design-review/tests/test_review.py`

**Interfaces:**
- Consumes: `Tracker.get_focus_items_by_priority()` from Task 3
- Consumes: `PRIORITY_ORDER` from Task 3
- Produces: `--chunked` CLI flag, chunked implementor execution loop,
  JSONL chunk-boundary events

This task replaces the single implementor invocation (review.py lines
609-723) with a priority-chunked loop when `--chunked` is passed.

- [ ] **Step 1: Write failing tests for chunk JSONL events**

Add to `tests/test_review.py`:

```python
class TestChunkEvents:

    def test_chunk_start_event(self):
        from review import _build_chunk_start_event
        event = _build_chunk_start_event(round_num=1, priority="HIGH",
                                          chunk_index=0, total_chunks=3,
                                          item_count=4)
        assert event["event"] == "chunk_start"
        assert event["round"] == 1
        assert event["priority"] == "HIGH"
        assert event["chunkIndex"] == 0
        assert event["totalChunks"] == 3
        assert event["itemCount"] == 4

    def test_chunk_end_event(self):
        from review import _build_chunk_end_event
        event = _build_chunk_end_event(round_num=1, priority="HIGH",
                                        chunk_index=0, addressed=3, skipped=1)
        assert event["event"] == "chunk_end"
        assert event["round"] == 1
        assert event["priority"] == "HIGH"
        assert event["addressed"] == 3
        assert event["skipped"] == 1
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest ~/claude/hortora/soredium/design-review/tests/test_review.py::TestChunkEvents -v`
Expected: FAIL with `ImportError`

- [ ] **Step 3: Implement chunk event builders**

Add to `review.py`, after `_build_implementor_events`:

```python
def _build_chunk_start_event(round_num: int, priority: str,
                              chunk_index: int, total_chunks: int,
                              item_count: int) -> dict:
    return {
        "event": "chunk_start",
        "round": round_num,
        "priority": priority,
        "chunkIndex": chunk_index,
        "totalChunks": total_chunks,
        "itemCount": item_count,
    }


def _build_chunk_end_event(round_num: int, priority: str,
                            chunk_index: int,
                            addressed: int, skipped: int) -> dict:
    return {
        "event": "chunk_end",
        "round": round_num,
        "priority": priority,
        "chunkIndex": chunk_index,
        "addressed": addressed,
        "skipped": skipped,
    }
```

- [ ] **Step 4: Run chunk event tests**

Run: `python3 -m pytest ~/claude/hortora/soredium/design-review/tests/test_review.py::TestChunkEvents -v`
Expected: PASS

- [ ] **Step 5: Add `--chunked` flag to argparse**

In `parse_args()`, add:

```python
parser.add_argument("--chunked", action="store_true",
                    help="Run implementor in priority-ordered chunks (HIGH → MEDIUM → LOW)")
```

- [ ] **Step 6: Implement `_run_implementor_chunked`**

Add a new function that handles the chunked implementor loop. This
function replaces the inline implementor block (lines 609-723) when
`--chunked` is active.

```python
def _run_implementor_chunked(
    ws: Path,
    tracker: Tracker,
    round_num: int,
    source_dirs: list[str],
    spec_path: str,
    model: str,
    budget: float,
    effort: str,
    mode: str,
    depth: str | None,
    is_interactive: bool,
) -> tuple[float, list[dict]]:
    """Run implementor in priority-ordered chunks.

    Returns (cumulative_chunk_cost, all_implementor_events).
    """
    from tracker import PRIORITY_ORDER

    grouped = tracker.get_focus_items_by_priority()
    priorities = [p for p in PRIORITY_ORDER if p in grouped]
    total_chunks = len(priorities)
    chunk_cost = 0.0
    all_events: list[dict] = []

    for chunk_idx, priority in enumerate(priorities):
        chunk_items = grouped[priority]
        _log(f"  Chunk {chunk_idx + 1}/{total_chunks}: {priority} ({len(chunk_items)} items)")

        # Emit chunk_start event
        all_events.append(_build_chunk_start_event(
            round_num, priority, chunk_idx, total_chunks, len(chunk_items)))

        # Build prompt scoped to this priority tier
        chunk_prompt = build_implementor_prompt(
            round_num=round_num, focus_items=chunk_items,
            source_dirs=source_dirs, workspace_root=str(ws),
            spec_path=spec_path, mode=mode, depth=depth,
        )

        # Use chunk-specific response file
        chunk_file_name = f"responses/implementor-{round_num}-chunk-{chunk_idx}.md"
        chunk_file = ws / chunk_file_name

        result = _invoke_claude(
            ws, "implementor", chunk_prompt, source_dirs,
            model, budget, effort,
            expected_file=chunk_file_name,
            focus_count=len(chunk_items),
        )
        if result is None:
            _log(f"  Chunk {priority} failed — aborting chunked run")
            break

        chunk_cost += result.get("cost", 0.0)
        _log(f"  Chunk {priority} done (${result.get('cost', 0.0):.2f})")

        # Process chunk responses
        addressed = 0
        if chunk_file.exists():
            content = chunk_file.read_text()
            responses = extract_issue_responses(content)
            for resp in responses:
                if not tracker.has_issue(resp.issue_id):
                    continue
                try:
                    if resp.status == "FIXED":
                        tracker.mark_addressed(
                            resp.issue_id, section_ref=resp.section_ref or "",
                            commit_hash="", rationale=resp.body[:200])
                        addressed += 1
                    elif resp.status == "REJECTED":
                        tracker.mark_rejected(resp.issue_id, rationale=resp.rationale[:200])
                        addressed += 1
                    elif resp.status == "ESCALATED":
                        tracker.mark_deferred(resp.issue_id, note="DECISION_NEEDED")
                        addressed += 1
                except ValueError:
                    pass

            impl_signal = extract_signal(content)
            impl_responses = extract_issue_responses(content)
            chunk_events = _build_implementor_events(
                round_num, impl_responses,
                (impl_signal.signal_type, impl_signal.description),
                extract_assumptions(content),
                extract_settled_decisions(content),
            )
            all_events.extend(chunk_events)

        skipped = len(chunk_items) - addressed
        all_events.append(_build_chunk_end_event(
            round_num, priority, chunk_idx, addressed, skipped))

        # Commit chunk response
        _git_commit(ws, [chunk_file_name], f"round {round_num}: implementor chunk {priority}")
        if mode in ("final-review", "code-review"):
            for sd in source_dirs:
                subprocess.run(["git", "add", "-A"], cwd=sd, capture_output=True)
                subprocess.run(["git", "commit", "-m",
                    f"review: code fixes — {mode} round {round_num} chunk {priority}",
                    "--allow-empty"], cwd=sd, capture_output=True)
        elif spec_path:
            spec_dir = Path(spec_path).parent
            spec_name = Path(spec_path).name
            subprocess.run(["git", "add", spec_name], cwd=spec_dir, capture_output=True)
            subprocess.run(["git", "commit", "-m",
                f"docs: spec revised — round {round_num} chunk {priority}",
                "--allow-empty"], cwd=spec_dir, capture_output=True)

        # HIL checkpoint between chunks (not after the last one)
        if chunk_idx < total_chunks - 1:
            remaining_priorities = priorities[chunk_idx + 1:]
            remaining_count = sum(len(grouped[p]) for p in remaining_priorities)
            _log(f"  Next: {', '.join(remaining_priorities)} ({remaining_count} items)")

            action = _prompt_hil(
                f"[c]ontinue to {remaining_priorities[0]} / [s]kip remaining / [a]bort? ",
                default="c",
            )
            if action == "s":
                _log(f"  Skipping remaining chunks — marking {remaining_count} items DEFERRED")
                for p in remaining_priorities:
                    for iid in grouped[p]:
                        try:
                            tracker.mark_deferred(iid, note=f"skipped at chunk checkpoint (priority {p})")
                        except ValueError:
                            issue = tracker.get_issue(iid)
                            issue.status = IssueStatus.DEFERRED
                            issue.notes = f"skipped at chunk checkpoint (priority {p})"
                break
            elif action == "a":
                _log("  Chunked run aborted by user")
                break

    return chunk_cost, all_events
```

- [ ] **Step 7: Wire `--chunked` into the round loop**

In `main()`, after the reviewer step (around line 609), replace the
inline implementor block with a conditional:

```python
        # --- Step 3: Implementor ---
        if args.chunked:
            chunk_cost, impl_events = _run_implementor_chunked(
                ws, tracker, round_num, args.source_dirs, spec_path,
                model, budget, effort, args.mode, resolved_depth,
                _is_interactive(),
            )
            cumulative_cost += chunk_cost
            _write_jsonl(ws, "implementor", round_num, impl_events)
            # Commit tracker after all chunks
            tracker.current_round = round_num
            tracker.record_round(round_num)
            tracker.write(ws / "tracker.md")
            _git_commit(ws, ["tracker.md"], f"tracker: round {round_num} chunked verification")
            _check_unstaged(ws)

            # Skip the rest of the inline implementor block
            # (termination checks follow below)
        else:
            # ... existing implementor code unchanged ...
```

The existing implementor block (lines 609-790) stays as-is for the
non-chunked path. The `if args.chunked:` branch calls the new function
and then falls through to the termination checks.

- [ ] **Step 8: Run full test suite**

Run: `python3 -m pytest ~/claude/hortora/soredium/design-review/tests/ -v`
Expected: all tests pass (no existing behavior changed)

- [ ] **Step 9: Commit**

```bash
git -C ~/claude/hortora/soredium add design-review/review.py design-review/tests/test_review.py
git -C ~/claude/hortora/soredium commit -m "feat(design-review): add --chunked mode for priority-ordered implementor

Runs the implementor in HIGH → MEDIUM → LOW chunks instead of one
batch. Between chunks: HIL checkpoint for skip/abort. Each chunk gets
its own response file and JSONL events. Default behavior unchanged.

Refs casehubio/drafthouse#97"
```

---

### Task 5: Write Research Report

**Files:**
- Create: `~/claude/public/casehub/drafthouse/specs/2026-07-18-chunked-orchestration-report.md`

**Interfaces:**
- Consumes: cost data from Task 1, pattern analysis from Task 2, implementation from Tasks 3-4
- Produces: recommendation document

- [ ] **Step 1: Run cost report**

Run: `python3 ~/adr/adr-status.py --costs`

Capture the output. Extract summary statistics:
- Average reviewer cost per round across all reviews
- Average implementor cost per round across all reviews
- Cost taper pattern (early vs late rounds)
- Total review cost vs issue count correlation

- [ ] **Step 2: Write the report**

Structure:

```markdown
# Chunked Orchestration Research — Report

## Phase 1a: Cost Baselines

[Cost tables from adr-status.py --costs]
[Summary statistics]

## Phase 1b: Cross-Issue Pattern Analysis

[Findings from Task 2]
[Go/no-go decision and rationale]

## Phase 2: --chunked Implementation

[What was built, how it works]
[Architecture: priority grouping → chunk loop → HIL checkpoint]

## Recommendation

[chunk / keep batch / hybrid — with evidence]

## Next Steps

Phase 3: pilot --chunked on next 3-5 real reviews to collect:
- Per-chunk cost data
- Early termination frequency
- Cross-issue quality assessment across diverse specs
```

- [ ] **Step 3: Commit the report**

```bash
git -C ~/claude/public/casehub/drafthouse add specs/2026-07-18-chunked-orchestration-report.md
git -C ~/claude/public/casehub/drafthouse commit -m "docs: chunked orchestration research report (#97)

Refs #97"
```
