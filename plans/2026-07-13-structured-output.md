# Structured Design-Review Output — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #96 — Improve design-review structured output
**Issue group:** #96

**Goal:** Add structured metadata lines (LOCATION, PRIORITY, DEPENDS,
EVIDENCE) to design-review agent output, emit JSONL sidecar files as
typed domain events, replace broken diff verification with mechanical
evidence checking, and update DraftHouse's Java parser to consume JSONL.

**Architecture:** Two coordinated changes across soredium (Python) and
drafthouse (Java). Python parser extracts new metadata, review.py writes
JSONL sidecar per response, tracker.py gains evidence verification. Java
WorkspaceParser reads JSONL with per-round markdown fallback.
WorkspaceReplayAdapter uses enriched fields (location, priority) and
verdict-based confirmation dispatch.

**Tech Stack:** Python 3.11+ (soredium), Java 21 / Quarkus 3.34
(DraftHouse), Jackson (JSON parsing), JUnit 5 + @QuarkusTest

## Global Constraints

- Priority values: `HIGH`, `MEDIUM`, `LOW` — matching `blocks.conversation.Priority`
- JSONL schema version: `1` (header line: `{"event": "schema_version", "version": 1}`)
- Confirmation verdict: `"resolved" | "accepted" | "contested"` — replaces boolean pair
- Evidence format: `<location> | commit:<hash>` with optional `| lines:<start>-<end>`
- All Python files: edit in soredium (`~/claude/hortora/soredium/design-review/`), then `sync-local`
- All Java files: edit in drafthouse (`~/claude/casehub/drafthouse/`)
- Old workspaces without structured markers must parse unchanged (defaults: location=null, priority="LOW", depends=[], evidence=[])

---

### Task 1: Python parser enrichment (`parser.py`)

**Files:**
- Modify: `~/claude/hortora/soredium/design-review/parser.py`
- Create: `~/claude/hortora/soredium/design-review/tests/test_parser.py`

**Interfaces:**
- Produces: `Evidence(location, commit, lines)` dataclass, enriched `Issue(issue_id, title, body, location, priority, depends)`, enriched `IssueResponse(..., evidence)`, `Confirmation(issue_id, verdict, reason)`

- [ ] **Step 1: Create test directory and test file**

```bash
mkdir -p ~/claude/hortora/soredium/design-review/tests
touch ~/claude/hortora/soredium/design-review/tests/__init__.py
```

- [ ] **Step 2: Write failing tests for LOCATION/PRIORITY/DEPENDS extraction**

Create `~/claude/hortora/soredium/design-review/tests/test_parser.py`:

```python
"""Tests for structured metadata extraction in parser.py."""

import sys
from pathlib import Path

# Bootstrap: add skill dir to path so imports work
sys.path.insert(0, str(Path(__file__).parent.parent))

from parser import (
    Evidence,
    extract_new_issues,
    extract_issue_responses,
    extract_confirmations,
)


class TestIssueMetadataExtraction:
    """LOCATION, PRIORITY, DEPENDS extraction from reviewer issues."""

    def test_all_metadata_present(self):
        content = (
            "### Missing failure mode\n"
            "LOCATION: §4.1 Payment Flow\n"
            "PRIORITY: HIGH\n"
            "DEPENDS: R1-02, R1-03\n"
            "\n"
            "The spec doesn't handle timeouts.\n"
        )
        issues = extract_new_issues(content, 1, set())
        assert len(issues) == 1
        issue = issues[0]
        assert issue.location == "§4.1 Payment Flow"
        assert issue.priority == "HIGH"
        assert issue.depends == ["R1-02", "R1-03"]
        assert "LOCATION:" not in issue.body
        assert "PRIORITY:" not in issue.body
        assert "DEPENDS:" not in issue.body
        assert "timeouts" in issue.body

    def test_no_metadata(self):
        content = "### Some issue\n\nJust prose here.\n"
        issues = extract_new_issues(content, 1, set())
        assert len(issues) == 1
        assert issues[0].location is None
        assert issues[0].priority == "LOW"
        assert issues[0].depends == []

    def test_partial_metadata(self):
        content = (
            "### Partial issue\n"
            "LOCATION: §2.3\n"
            "\n"
            "Only location provided.\n"
        )
        issues = extract_new_issues(content, 1, set())
        assert len(issues) == 1
        assert issues[0].location == "§2.3"
        assert issues[0].priority == "LOW"
        assert issues[0].depends == []

    def test_priority_case_insensitive(self):
        content = "### Issue\nPRIORITY: medium\n\nBody.\n"
        issues = extract_new_issues(content, 1, set())
        assert issues[0].priority == "MEDIUM"

    def test_depends_single(self):
        content = "### Issue\nDEPENDS: R1-01\n\nBody.\n"
        issues = extract_new_issues(content, 1, set())
        assert issues[0].depends == ["R1-01"]

    def test_multiple_issues_each_with_metadata(self):
        content = (
            "### First issue\n"
            "LOCATION: §1.1\n"
            "PRIORITY: HIGH\n"
            "\n"
            "First body.\n"
            "\n"
            "### Second issue\n"
            "LOCATION: §2.2\n"
            "PRIORITY: LOW\n"
            "\n"
            "Second body.\n"
        )
        issues = extract_new_issues(content, 1, set())
        assert len(issues) == 2
        assert issues[0].location == "§1.1"
        assert issues[0].priority == "HIGH"
        assert issues[1].location == "§2.2"
        assert issues[1].priority == "LOW"


class TestEvidenceExtraction:
    """EVIDENCE extraction from implementor FIXED responses."""

    def test_single_evidence(self):
        content = (
            "### R1-01: FIXED\n"
            "EVIDENCE: §4.1 | commit:abc123\n"
            "\n"
            "Updated the section.\n"
        )
        responses = extract_issue_responses(content)
        assert len(responses) == 1
        assert len(responses[0].evidence) == 1
        e = responses[0].evidence[0]
        assert e.location == "§4.1"
        assert e.commit == "abc123"
        assert e.lines is None

    def test_multiple_evidence(self):
        content = (
            "### R1-01: FIXED\n"
            "EVIDENCE: §4.1 | commit:abc123\n"
            "EVIDENCE: §4.2 | commit:abc123\n"
            "\n"
            "Updated both sections.\n"
        )
        responses = extract_issue_responses(content)
        assert len(responses[0].evidence) == 2

    def test_evidence_with_lines(self):
        content = (
            "### R1-01: FIXED\n"
            "EVIDENCE: src/Main.java | commit:def456 | lines:45-78\n"
            "\n"
            "Fixed the code.\n"
        )
        responses = extract_issue_responses(content)
        e = responses[0].evidence[0]
        assert e.location == "src/Main.java"
        assert e.commit == "def456"
        assert e.lines == "45-78"

    def test_no_evidence_on_fixed(self):
        content = "### R1-01: FIXED\n\nFixed it.\n"
        responses = extract_issue_responses(content)
        assert responses[0].evidence == []

    def test_evidence_not_extracted_on_rejected(self):
        content = (
            "### R1-01: REJECTED\n"
            "EVIDENCE: §4.1 | commit:abc123\n"
            "\n"
            "Not relevant.\n"
        )
        responses = extract_issue_responses(content)
        assert responses[0].evidence == []

    def test_evidence_stripped_from_body(self):
        content = (
            "### R1-01: FIXED\n"
            "EVIDENCE: §4.1 | commit:abc123\n"
            "\n"
            "Updated the section.\n"
        )
        responses = extract_issue_responses(content)
        assert "EVIDENCE:" not in responses[0].body


class TestConfirmationVerdict:
    """Confirmation verdict discriminator replaces boolean pair."""

    def test_resolved(self):
        content = "- R1-01: resolved\n"
        confs = extract_confirmations(content)
        assert len(confs) == 1
        assert confs[0].verdict == "resolved"

    def test_accepted(self):
        content = "- R1-02: accepted\n"
        confs = extract_confirmations(content)
        assert confs[0].verdict == "accepted"

    def test_still_open(self):
        content = "- R1-03: still open — needs more work\n"
        confs = extract_confirmations(content)
        assert confs[0].verdict == "contested"
        assert "needs more work" in confs[0].reason
```

- [ ] **Step 3: Run tests to verify they fail**

```bash
cd ~/claude/hortora/soredium/design-review && python3 -m pytest tests/test_parser.py -v
```

Expected: failures — `Evidence` not defined, `Issue` missing `location`/`priority`/`depends`, `Confirmation` missing `verdict`.

- [ ] **Step 4: Implement parser changes**

Use `Edit` on `~/claude/hortora/soredium/design-review/parser.py` (Python files outside IntelliJ project scope):

1. Add `Evidence` dataclass after `Confirmation`:
```python
@dataclass(frozen=True)
class Evidence:
    location: str
    commit: str
    lines: str | None = None
```

2. Add fields to `Issue`:
```python
@dataclass
class Issue:
    issue_id: str
    title: str
    body: str
    location: str | None = None
    priority: str = "LOW"
    depends: list[str] = field(default_factory=list)
```

3. Replace `Confirmation`:
```python
@dataclass(frozen=True)
class Confirmation:
    issue_id: str
    verdict: str  # "resolved" | "accepted" | "contested"
    reason: str = ""
```

4. Add field to `IssueResponse`:
```python
@dataclass
class IssueResponse:
    issue_id: str
    status: str
    section_ref: str | None = None
    rationale: str = ""
    body: str = ""
    evidence: list[Evidence] = field(default_factory=list)
```

5. Add new regex patterns after `_SETTLED_RE`:
```python
_LOCATION_RE: Final = re.compile(r"^LOCATION:\s*(.+)$", re.MULTILINE)
_PRIORITY_RE: Final = re.compile(r"^PRIORITY:\s*(HIGH|MEDIUM|LOW)\b", re.IGNORECASE | re.MULTILINE)
_DEPENDS_RE: Final = re.compile(r"^DEPENDS:\s*(.+)$", re.MULTILINE)
_EVIDENCE_RE: Final = re.compile(
    r"^EVIDENCE:\s*(.+?)\s*\|\s*commit:(\S+)(?:\s*\|\s*lines:(\S+))?\s*$",
    re.MULTILINE,
)
_METADATA_STRIP_RE: Final = re.compile(
    r"^(?:LOCATION|PRIORITY|DEPENDS|EVIDENCE):\s*.+$\n?", re.MULTILINE,
)
```

6. Update `extract_new_issues()` — after extracting body, parse metadata lines from the body and strip them:
```python
# Inside the loop, after body is computed and signal stripped:
location_match = _LOCATION_RE.search(body)
location = location_match.group(1).strip() if location_match else None

priority_match = _PRIORITY_RE.search(body)
priority = priority_match.group(1).upper() if priority_match else "LOW"

depends_match = _DEPENDS_RE.search(body)
depends = [d.strip() for d in depends_match.group(1).split(",") if d.strip()] if depends_match else []

body = _METADATA_STRIP_RE.sub("", body).strip()

issue_id = f"R{round_num}-{seq:02d}"
issues.append(Issue(issue_id=issue_id, title=title, body=body,
                    location=location, priority=priority, depends=depends))
```

7. Update `extract_issue_responses()` — after extracting body, parse EVIDENCE lines for FIXED status only:
```python
# Inside the loop, after body is computed:
evidence: list[Evidence] = []
if status == "FIXED":
    for ev_match in _EVIDENCE_RE.finditer(body):
        evidence.append(Evidence(
            location=ev_match.group(1).strip(),
            commit=ev_match.group(2),
            lines=ev_match.group(3),
        ))
    body = _METADATA_STRIP_RE.sub("", body).strip()

responses.append(IssueResponse(
    issue_id=issue_id, status=status, section_ref=section_ref,
    rationale=rationale, body=body, evidence=evidence,
))
```

8. Update `extract_confirmations()` — replace boolean logic with verdict:
```python
def extract_confirmations(content: str) -> list[Confirmation]:
    confirmations: list[Confirmation] = []
    for match in _CONFIRMATION_RE.finditer(content):
        issue_id = f"R{match.group(1)}-{match.group(2)}"
        status_text = match.group(3).lower().strip()
        if "resolved" in status_text and "still" not in status_text:
            verdict = "resolved"
        elif "accepted" in status_text:
            verdict = "accepted"
        else:
            verdict = "contested"

        reason = ""
        if verdict == "contested":
            line_end = content.find("\n", match.end())
            if line_end == -1:
                line_end = len(content)
            after = content[match.end():line_end].strip()
            after = re.sub(r"^[\s—\-:]+", "", after).strip()
            reason = after

        confirmations.append(Confirmation(
            issue_id=issue_id, verdict=verdict, reason=reason,
        ))
    return confirmations
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
cd ~/claude/hortora/soredium/design-review && python3 -m pytest tests/test_parser.py -v
```

Expected: all PASS.

- [ ] **Step 6: Commit**

```bash
git -C ~/claude/hortora/soredium add design-review/parser.py design-review/tests/
git -C ~/claude/hortora/soredium commit -m "feat(design-review): enrich parser with LOCATION/PRIORITY/DEPENDS/EVIDENCE extraction and verdict confirmation

Refs casehubio/drafthouse#96"
```

---

### Task 2: Python tracker enrichment (`tracker.py`)

**Files:**
- Modify: `~/claude/hortora/soredium/design-review/tracker.py`
- Create: `~/claude/hortora/soredium/design-review/tests/test_tracker.py`

**Interfaces:**
- Consumes: `Evidence` from parser.py (Task 1)
- Produces: `TrackedIssue` with `location`, `priority`, `depends` fields; `verify_evidence_against_diff(evidence, diff, spec_content) → EvidenceResult`; `verify_against_diff()` deleted

- [ ] **Step 1: Write failing tests for tracker enrichment and evidence verification**

Create `~/claude/hortora/soredium/design-review/tests/test_tracker.py`:

```python
"""Tests for tracker enrichment and evidence verification."""

import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent.parent))

from parser import Evidence
from tracker import (
    EvidenceResult,
    IssueStatus,
    Tracker,
    verify_evidence_against_diff,
)


class TestTrackerEnrichment:

    def test_add_issue_with_metadata(self):
        t = Tracker(project_name="test")
        t.add_issue("R1-01", "Test issue", round_raised=1,
                     location="§4.1", priority="HIGH", depends=["R1-02"])
        issue = t.get_issue("R1-01")
        assert issue.location == "§4.1"
        assert issue.priority == "HIGH"
        assert issue.depends == ["R1-02"]

    def test_add_issue_without_metadata_uses_defaults(self):
        t = Tracker(project_name="test")
        t.add_issue("R1-01", "Test issue", round_raised=1)
        issue = t.get_issue("R1-01")
        assert issue.location == ""
        assert issue.priority == "LOW"
        assert issue.depends == []

    def test_render_includes_metadata(self):
        t = Tracker(project_name="test")
        t.add_issue("R1-01", "Test issue", round_raised=1,
                     location="§4.1", priority="HIGH", depends=["R1-02"])
        rendered = t.render()
        assert "**Location:** §4.1" in rendered
        assert "**Priority:** HIGH" in rendered
        assert "**Depends:** R1-02" in rendered

    def test_render_omits_empty_metadata(self):
        t = Tracker(project_name="test")
        t.add_issue("R1-01", "Test issue", round_raised=1)
        rendered = t.render()
        assert "**Location:**" not in rendered
        assert "**Priority:**" not in rendered
        assert "**Depends:**" not in rendered


class TestVerifyEvidenceAgainstDiff:

    def test_empty_evidence_returns_not_verified(self):
        result = verify_evidence_against_diff([], "", "")
        assert not result.verified
        assert "no evidence provided" in result.note

    def test_section_found_in_diff(self):
        spec = "# Spec\n\n## §4.1 Payment Flow\n\nContent here.\n\n## §4.2 Next\n"
        diff = (
            "--- a/spec.md\n"
            "+++ b/spec.md\n"
            "@@ -3,3 +3,4 @@\n"
            " ## §4.1 Payment Flow\n"
            " \n"
            "-Content here.\n"
            "+Updated content here.\n"
            "+Added line.\n"
        )
        evidence = [Evidence(location="§4.1", commit="abc123")]
        result = verify_evidence_against_diff(evidence, diff, spec)
        assert result.verified

    def test_section_not_in_diff(self):
        spec = "# Spec\n\n## §4.1 Payment\n\nContent.\n\n## §4.2 Next\n\nOther.\n"
        diff = (
            "--- a/spec.md\n"
            "+++ b/spec.md\n"
            "@@ -7,2 +7,3 @@\n"
            " ## §4.2 Next\n"
            " \n"
            "-Other.\n"
            "+Changed other.\n"
        )
        evidence = [Evidence(location="§4.1", commit="abc123")]
        result = verify_evidence_against_diff(evidence, diff, spec)
        assert not result.verified
        assert "§4.1" in result.note

    def test_section_not_found_in_spec(self):
        spec = "# Spec\n\n## §1.1 Intro\n\nContent.\n"
        diff = "@@ -1,1 +1,1 @@\n-old\n+new\n"
        evidence = [Evidence(location="§9.9", commit="abc123")]
        result = verify_evidence_against_diff(evidence, diff, spec)
        assert not result.verified
        assert "not found in spec" in result.note
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd ~/claude/hortora/soredium/design-review && python3 -m pytest tests/test_tracker.py -v
```

Expected: failures — `EvidenceResult` not defined, `add_issue()` doesn't accept keyword args, `verify_evidence_against_diff` not defined.

- [ ] **Step 3: Implement tracker changes**

Edit `~/claude/hortora/soredium/design-review/tracker.py`:

1. Add `EvidenceResult` after existing dataclasses:
```python
@dataclass(frozen=True)
class EvidenceResult:
    verified: bool
    note: str = ""
```

2. Add fields to `TrackedIssue`:
```python
@dataclass
class TrackedIssue:
    issue_id: str
    summary: str
    round_raised: int
    status: IssueStatus = IssueStatus.OPEN
    contested_rounds: int = 0
    commit_before: str = ""
    commit_after: str = ""
    section_ref: str = ""
    rationale: str = ""
    notes: str = ""
    location: str = ""
    priority: str = "LOW"
    depends: list[str] = field(default_factory=list)
```

3. Update `Tracker.add_issue()` to accept optional keyword args:
```python
def add_issue(self, issue_id: str, summary: str, round_raised: int,
              location: str = "", priority: str = "LOW",
              depends: list[str] | None = None) -> None:
    self._issues[issue_id] = TrackedIssue(
        issue_id=issue_id, summary=summary, round_raised=round_raised,
        location=location, priority=priority,
        depends=depends if depends is not None else [],
    )
```

4. Update `Tracker.render()` — in the issue rendering loop, add metadata lines (only when non-empty/non-default):
```python
# After existing lines in the loop:
if issue.location:
    lines.append(f"- **Location:** {issue.location}")
if issue.priority and issue.priority != "LOW":
    lines.append(f"- **Priority:** {issue.priority}")
if issue.depends:
    lines.append(f"- **Depends:** {', '.join(issue.depends)}")
```

5. Add `verify_evidence_against_diff()` function and delete `verify_against_diff()`:

```python
def verify_evidence_against_diff(
    evidence: list,
    diff: str,
    spec_content: str,
) -> EvidenceResult:
    if not evidence:
        return EvidenceResult(verified=False, note="no evidence provided")

    for ev in evidence:
        section_ref = _extract_section_number(ev.location)
        if section_ref is None:
            continue

        section_range = _find_section_range(spec_content, section_ref)
        if section_range is None:
            return EvidenceResult(
                verified=False,
                note=f"section {ev.location} not found in spec",
            )

        modified_lines = _parse_diff_modified_lines(diff)
        start, end = section_range
        if any(start <= line <= end for line in modified_lines):
            return EvidenceResult(verified=True)

    first_loc = evidence[0].location if evidence else "unknown"
    return EvidenceResult(
        verified=False,
        note=f"{first_loc} not modified in diff",
    )


def _extract_section_number(location: str) -> str | None:
    m = re.search(r"§(\d+(?:\.\d+)*)", location)
    return m.group(1) if m else None


def _find_section_range(content: str, section_ref: str) -> tuple[int, int] | None:
    lines = content.split("\n")
    heading_re = re.compile(r"^(#{1,6})\s+(.+)")
    start_line = None
    start_level = 0

    for i, line in enumerate(lines, 1):
        m = heading_re.match(line)
        if not m:
            continue
        level = len(m.group(1))
        title = m.group(2)
        num_match = re.match(r"[§S]?(\d+(?:\.\d+)*)", title)
        if num_match and num_match.group(1) == section_ref:
            start_line = i
            start_level = level
            continue
        if start_line is not None and level <= start_level:
            return (start_line, i - 1)

    if start_line is not None:
        return (start_line, len(lines))

    # Fallback: search for §ref anywhere
    for i, line in enumerate(lines, 1):
        if f"§{section_ref}" in line:
            return (i, i)

    return None


def _parse_diff_modified_lines(diff: str) -> set[int]:
    modified = set()
    current_line = 0
    for line in diff.split("\n"):
        if line.startswith("@@"):
            m = re.search(r"\+(\d+)", line)
            if m:
                current_line = int(m.group(1))
            continue
        if line.startswith("+") and not line.startswith("+++"):
            modified.add(current_line)
            current_line += 1
        elif line.startswith("-") and not line.startswith("---"):
            pass  # deletion — don't advance line counter
        else:
            current_line += 1
    return modified
```

6. Delete the old `verify_against_diff()` function.

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd ~/claude/hortora/soredium/design-review && python3 -m pytest tests/test_tracker.py -v
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git -C ~/claude/hortora/soredium add design-review/tracker.py design-review/tests/test_tracker.py
git -C ~/claude/hortora/soredium commit -m "feat(design-review): enrich TrackedIssue, add verify_evidence_against_diff, delete verify_against_diff

Refs casehubio/drafthouse#96"
```

---

### Task 3: Python JSONL generation and review.py updates

**Files:**
- Modify: `~/claude/hortora/soredium/design-review/review.py`
- Create: `~/claude/hortora/soredium/design-review/tests/test_review.py`

**Interfaces:**
- Consumes: `Evidence`, enriched `Issue`, enriched `IssueResponse`, `Confirmation.verdict` from parser.py (Task 1); `verify_evidence_against_diff()` from tracker.py (Task 2)
- Produces: JSONL sidecar files (`reviewer-N.jsonl`, `implementor-N.jsonl`)

- [ ] **Step 1: Write failing tests for JSONL event building**

Create `~/claude/hortora/soredium/design-review/tests/test_review.py`:

```python
"""Tests for JSONL event building and evidence verification wrapper."""

import json
import sys
import tempfile
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent.parent))

from parser import Confirmation, Evidence, Issue, IssueResponse
from review import _build_reviewer_events, _build_implementor_events, _write_jsonl


class TestBuildReviewerEvents:

    def test_issues_become_issue_raised_events(self):
        issues = [
            Issue("R1-01", "Missing handler", "Body text",
                  location="§4.1", priority="HIGH", depends=["R1-02"]),
        ]
        signal = ("CONTINUE", None)
        assumptions = ["All input is UTF-8"]
        events = _build_reviewer_events(1, issues, signal, assumptions, [],
                                         "reviewer-1.md")
        raised = [e for e in events if e["event"] == "issue_raised"]
        assert len(raised) == 1
        assert raised[0]["id"] == "R1-01"
        assert raised[0]["location"] == "§4.1"
        assert raised[0]["priority"] == "HIGH"
        assert raised[0]["depends"] == ["R1-02"]
        assert raised[0]["round"] == 1

    def test_signal_event(self):
        events = _build_reviewer_events(1, [], ("CONTINUE", None), [], [],
                                         "reviewer-1.md")
        signals = [e for e in events if e["event"] == "round_signal"]
        assert len(signals) == 1
        assert signals[0]["signal"] == "CONTINUE"
        assert signals[0]["role"] == "reviewer"

    def test_assumption_events(self):
        events = _build_reviewer_events(1, [], ("CONTINUE", None),
                                         ["Assumption one"], [],
                                         "reviewer-1.md")
        assumptions = [e for e in events if e["event"] == "assumption"]
        assert len(assumptions) == 1
        assert assumptions[0]["text"] == "Assumption one"

    def test_confirmation_events(self):
        confs = [
            Confirmation("R1-01", "resolved", ""),
            Confirmation("R1-02", "accepted", ""),
            Confirmation("R1-03", "contested", "needs work"),
        ]
        events = _build_reviewer_events(2, [], ("CONTINUE", None), [], confs,
                                         "reviewer-2.md")
        conf_events = [e for e in events if e["event"] == "confirmation"]
        assert len(conf_events) == 3
        assert conf_events[0]["verdict"] == "resolved"
        assert conf_events[1]["verdict"] == "accepted"
        assert conf_events[2]["verdict"] == "contested"


class TestBuildImplementorEvents:

    def test_fixed_with_evidence(self):
        responses = [
            IssueResponse("R1-01", "FIXED", "4.1", "Updated section",
                          "Full body", [Evidence("§4.1", "abc123")]),
        ]
        events = _build_implementor_events(1, responses, ("CONTINUE", None),
                                            [], [])
        fixed = [e for e in events if e["event"] == "issue_fixed"]
        assert len(fixed) == 1
        assert fixed[0]["evidence"][0]["location"] == "§4.1"
        assert fixed[0]["evidence"][0]["commit"] == "abc123"

    def test_rejected_and_escalated(self):
        responses = [
            IssueResponse("R1-01", "REJECTED", rationale="Not a bug"),
            IssueResponse("R1-02", "ESCALATED", rationale="Needs decision"),
        ]
        events = _build_implementor_events(1, responses, ("CONTINUE", None),
                                            [], [])
        rejected = [e for e in events if e["event"] == "issue_rejected"]
        escalated = [e for e in events if e["event"] == "issue_escalated"]
        assert len(rejected) == 1
        assert len(escalated) == 1


class TestWriteJsonl:

    def test_writes_schema_version_and_events(self):
        with tempfile.TemporaryDirectory() as tmpdir:
            ws = Path(tmpdir)
            (ws / "responses").mkdir()
            events = [{"event": "issue_raised", "round": 1, "id": "R1-01"}]
            _write_jsonl(ws, "reviewer", 1, events)

            jsonl_path = ws / "responses" / "reviewer-1.jsonl"
            assert jsonl_path.exists()
            lines = jsonl_path.read_text().strip().split("\n")
            assert len(lines) == 2  # schema_version + 1 event
            header = json.loads(lines[0])
            assert header["event"] == "schema_version"
            assert header["version"] == 1
            event = json.loads(lines[1])
            assert event["event"] == "issue_raised"
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd ~/claude/hortora/soredium/design-review && python3 -m pytest tests/test_review.py -v
```

Expected: `ImportError` — `_build_reviewer_events`, `_build_implementor_events`, `_write_jsonl` not yet defined.

- [ ] **Step 3: Implement review.py changes**

Edit `~/claude/hortora/soredium/design-review/review.py`:

1. Add `_write_jsonl()` function:
```python
def _write_jsonl(ws: Path, role: str, round_num: int, events: list[dict]) -> None:
    jsonl_path = ws / "responses" / f"{role}-{round_num}.jsonl"
    tmp_path = jsonl_path.with_suffix(".jsonl.tmp")
    with open(tmp_path, "w") as f:
        f.write(json.dumps({"event": "schema_version", "version": 1}) + "\n")
        for event in events:
            f.write(json.dumps(event) + "\n")
    os.rename(tmp_path, jsonl_path)
```

2. Add `_build_reviewer_events()` function:
```python
def _build_reviewer_events(
    round_num: int,
    issues: list,
    signal: tuple[str, str | None],
    assumptions: list[str],
    confirmations: list,
    source_file: str,
) -> list[dict]:
    events: list[dict] = []
    for conf in confirmations:
        events.append({
            "event": "confirmation",
            "round": round_num,
            "id": conf.issue_id,
            "verdict": conf.verdict,
            "reason": conf.reason,
        })
    for issue in issues:
        events.append({
            "event": "issue_raised",
            "round": round_num,
            "id": issue.issue_id,
            "title": issue.title,
            "body": issue.body,
            "location": issue.location,
            "priority": issue.priority,
            "depends": issue.depends,
            "scope": None,
        })
    for text in assumptions:
        events.append({
            "event": "assumption",
            "round": round_num,
            "text": text,
            "source": source_file,
        })
    events.append({
        "event": "round_signal",
        "round": round_num,
        "role": "reviewer",
        "signal": signal[0],
        "description": signal[1],
    })
    return events
```

3. Add `_build_implementor_events()` function:
```python
def _build_implementor_events(
    round_num: int,
    responses: list,
    signal: tuple[str, str | None],
    assumptions: list[str],
    settled: list,
) -> list[dict]:
    events: list[dict] = []
    for resp in responses:
        event_type = {
            "FIXED": "issue_fixed",
            "REJECTED": "issue_rejected",
            "ESCALATED": "issue_escalated",
        }.get(resp.status, "issue_fixed")
        ev: dict = {
            "event": event_type,
            "round": round_num,
            "id": resp.issue_id,
            "rationale": resp.rationale or resp.body,
        }
        if resp.status == "FIXED":
            ev["sectionRef"] = resp.section_ref
            ev["evidence"] = [
                {"location": e.location, "commit": e.commit, "lines": e.lines}
                for e in resp.evidence
            ]
        events.append(ev)
    for sd in settled:
        events.append({
            "event": "settled_decision",
            "round": round_num,
            "text": sd.text,
            "fromIssue": sd.from_issue,
        })
    for text in assumptions:
        events.append({
            "event": "assumption",
            "round": round_num,
            "text": text,
            "source": f"implementor-{round_num}.md",
        })
    events.append({
        "event": "round_signal",
        "round": round_num,
        "role": "implementor",
        "signal": signal[0],
        "description": signal[1],
    })
    return events
```

4. In the `main()` round loop — after reviewer extraction block (after `_git_commit` for tracker + reviewer), add JSONL write:
```python
# Build and write reviewer JSONL
reviewer_events = _build_reviewer_events(
    round_num, new_issues,
    (signal.signal_type, signal.description),
    extract_assumptions(reviewer_content),
    confirmations,
    f"reviewer-{round_num}.md",
)
_write_jsonl(ws, "reviewer", round_num, reviewer_events)
```

And update the `_git_commit` call to include the JSONL file:
```python
_git_commit(ws, ["tracker.md", f"responses/reviewer-{round_num}.md",
                  f"responses/reviewer-{round_num}.jsonl"],
             f"tracker: round {round_num} reviewer issues")
```

5. After implementor extraction block, add JSONL write:
```python
impl_signal = extract_signal(impl_content)
impl_events = _build_implementor_events(
    round_num, responses,
    (impl_signal.signal_type, impl_signal.description),
    extract_assumptions(impl_content),
    extract_settled_decisions(impl_content),
)
_write_jsonl(ws, "implementor", round_num, impl_events)
```

And update the implementor `_git_commit` to include JSONL:
```python
_git_commit(ws, [f"responses/implementor-{round_num}.md",
                  f"responses/implementor-{round_num}.jsonl"],
             f"round {round_num}: implementor response")
```

6. Update confirmation branches in `main()` and `_rebuild_tracker()` to use `verdict`:
```python
# Replace:
#   if conf.is_resolved or conf.is_accepted:
# With:
if conf.verdict in ("resolved", "accepted"):
    if conf.verdict == "resolved":
        try:
            tracker.mark_verified(conf.issue_id)
        except ValueError:
            pass
    else:
        try:
            tracker.mark_accepted(conf.issue_id)
        except ValueError:
            pass
else:
    tracker.mark_contested(conf.issue_id, reason=conf.reason)
```

7. Replace `verify_against_diff` call site with `_verify_evidence()`:
```python
def _verify_evidence(evidence: list, spec_path: str | None) -> 'EvidenceResult':
    from adversarial_design_review.tracker import EvidenceResult, verify_evidence_against_diff
    if not evidence:
        return EvidenceResult(verified=False, note="no evidence provided")
    if not spec_path:
        return EvidenceResult(verified=False, note="no spec path")
    e = evidence[0]
    diff = _get_commit_diff(e.commit, spec_path)
    if diff is None:
        return EvidenceResult(verified=False, note=f"commit {e.commit} not found")
    try:
        spec_content = Path(spec_path).read_text()
    except OSError:
        return EvidenceResult(verified=False, note=f"spec file not readable")
    return verify_evidence_against_diff(evidence, diff, spec_content)


def _get_commit_diff(commit: str, spec_path_str: str) -> str | None:
    sp = Path(spec_path_str)
    result = subprocess.run(
        ["git", "diff", f"{commit}~1", commit, "--", sp.name],
        cwd=sp.parent, capture_output=True, text=True,
    )
    if result.returncode != 0:
        # Try git show for initial commit
        result2 = subprocess.run(
            ["git", "show", f"{commit}:{sp.name}"],
            cwd=sp.parent, capture_output=True, text=True,
        )
        if result2.returncode == 0:
            return f"@@ -0,0 +1,{len(result2.stdout.splitlines())} @@\n" + \
                   "\n".join(f"+{line}" for line in result2.stdout.splitlines())
        return None
    return result.stdout
```

Then replace the call site in the implementor processing loop where
`verify_against_diff(diff, resp.section_ref)` is called — use
`_verify_evidence(resp.evidence, spec_path)` instead.

8. Pass enriched issue metadata to `tracker.add_issue()`:
```python
for issue in new_issues:
    tracker.add_issue(issue.issue_id, issue.title, round_raised=round_num,
                      location=issue.location or "",
                      priority=issue.priority,
                      depends=issue.depends)
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd ~/claude/hortora/soredium/design-review && python3 -m pytest tests/ -v
```

Expected: all PASS (parser, tracker, and review tests).

- [ ] **Step 5: Commit**

```bash
git -C ~/claude/hortora/soredium add design-review/review.py design-review/tests/test_review.py
git -C ~/claude/hortora/soredium commit -m "feat(design-review): JSONL sidecar generation, evidence verification, verdict-based confirmations

Refs casehubio/drafthouse#96"
```

---

### Task 4: Python agent prompt update (`setup.py`)

**Files:**
- Modify: `~/claude/hortora/soredium/design-review/setup.py`

**Interfaces:**
- Produces: Updated `context.md` template with LOCATION/PRIORITY/DEPENDS/EVIDENCE format documentation

- [ ] **Step 1: Update `_default_context_md()` in setup.py**

Add two new subsections to the "Structured Output Format" section, after the
existing "Signal" subsection. Edit `~/claude/hortora/soredium/design-review/setup.py`:

Insert before the closing `"""` of `_default_context_md()`:

```
### Issue metadata (reviewer: after each issue heading)

    ### Missing failure mode for payment timeout
    LOCATION: §4.1 Payment Flow
    PRIORITY: HIGH
    DEPENDS: R1-02

    The spec doesn't handle the case where...

LOCATION, PRIORITY, and DEPENDS must appear on separate lines immediately
after the issue heading, before the body text. All three are optional:
- LOCATION: the spec section this issue relates to (§N.N format)
- PRIORITY: HIGH, MEDIUM, or LOW (default: LOW if omitted)
- DEPENDS: comma-separated issue IDs (e.g. R1-01, R1-03)

### Evidence markers (implementor: FIXED responses only)

    ### R1-01: FIXED
    EVIDENCE: §4.1 | commit:abc123
    EVIDENCE: §4.2 | commit:abc123

    Updated §4.1 with terminal PAYMENT_FAILED state.

EVIDENCE lines must appear immediately after the FIXED heading, before
the rationale text. Each line is: location | commit:<hash>
Optionally add | lines:<start>-<end> for code changes.
At least one EVIDENCE line is required for every FIXED response.
```

- [ ] **Step 2: Commit**

```bash
git -C ~/claude/hortora/soredium add design-review/setup.py
git -C ~/claude/hortora/soredium commit -m "feat(design-review): add structured metadata format to context.md template

Refs casehubio/drafthouse#96"
```

- [ ] **Step 3: Sync local**

```bash
sync-local
```

---

### Task 5: Java JSONL reader and parser enrichment (`WorkspaceParser.java`)

**Files:**
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/debate/WorkspaceParser.java` (use IntelliJ MCP)
- Modify: `server/runtime/src/test/java/io/casehub/drafthouse/debate/WorkspaceParserTest.java` (use IntelliJ MCP)
- Create: `server/runtime/src/test/resources/fixtures/workspace-replay-jsonl/` (fixture files — bash ok for non-code)

**Interfaces:**
- Produces: `Evidence(location, commit, lines)` record, enriched `ParsedIssue(issueId, title, body, location, priority, depends)`, enriched `ParsedResponse(..., evidence)`, `ParsedConfirmation(issueId, verdict, reason)`, JSONL reader with per-round markdown fallback

- [ ] **Step 1: Create JSONL fixture files**

Create fixture directory and JSONL files for a 3-round workspace:

```bash
mkdir -p server/runtime/src/test/resources/fixtures/workspace-replay-jsonl/responses
```

Create `reviewer-1.jsonl`:
```jsonl
{"event": "schema_version", "version": 1}
{"event": "issue_raised", "round": 1, "id": "R1-01", "title": "Missing error handling in parser", "body": "The parser does not handle malformed input.", "location": "§3.2", "priority": "HIGH", "depends": [], "scope": null}
{"event": "issue_raised", "round": 1, "id": "R1-02", "title": "API endpoint returns wrong status code", "body": "Returns 200 on validation failure.", "location": "§4.1", "priority": "MEDIUM", "depends": [], "scope": null}
{"event": "issue_raised", "round": 1, "id": "R1-03", "title": "Race condition in concurrent access", "body": "Multiple threads can modify shared state.", "location": null, "priority": "HIGH", "depends": ["R1-01"], "scope": null}
{"event": "assumption", "round": 1, "text": "All input files are UTF-8 encoded.", "source": "reviewer-1.md"}
{"event": "round_signal", "round": 1, "role": "reviewer", "signal": "CONTINUE", "description": null}
```

Create `implementor-1.jsonl`:
```jsonl
{"event": "schema_version", "version": 1}
{"event": "issue_fixed", "round": 1, "id": "R1-01", "sectionRef": "3.2", "evidence": [{"location": "§3.2", "commit": "abc123"}], "rationale": "Added try-catch blocks."}
{"event": "issue_rejected", "round": 1, "id": "R1-02", "rationale": "200 is intentional — response-envelope pattern."}
{"event": "issue_escalated", "round": 1, "id": "R1-03", "rationale": "Needs architectural decision."}
{"event": "round_signal", "round": 1, "role": "implementor", "signal": "CONTINUE", "description": null}
```

Create `reviewer-2.jsonl`:
```jsonl
{"event": "schema_version", "version": 1}
{"event": "confirmation", "round": 2, "id": "R1-01", "verdict": "resolved", "reason": ""}
{"event": "confirmation", "round": 2, "id": "R1-02", "verdict": "accepted", "reason": ""}
{"event": "confirmation", "round": 2, "id": "R1-03", "verdict": "contested", "reason": "needs architectural input"}
{"event": "issue_raised", "round": 2, "id": "R2-01", "title": "Test coverage below threshold", "body": "Coverage is 45%.", "location": null, "priority": "MEDIUM", "depends": [], "scope": null}
{"event": "settled_decision", "round": 2, "text": "Response-envelope pattern is the standard for this API", "fromIssue": "R1-02"}
{"event": "round_signal", "round": 2, "role": "reviewer", "signal": "APPROVED", "description": null}
```

Create `implementor-2.jsonl`:
```jsonl
{"event": "schema_version", "version": 1}
{"event": "issue_fixed", "round": 2, "id": "R2-01", "sectionRef": null, "evidence": [], "rationale": "Added tests."}
{"event": "round_signal", "round": 2, "role": "implementor", "signal": "CONTINUE", "description": null}
```

Create `reviewer-3.jsonl`:
```jsonl
{"event": "schema_version", "version": 1}
{"event": "confirmation", "round": 3, "id": "R2-01", "verdict": "resolved", "reason": ""}
{"event": "round_signal", "round": 3, "role": "reviewer", "signal": "APPROVED", "description": null}
```

Also copy the `.spec-path` and `.mode` from existing fixture:
```bash
cp server/runtime/src/test/resources/fixtures/workspace-replay/.spec-path server/runtime/src/test/resources/fixtures/workspace-replay-jsonl/
cp server/runtime/src/test/resources/fixtures/workspace-replay/.mode server/runtime/src/test/resources/fixtures/workspace-replay-jsonl/
echo "This is a test review context note." > server/runtime/src/test/resources/fixtures/workspace-replay-jsonl/context.md
```

- [ ] **Step 2: Write failing tests for JSONL parsing**

Use `ide_insert_member` to add test methods to `WorkspaceParserTest.java`:

```java
// New test class or additional tests in existing class

@Test
void jsonl_round_count() {
    Path fixture = Path.of("src/test/resources/fixtures/workspace-replay-jsonl");
    var result = WorkspaceParser.parse(fixture);
    assertEquals(3, result.rounds().size());
}

@Test
void jsonl_round1_issues_with_metadata() {
    Path fixture = Path.of("src/test/resources/fixtures/workspace-replay-jsonl");
    var result = WorkspaceParser.parse(fixture);
    var round1 = result.rounds().get(0);
    assertEquals(3, round1.issues().size());

    var r101 = round1.issues().get(0);
    assertEquals("R1-01", r101.issueId());
    assertEquals("§3.2", r101.location());
    assertEquals("HIGH", r101.priority());
    assertTrue(r101.depends().isEmpty());

    var r103 = round1.issues().get(2);
    assertEquals(List.of("R1-01"), r103.depends());
}

@Test
void jsonl_round1_responses_with_evidence() {
    Path fixture = Path.of("src/test/resources/fixtures/workspace-replay-jsonl");
    var result = WorkspaceParser.parse(fixture);
    var round1 = result.rounds().get(0);

    var resp1 = round1.responses().get(0);
    assertEquals("FIXED", resp1.status());
    assertEquals(1, resp1.evidence().size());
    assertEquals("§3.2", resp1.evidence().get(0).location());
    assertEquals("abc123", resp1.evidence().get(0).commit());
}

@Test
void jsonl_confirmations_use_verdict() {
    Path fixture = Path.of("src/test/resources/fixtures/workspace-replay-jsonl");
    var result = WorkspaceParser.parse(fixture);
    var round1 = result.rounds().get(0);

    // Confirmations from reviewer-2.jsonl, in round 2's ParsedRound
    var round2 = result.rounds().get(1);
    var confs = round2.confirmations();
    // R1-01, R1-02 confirmations are in round 2 (in-source-round model)
    // but R1-03 confirmation is also in round 2
    assertTrue(confs.stream().anyMatch(c -> c.issueId().equals("R1-01")
            && "resolved".equals(c.verdict())));
    assertTrue(confs.stream().anyMatch(c -> c.issueId().equals("R1-02")
            && "accepted".equals(c.verdict())));
}

@Test
void jsonl_signal_extraction() {
    Path fixture = Path.of("src/test/resources/fixtures/workspace-replay-jsonl");
    var result = WorkspaceParser.parse(fixture);
    assertEquals("CONTINUE", result.rounds().get(0).signal());
    assertEquals("APPROVED", result.rounds().get(1).signal());
}

@Test
void markdown_fallback_still_works() {
    // Existing fixture has no JSONL — should parse from markdown as before
    Path fixture = Path.of("src/test/resources/fixtures/workspace-replay");
    var result = WorkspaceParser.parse(fixture);
    assertEquals(3, result.rounds().size());
    assertEquals(3, result.rounds().get(0).issues().size());
}
```

- [ ] **Step 3: Run tests to verify they fail**

```bash
/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=WorkspaceParserTest
```

Expected: compilation errors — `Evidence` record not defined, `ParsedIssue` missing `location`/`priority`/`depends` fields, `ParsedConfirmation` missing `verdict`.

- [ ] **Step 4: Implement Java changes**

Use IntelliJ MCP for all edits:

1. Add `Evidence` record to `WorkspaceParser.java`:
```java
public record Evidence(String location, String commit, String lines) {}
```

2. Update `ParsedIssue` record:
```java
public record ParsedIssue(String issueId, String title, String body,
        String location, String priority, List<String> depends) {}
```

3. Update `ParsedResponse` record:
```java
public record ParsedResponse(
        String issueId, String status, String sectionRef,
        String rationale, String body, List<Evidence> evidence) {}
```

4. Replace `ParsedConfirmation` record:
```java
public record ParsedConfirmation(
        String issueId, String verdict, String reason) {}
```

5. Update all internal callers of `ParsedIssue` constructor — `extractNewIssues()` now needs to pass `null, "LOW", List.of()` for the new fields when parsing from markdown.

6. Update all internal callers of `ParsedResponse` constructor — pass `List.of()` for `evidence`.

7. Update `extractConfirmations()` to produce verdict instead of booleans:
```java
String statusText = m.group(3).toLowerCase();
String verdict;
if (statusText.contains("resolved") && !statusText.contains("still")) {
    verdict = "resolved";
} else if (statusText.contains("accepted")) {
    verdict = "accepted";
} else {
    verdict = "contested";
}
// reason extraction same as before, but only for "contested"
String reason = "";
if ("contested".equals(verdict)) {
    // ... existing reason extraction logic
}
confirmations.add(new ParsedConfirmation(issueId, verdict, reason));
```

8. Add `parseRoundFromJsonl()` method — reads JSONL files for a given round:
```java
private static ParsedRound parseRoundFromJsonl(Path responsesDir, int roundNum) {
    // Read reviewer-N.jsonl and implementor-N.jsonl
    // Deserialize each line by "event" type discriminator
    // Assemble ParsedRound from events
}
```

Implementation uses `javax.json` or Jackson `ObjectMapper` — whichever is on classpath. Parse each JSON line, switch on `event` field, build lists of issues, responses, confirmations, assumptions, settled decisions, and signal.

9. Add `hasJsonlForRound()` method:
```java
private static boolean hasJsonlForRound(Path responsesDir, int roundNum) {
    return Files.exists(responsesDir.resolve("reviewer-" + roundNum + ".jsonl"))
            || Files.exists(responsesDir.resolve("implementor-" + roundNum + ".jsonl"));
}
```

10. Update `parseRounds()` to use per-round fallback:
```java
private static List<ParsedRound> parseRounds(Path workspaceDir) {
    Path responsesDir = workspaceDir.resolve("responses");
    if (!Files.isDirectory(responsesDir)) return List.of();

    int maxRound = discoverMaxRound(responsesDir);
    List<ParsedRound> rounds = new ArrayList<>();

    for (int n = 1; n <= maxRound; n++) {
        if (hasJsonlForRound(responsesDir, n)) {
            rounds.add(parseRoundFromJsonl(responsesDir, n));
        } else {
            rounds.add(parseRoundFromMarkdown(responsesDir, n, maxRound));
        }
    }
    return rounds;
}
```

11. Extract existing markdown parsing into `parseRoundFromMarkdown()` — normalize confirmation routing to in-source-round model (confirmations stay in the round they're written in, not cross-routed).

12. Update `discoverMaxRound()` to also check for `.jsonl` files:
```java
private static int discoverMaxRound(Path responsesDir) {
    int max = 0;
    for (int n = 1; n <= 100; n++) {
        if (Files.exists(responsesDir.resolve("reviewer-" + n + ".md"))
                || Files.exists(responsesDir.resolve("implementor-" + n + ".md"))
                || Files.exists(responsesDir.resolve("reviewer-" + n + ".jsonl"))
                || Files.exists(responsesDir.resolve("implementor-" + n + ".jsonl"))) {
            max = n;
        } else {
            break;
        }
    }
    return max;
}
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=WorkspaceParserTest
```

Expected: all PASS.

- [ ] **Step 6: Commit**

```bash
git -C ~/claude/casehub/drafthouse add server/runtime/src/main/java/io/casehub/drafthouse/debate/WorkspaceParser.java server/runtime/src/test/java/io/casehub/drafthouse/debate/WorkspaceParserTest.java server/runtime/src/test/resources/fixtures/workspace-replay-jsonl/
git -C ~/claude/casehub/drafthouse commit -m "feat: JSONL reader with per-round fallback, enriched records, verdict confirmations (#96)

Refs #96"
```

---

### Task 6: Java adapter update (`WorkspaceReplayAdapter.java`)

**Files:**
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/debate/WorkspaceReplayAdapter.java` (use IntelliJ MCP)
- Modify: `server/runtime/src/test/java/io/casehub/drafthouse/debate/WorkspaceReplayAdapterTest.java` (use IntelliJ MCP)

**Interfaces:**
- Consumes: Enriched `ParsedIssue` with `location()`, `priority()` from Task 5; `ParsedConfirmation.verdict()` from Task 5; `ParsedResponse.evidence()` from Task 5

- [ ] **Step 1: Write failing test for enriched adapter behavior**

Add to `WorkspaceReplayAdapterTest.java`:

```java
@Test
void replay_passes_location_and_priority_from_jsonl() {
    Path fixture = Path.of("src/test/resources/fixtures/workspace-replay-jsonl");
    var parseResult = WorkspaceParser.parse(fixture);

    String channelName = "drafthouse/debate/jsonl-test-" + System.nanoTime();
    Channel channel = channelService.create(ChannelCreateRequest.builder(channelName)
            .description("test jsonl replay").semantic(ChannelSemantic.APPEND).build());

    DebateSession session = new DebateSession(
            channel.id(), channel.id().toString(), channel.name(), null);

    var adapter = new WorkspaceReplayAdapter(
            messageService, instanceService, channelGateway, eventBus);

    var result = adapter.replay(session, parseResult);
    assertTrue(result.entryCount() > 0);

    var projected = projectionService.project(channel.id(), debateProjection);
    ConversationState state = projected.state();
    assertNotNull(state);

    // R1-01 should have location and priority from JSONL
    var r101 = state.points().stream()
            .filter(p -> "R1-01".equals(p.correlationId()))
            .findFirst().orElse(null);
    assertNotNull(r101);
    assertEquals("§3.2", r101.classification().location());
    assertEquals(io.casehub.blocks.conversation.Priority.HIGH,
            r101.classification().priority());
}

@Test
void replay_uses_verdict_for_confirmation_dispatch() {
    Path fixture = Path.of("src/test/resources/fixtures/workspace-replay-jsonl");
    var parseResult = WorkspaceParser.parse(fixture);

    String channelName = "drafthouse/debate/verdict-test-" + System.nanoTime();
    Channel channel = channelService.create(ChannelCreateRequest.builder(channelName)
            .description("test verdict").semantic(ChannelSemantic.APPEND).build());

    DebateSession session = new DebateSession(
            channel.id(), channel.id().toString(), channel.name(), null);

    var adapter = new WorkspaceReplayAdapter(
            messageService, instanceService, channelGateway, eventBus);

    adapter.replay(session, parseResult);

    var projected = projectionService.project(channel.id(), debateProjection);
    ConversationState state = projected.state();

    // R1-01 should be VERIFIED (verdict=resolved)
    var r101 = state.points().stream()
            .filter(p -> "R1-01".equals(p.correlationId()))
            .findFirst().orElse(null);
    assertNotNull(r101);
    assertEquals(ConversationState.PointStatus.VERIFIED, r101.status());
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=WorkspaceReplayAdapterTest
```

Expected: compilation errors or assertion failures — adapter doesn't pass location/priority, confirmation dispatch still uses boolean methods.

- [ ] **Step 3: Implement adapter changes**

Use IntelliJ MCP:

1. Update RAISE section — use `issue.location()` directly instead of `extractLocation(body)`:
```java
// Replace extractLocation(issue.body()) with:
String location = issue.location();
if (location == null) {
    location = extractLocation(issue.body());
    if (location == null) {
        location = findLocationFromResponses(issue.issueId(), round.responses());
    }
}
String priority = issue.priority();

var meta = buildMeta("RAISE", "REV", n, priority, null, location);
```

2. Update confirmation section — replace boolean dispatch with verdict switch:
```java
for (var conf : round.confirmations()) {
    String entryType;
    MessageType msgType;
    String content;

    switch (conf.verdict()) {
        case "resolved" -> {
            entryType = "VERIFIED";
            msgType = MessageType.DONE;
            content = "Fix verified.";
        }
        case "accepted" -> {
            entryType = "AGREE";
            msgType = MessageType.DONE;
            content = "Rejection accepted.";
        }
        default -> {
            entryType = "DISPUTE";
            msgType = MessageType.DECLINE;
            content = conf.reason().isEmpty() ? "Still open." : conf.reason();
        }
    }

    var meta = buildMeta(entryType, "REV", n, null, null, null);
    // ... rest unchanged
}
```

3. Remove the `extractLocation()` helper method — no longer the primary path (kept as fallback for markdown-parsed issues without location field).

4. Update `findDeferredRound()` — replace `!conf.resolved() && !conf.accepted()` with `"contested".equals(conf.verdict())`.

- [ ] **Step 4: Run all tests**

```bash
/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime
```

Expected: all PASS — both WorkspaceParserTest and WorkspaceReplayAdapterTest.

- [ ] **Step 5: Commit**

```bash
git -C ~/claude/casehub/drafthouse add server/runtime/src/main/java/io/casehub/drafthouse/debate/WorkspaceReplayAdapter.java server/runtime/src/test/java/io/casehub/drafthouse/debate/WorkspaceReplayAdapterTest.java
git -C ~/claude/casehub/drafthouse commit -m "feat: adapter uses location/priority from enriched issues, verdict-based confirmation (#96)

Refs #96"
```
