# Replay Adapter Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #95 — Build replay adapter for completed design-review workspaces
**Issue group:** #95

**Goal:** Parse completed design-review workspace directories and replay them as debate sessions in DraftHouse via Qhorus channel dispatch.

**Architecture:** A pure `WorkspaceParser` extracts structured data from workspace markdown files using regex patterns ported from Python. A `WorkspaceReplayAdapter` orchestrator creates a Qhorus channel, encodes each parsed entry with `DHMETA:` sentinel format, and dispatches via `MessageService.dispatch()`. The existing `DebateChannelProjection` folds entries into `ConversationState`. A thin MCP tool `load_workspace` on `DebateMcpTools` exposes the functionality.

**Tech Stack:** Java 21, Quarkus 3.34.3, casehub-qhorus 0.2-SNAPSHOT, casehub-blocks 0.2-SNAPSHOT, Playwright (E2E), TypeScript (review tracker panel)

## Global Constraints

- All dispatch uses `MessageService.dispatch(MessageDispatch.builder()...build())` — never `ChannelService.dispatch()` (which does not exist)
- `DispatchResult.messageId()` provides the persisted message ID — use for `inReplyTo` linkage
- Channel names must start with `drafthouse/debate/` for `DebateChannelBackendFactory` routing
- `debateSessionId` must be a UUID string — `resolveSession()` calls `UUID.fromString()`
- All entries encoded via `ChannelMessageMeta.encode(DebateProtocol.META_SENTINEL, meta, content)`
- MCP tool errors returned as `"error: ..."` strings, never exceptions (PP-20260604-6e8d5d)
- `statusAfter()` must return null for unknown entry types (PP-20260610-a47ef5)
- Sender strings from `DebateSession.instanceId(role, sessionId)` — never hardcoded (PP-20260607-508f7b)
- `actorType(ActorType.AGENT)` on all dispatches
- Protocols: PP-20260607-508f7b, PP-20260610-a47ef5, PP-20260608-d94c7d, PP-20260604-6e8d5d

---

### Task 1: Add VERIFIED and DEFERRED to the domain model and projection

**Files:**
- Modify: `server/api/src/main/java/io/casehub/drafthouse/debate/EntryType.java`
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/debate/DebateChannelProjection.java`
- Modify: `server/runtime/src/main/webui/src/panels/drafthouse-review-tracker.js`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/debate/DebateChannelProjectionTest.java` (new)

**Interfaces:**
- Consumes: existing `EntryType` enum, `DebateChannelProjection`, review tracker panel
- Produces: `EntryType.VERIFIED`, `EntryType.DEFERRED` enum values; `statusAfter("VERIFIED") → "VERIFIED"`, `statusAfter("DEFERRED") → "DEFERRED"` in projection; `ENTRY_TO_STATUS.VERIFIED → 'VERIFIED'`, `ENTRY_TO_STATUS.DEFERRED → 'DEFERRED'` in JS panel

- [ ] **Step 1: Write failing test for VERIFIED status derivation**

Create `DebateChannelProjectionTest.java`:

```java
package io.casehub.drafthouse.debate;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class DebateChannelProjectionTest {

    private final DebateChannelProjection projection = new DebateChannelProjection();

    @Test
    void statusAfter_verified_returns_VERIFIED() {
        assertEquals("VERIFIED", projection.statusAfter("VERIFIED"));
    }

    @Test
    void statusAfter_deferred_returns_DEFERRED() {
        assertEquals("DEFERRED", projection.statusAfter("DEFERRED"));
    }

    @Test
    void statusAfter_unknown_returns_null() {
        assertNull(projection.statusAfter("UNKNOWN_FUTURE_TYPE"));
    }

    @Test
    void statusAfter_existing_types_unchanged() {
        assertEquals("AGREED", projection.statusAfter("AGREE"));
        assertEquals("ACTIVE", projection.statusAfter("COUNTER"));
        assertEquals("ACTIVE", projection.statusAfter("QUALIFY"));
        assertEquals("DISPUTED", projection.statusAfter("DISPUTE"));
        assertEquals("DECLINED", projection.statusAfter("DECLINED"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DebateChannelProjectionTest -DfailIfNoTests=false`
Expected: FAIL — `statusAfter("VERIFIED")` returns null

- [ ] **Step 3: Add VERIFIED and DEFERRED to EntryType enum**

Add to `server/api/src/main/java/io/casehub/drafthouse/debate/EntryType.java` after `DECLINED`:

```java
VERIFIED, DEFERRED,
```

Full enum line becomes:
```java
RAISE, AGREE, COUNTER, DISPUTE, QUALIFY, FLAG_HUMAN, DECLINED,
VERIFIED, DEFERRED,
```

- [ ] **Step 4: Add VERIFIED and DEFERRED to DebateChannelProjection.statusAfter()**

Replace the `statusAfter` method body in `DebateChannelProjection.java`:

```java
return switch (entryType) {
    case "AGREE" -> "AGREED";
    case "COUNTER", "QUALIFY" -> "ACTIVE";
    case "DISPUTE" -> "DISPUTED";
    case "DECLINED" -> "DECLINED";
    case "VERIFIED" -> "VERIFIED";
    case "DEFERRED" -> "DEFERRED";
    default -> null;
};
```

- [ ] **Step 5: Add VERIFIED and DEFERRED to DEBATE_CONFIG**

Update `DEBATE_CONFIG` in `DebateChannelProjection.java`:

Status emojis — add `"VERIFIED", "✅"` and `"DEFERRED", "⏸"` to the map.

Resolved statuses — change to `Set.of("AGREED", "DECLINED", "VERIFIED", "DEFERRED")`.

Entry type labels — add `"VERIFIED", "verified"` and `"DEFERRED", "deferred"` to the map.

Note: `Map.of()` supports max 10 entries. The entryTypeLabel map will have 9 entries after this change (RAISE, AGREE, COUNTER, DISPUTE, QUALIFY, FLAG_HUMAN, DECLINED, VERIFIED, DEFERRED). The statusEmoji map will have 8 entries (OPEN, ACTIVE, AGREED, ESCALATED, DECLINED, DISPUTED, VERIFIED, DEFERRED). Both exceed `Map.of()` 10-entry limit — use `Map.ofEntries()`:

```java
.statusEmoji(Map.ofEntries(
        Map.entry("OPEN", "🔴"),
        Map.entry("ACTIVE", "🟡"),
        Map.entry("AGREED", "✅"),
        Map.entry("ESCALATED", "🔵"),
        Map.entry("DECLINED", "🚫"),
        Map.entry("DISPUTED", "⚡"),
        Map.entry("VERIFIED", "✅"),
        Map.entry("DEFERRED", "⏸")))
```

```java
.resolvedStatuses(Set.of("AGREED", "DECLINED", "VERIFIED", "DEFERRED"))
```

```java
.entryTypeLabel(Map.ofEntries(
        Map.entry("RAISE", "raised"),
        Map.entry("AGREE", "agreed"),
        Map.entry("COUNTER", "countered"),
        Map.entry("DISPUTE", "disputed"),
        Map.entry("QUALIFY", "qualified"),
        Map.entry("FLAG_HUMAN", "flag"),
        Map.entry("DECLINED", "declined"),
        Map.entry("VERIFIED", "verified"),
        Map.entry("DEFERRED", "deferred")))
```

- [ ] **Step 6: Update review tracker panel (TypeScript)**

In `server/runtime/src/main/webui/src/panels/drafthouse-review-tracker.js`:

Add to `ENTRY_TO_STATUS`:
```javascript
const ENTRY_TO_STATUS = {
  RAISE: 'OPEN',
  AGREE: 'AGREED',
  COUNTER: 'ACTIVE',
  DISPUTE: 'DISPUTED',
  QUALIFY: 'ACTIVE',
  FLAG_HUMAN: 'PENDING_HUMAN',
  DECLINED: 'DECLINED',
  VERIFIED: 'VERIFIED',
  DEFERRED: 'DEFERRED',
};
```

Add to `STATUS_ORDER`:
```javascript
const STATUS_ORDER = {
  OPEN: 0,
  PENDING_HUMAN: 1,
  ACTIVE: 2,
  DISPUTED: 3,
  AGREED: 4,
  DECLINED: 5,
  VERIFIED: 6,
  DEFERRED: 7,
};
```

Add to `STATUS_ICON`:
```javascript
const STATUS_ICON = {
  OPEN: '○',
  ACTIVE: '⟳',
  AGREED: '✓',
  PENDING_HUMAN: '⚑',
  DECLINED: '✓',
  DISPUTED: '✕',
  VERIFIED: '✓✓',
  DEFERRED: '⏸',
};
```

Find the resolved status check in `#derivePoints()` — it checks `status === 'AGREED' || status === 'DECLINED'`. Add `VERIFIED` and `DEFERRED`:

```javascript
const isResolved = status === 'AGREED' || status === 'DECLINED' || status === 'VERIFIED' || status === 'DEFERRED';
```

- [ ] **Step 7: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DebateChannelProjectionTest`
Expected: PASS — all four tests green

- [ ] **Step 8: Commit**

```bash
git add server/api/src/main/java/io/casehub/drafthouse/debate/EntryType.java \
       server/runtime/src/main/java/io/casehub/drafthouse/debate/DebateChannelProjection.java \
       server/runtime/src/main/webui/src/panels/drafthouse-review-tracker.js \
       server/runtime/src/test/java/io/casehub/drafthouse/debate/DebateChannelProjectionTest.java
git commit -m "feat(#95): add VERIFIED and DEFERRED to domain model, projection, and tracker panel

Refs #95"
```

---

### Task 2: WorkspaceParser — regex extraction from workspace files

**Files:**
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/debate/WorkspaceParser.java`
- Create: `server/runtime/src/test/java/io/casehub/drafthouse/debate/WorkspaceParserTest.java`
- Create: `server/runtime/src/test/resources/fixtures/workspace-replay/` (fixture workspace)

**Interfaces:**
- Consumes: filesystem paths (workspace directory)
- Produces: `WorkspaceParser.parse(Path workspaceDir) → WorkspaceParseResult`; inner records `WorkspaceParseResult`, `ParsedRound`, `ParsedIssue`, `ParsedResponse`, `ParsedConfirmation`, `ParsedSettledDecision`, `ParsedTrackerEntry`

This is the largest task — all regex porting, file reading, and round discovery. The parser is a pure function with no DraftHouse dependencies.

- [ ] **Step 1: Create fixture workspace**

Create the directory `server/runtime/src/test/resources/fixtures/workspace-replay/` with these files:

**`.spec-path`:** `/tmp/test-spec.md`

**`.mode`:** `spec-review`

**`context.md`:**
```markdown
This is a test review context note.
```

**`responses/reviewer-1.md`:**
```markdown
# Round 1 — Reviewer

## Overview

Brief overview of findings.

---

### R1-01: Missing error handling in parser

The parser does not handle malformed input. Section §3.2 describes the expected format but the implementation silently drops invalid lines.

ASSUMPTION: All input files are UTF-8 encoded.

---

### R1-02: API endpoint returns wrong status code

The /api/parse endpoint returns 200 on validation failure. Section 4.1 specifies that validation errors should return 422.

---

### R1-03: Race condition in concurrent access

Multiple threads can modify the shared state without synchronization.

SIGNAL: CONTINUE
```

**`responses/implementor-1.md`:**
```markdown
# Round 1 — Implementor

### R1-01: FIXED

Added try-catch blocks around all parsing operations. Invalid lines are now logged and skipped.

§3.2 updated with error handling specification.

### R1-02: REJECTED

The 200 status code is intentional. The API follows a response-envelope pattern where the HTTP status always indicates transport success. Validation errors are in the response body.

### R1-03: ESCALATED

This requires an architectural decision about the concurrency model. Flagging for human review.

SIGNAL: CONTINUE
```

**`responses/reviewer-2.md`:**
```markdown
# Round 2 — Reviewer

## Addressed Items

- R1-01: resolved — error handling is comprehensive
- R1-02: accepted — response-envelope pattern is documented
- R1-03: still open — needs architectural input, but no new information provided

## New Issues

### R2-01: Test coverage below threshold

Unit test coverage for the parser module is at 45%, below the 80% project minimum.

SETTLED: Response-envelope pattern is the standard for this API (from R1-02)

SIGNAL: APPROVED
```

**`responses/implementor-2.md`:**
```markdown
# Round 2 — Implementor

### R2-01: FIXED

Added comprehensive unit tests for all parser edge cases. Coverage now at 87%.

SIGNAL: APPROVED
```

**`responses/reviewer-3.md`:**
```markdown
# Round 3 — Reviewer

## Addressed Items

- R2-01: resolved — coverage is now above threshold
- R1-03: still open — no progress on concurrency model

SIGNAL: APPROVED
```

**`tracker.md`:**
```markdown
# Design Review Tracker

Spec: spec.md | Project: test-project
Started: 2026-07-06 | Current round: 3

## Issues

### R1-01: Missing error handling in parser
- **Raised:** Round 1
- **Status:** VERIFIED
- **Spec commit:** abc123 → def456

### R1-02: API endpoint returns wrong status code
- **Raised:** Round 1
- **Status:** ACCEPTED

### R1-03: Race condition in concurrent access
- **Raised:** Round 1
- **Status:** DEFERRED

### R2-01: Test coverage below threshold
- **Raised:** Round 2
- **Status:** VERIFIED
- **Spec commit:** ghi789 → jkl012
```

- [ ] **Step 2: Write failing tests for issue extraction**

Create `WorkspaceParserTest.java`:

```java
package io.casehub.drafthouse.debate;

import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import java.nio.file.Path;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

class WorkspaceParserTest {

    private static WorkspaceParser.WorkspaceParseResult result;

    @BeforeAll
    static void parse() {
        Path fixture = Path.of("src/test/resources/fixtures/workspace-replay");
        result = WorkspaceParser.parse(fixture);
    }

    @Test
    void metadata_extracted() {
        assertEquals("/tmp/test-spec.md", result.specPath());
        assertEquals("spec-review", result.mode());
        assertEquals("This is a test review context note.", result.contextNote().trim());
    }

    @Test
    void round_count() {
        assertEquals(3, result.rounds().size());
    }

    @Test
    void round1_issues() {
        var round1 = result.rounds().get(0);
        assertEquals(1, round1.roundNumber());
        assertEquals(3, round1.issues().size());

        var r101 = round1.issues().get(0);
        assertEquals("R1-01", r101.issueId());
        assertEquals("Missing error handling in parser", r101.title());
        assertTrue(r101.body().contains("parser does not handle malformed input"));
    }

    @Test
    void round1_responses() {
        var round1 = result.rounds().get(0);
        assertEquals(3, round1.responses().size());

        var resp1 = round1.responses().get(0);
        assertEquals("R1-01", resp1.issueId());
        assertEquals("FIXED", resp1.status());
        assertEquals("3.2", resp1.sectionRef());

        var resp2 = round1.responses().get(1);
        assertEquals("R1-02", resp2.issueId());
        assertEquals("REJECTED", resp2.status());

        var resp3 = round1.responses().get(2);
        assertEquals("R1-03", resp3.issueId());
        assertEquals("ESCALATED", resp3.status());
    }

    @Test
    void round1_confirmations_from_reviewer2() {
        var round1 = result.rounds().get(0);
        assertEquals(3, round1.confirmations().size());

        var c1 = round1.confirmations().get(0);
        assertEquals("R1-01", c1.issueId());
        assertTrue(c1.resolved());
        assertFalse(c1.accepted());

        var c2 = round1.confirmations().get(1);
        assertEquals("R1-02", c2.issueId());
        assertFalse(c2.resolved());
        assertTrue(c2.accepted());

        var c3 = round1.confirmations().get(2);
        assertEquals("R1-03", c3.issueId());
        assertFalse(c3.resolved());
        assertFalse(c3.accepted());
    }

    @Test
    void round2_issues() {
        var round2 = result.rounds().get(1);
        assertEquals(2, round2.roundNumber());
        assertEquals(1, round2.issues().size());
        assertEquals("R2-01", round2.issues().get(0).issueId());
        assertEquals("Test coverage below threshold", round2.issues().get(0).title());
    }

    @Test
    void round2_confirmations_from_reviewer3() {
        var round2 = result.rounds().get(1);
        assertEquals(1, round2.confirmations().size());
        var c = round2.confirmations().get(0);
        assertEquals("R2-01", c.issueId());
        assertTrue(c.resolved());
    }

    @Test
    void signal_extraction() {
        assertEquals("CONTINUE", result.rounds().get(0).signal());
        assertEquals("APPROVED", result.rounds().get(1).signal());
        assertEquals("APPROVED", result.rounds().get(2).signal());
    }

    @Test
    void assumption_extraction() {
        var round1 = result.rounds().get(0);
        assertEquals(1, round1.assumptions().size());
        assertEquals("All input files are UTF-8 encoded.", round1.assumptions().get(0));
    }

    @Test
    void settled_decision_extraction() {
        var round2 = result.rounds().get(1);
        assertEquals(1, round2.settledDecisions().size());
        var sd = round2.settledDecisions().get(0);
        assertEquals("Response-envelope pattern is the standard for this API", sd.text());
        assertEquals("R1-02", sd.fromIssue());
    }

    @Test
    void tracker_statuses() {
        assertEquals(4, result.trackerStatuses().size());

        var t1 = result.trackerStatuses().get(0);
        assertEquals("R1-01", t1.issueId());
        assertEquals("Missing error handling in parser", t1.title());
        assertEquals("VERIFIED", t1.status());
        assertEquals("abc123 → def456", t1.evidence());

        var t2 = result.trackerStatuses().get(1);
        assertEquals("R1-02", t2.issueId());
        assertEquals("ACCEPTED", t2.status());
        assertNull(t2.evidence());

        var t3 = result.trackerStatuses().get(2);
        assertEquals("R1-03", t3.issueId());
        assertEquals("DEFERRED", t3.status());

        var t4 = result.trackerStatuses().get(3);
        assertEquals("R2-01", t4.issueId());
        assertEquals("VERIFIED", t4.status());
    }

    @Test
    void known_section_headings_skipped() {
        // "Overview" and "Addressed Items" are known sections — not extracted as issues
        var allIssueIds = result.rounds().stream()
                .flatMap(r -> r.issues().stream())
                .map(WorkspaceParser.ParsedIssue::issueId)
                .toList();
        assertEquals(List.of("R1-01", "R1-02", "R1-03", "R2-01"), allIssueIds);
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=WorkspaceParserTest -DfailIfNoTests=false`
Expected: FAIL — `WorkspaceParser` class does not exist

- [ ] **Step 4: Implement WorkspaceParser**

Create `server/runtime/src/main/java/io/casehub/drafthouse/debate/WorkspaceParser.java`:

```java
package io.casehub.drafthouse.debate;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.*;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public final class WorkspaceParser {

    // ── Records ──────────────────────────────────────────────────────────────

    public record WorkspaceParseResult(
            String specPath,
            String mode,
            String contextNote,
            List<ParsedRound> rounds,
            List<ParsedTrackerEntry> trackerStatuses) {}

    public record ParsedRound(
            int roundNumber,
            String signal,
            String signalDescription,
            List<ParsedIssue> issues,
            List<ParsedResponse> responses,
            List<ParsedConfirmation> confirmations,
            List<String> assumptions,
            List<ParsedSettledDecision> settledDecisions) {}

    public record ParsedIssue(String issueId, String title, String body) {}

    public record ParsedResponse(
            String issueId,
            String status,
            String sectionRef,
            String rationale,
            String body) {}

    public record ParsedConfirmation(
            String issueId,
            boolean resolved,
            boolean accepted,
            String reason) {}

    public record ParsedSettledDecision(String text, String fromIssue) {}

    public record ParsedTrackerEntry(
            String issueId,
            String title,
            String status,
            String evidence) {}

    // ── Regex patterns (ported from parser.py) ───────────────────────────────

    private static final Pattern SIGNAL_RE = Pattern.compile(
            "^\\s*SIGNAL\\s*[:\\s]+\\s*(APPROVED|CONTINUE|DECISION_NEEDED)\\b\\s*[.:]*\\s*(.*?)\\s*$",
            Pattern.CASE_INSENSITIVE | Pattern.MULTILINE);

    private static final Pattern HEADING_RE = Pattern.compile(
            "^(#{2,3})\\s+(.+?)\\s*$", Pattern.MULTILINE);

    private static final Pattern ISSUE_ID_RE = Pattern.compile("R(\\d+)-(\\d+)");

    private static final Pattern ISSUE_RESPONSE_RE = Pattern.compile(
            "^#{2,3}\\s+R(\\d+)-(\\d+)\\s*[:\\s—\\-]+\\s*(FIXED|REJECTED|ESCALATED)\\b",
            Pattern.CASE_INSENSITIVE | Pattern.MULTILINE);

    private static final Pattern CONFIRMATION_RE = Pattern.compile(
            "R(\\d+)-(\\d+)\\b[^#\\n]*?\\b(resolved|accepted|still\\s+open)\\b",
            Pattern.CASE_INSENSITIVE);

    private static final Pattern SECTION_REF_RE = Pattern.compile(
            "§(\\d+(?:\\.\\d+)*)|[Ss]ection\\s+(\\d+(?:\\.\\d+)*)");

    private static final Pattern ASSUMPTION_RE = Pattern.compile(
            "^ASSUMPTION:\\s*(.+)$", Pattern.MULTILINE);

    private static final Pattern SETTLED_RE = Pattern.compile(
            "^SETTLED:\\s*(.+?)(?:\\(from\\s+(R\\d+-\\d+)\\))?\\s*$", Pattern.MULTILINE);

    private static final Pattern SIGNAL_SEPARATOR = Pattern.compile("\\n---\\s*\\n");

    private static final Pattern TRACKER_HEADING_RE = Pattern.compile(
            "^###\\s+(R\\d+-\\d+):\\s+(.+)$", Pattern.MULTILINE);

    private static final Pattern TRACKER_STATUS_RE = Pattern.compile(
            "^-\\s+\\*\\*Status:\\*\\*\\s+(\\w+)", Pattern.MULTILINE);

    private static final Pattern TRACKER_EVIDENCE_RE = Pattern.compile(
            "^-\\s+\\*\\*Spec commit:\\*\\*\\s*(.*)", Pattern.MULTILINE);

    private static final Set<String> KNOWN_SECTIONS = Set.of(
            "addressed items", "assumptions", "settled decisions", "signals", "signal",
            "summary", "overview", "overall assessment", "final assessment", "final sweep",
            "final verification", "final scan", "verdict", "strengths", "background",
            "context", "conclusion", "next steps", "notes", "references",
            "critical issues", "design issues", "completeness gaps", "completeness",
            "verified correct", "new issues", "items resolved with remaining concerns");

    private WorkspaceParser() {}

    // ── Public API ───────────────────────────────────────────────────────────

    public static WorkspaceParseResult parse(Path workspaceDir) {
        String specPath = readFileOrNull(workspaceDir.resolve(".spec-path"));
        String mode = readFileOrNull(workspaceDir.resolve(".mode"));
        String contextNote = readFileOrNull(workspaceDir.resolve("context.md"));

        List<ParsedRound> rounds = parseRounds(workspaceDir);
        List<ParsedTrackerEntry> trackerStatuses = parseTracker(workspaceDir);

        return new WorkspaceParseResult(
                specPath != null ? specPath.trim() : null,
                mode != null ? mode.trim() : null,
                contextNote,
                rounds,
                trackerStatuses);
    }

    // ── Round parsing ────────────────────────────────────────────────────────

    private static List<ParsedRound> parseRounds(Path workspaceDir) {
        Path responsesDir = workspaceDir.resolve("responses");
        if (!Files.isDirectory(responsesDir)) return List.of();

        List<ParsedRound> rounds = new ArrayList<>();
        Set<String> allExistingIds = new HashSet<>();

        int maxRound = discoverMaxRound(responsesDir);
        for (int n = 1; n <= maxRound; n++) {
            String reviewerContent = readFileOrNull(responsesDir.resolve("reviewer-" + n + ".md"));
            String implementorContent = readFileOrNull(responsesDir.resolve("implementor-" + n + ".md"));
            String nextReviewerContent = readFileOrNull(responsesDir.resolve("reviewer-" + (n + 1) + ".md"));

            List<ParsedIssue> issues = reviewerContent != null
                    ? extractNewIssues(reviewerContent, n, allExistingIds) : List.of();
            issues.forEach(i -> allExistingIds.add(i.issueId()));

            List<ParsedResponse> responses = implementorContent != null
                    ? extractIssueResponses(implementorContent) : List.of();

            List<ParsedConfirmation> confirmations = nextReviewerContent != null
                    ? extractConfirmations(nextReviewerContent) : List.of();

            String signal = "CONTINUE";
            String signalDescription = null;
            if (reviewerContent != null) {
                var sig = extractSignal(reviewerContent);
                signal = sig[0];
                signalDescription = sig[1];
            }

            List<String> assumptions = reviewerContent != null
                    ? extractAssumptions(reviewerContent) : List.of();

            List<ParsedSettledDecision> settled = List.of();
            if (reviewerContent != null) settled = extractSettledDecisions(reviewerContent);
            if (settled.isEmpty() && implementorContent != null) {
                settled = extractSettledDecisions(implementorContent);
            }

            rounds.add(new ParsedRound(n, signal, signalDescription,
                    issues, responses, confirmations, assumptions, settled));
        }

        return rounds;
    }

    private static int discoverMaxRound(Path responsesDir) {
        int max = 0;
        for (int n = 1; n <= 100; n++) {
            if (Files.exists(responsesDir.resolve("reviewer-" + n + ".md"))
                    || Files.exists(responsesDir.resolve("implementor-" + n + ".md"))) {
                max = n;
            } else {
                break;
            }
        }
        return max;
    }

    // ── Issue extraction ─────────────────────────────────────────────────────

    static List<ParsedIssue> extractNewIssues(String content, int roundNum, Set<String> existingIds) {
        List<ParsedIssue> issues = new ArrayList<>();
        Matcher m = HEADING_RE.matcher(content);
        List<int[]> headings = new ArrayList<>();
        List<String> titles = new ArrayList<>();

        while (m.find()) {
            headings.add(new int[]{m.start(), m.end()});
            titles.add(m.group(2));
        }

        int seq = 1;
        for (int i = 0; i < headings.size(); i++) {
            String title = titles.get(i);
            String titleLower = title.replaceAll(":.*$", "").trim().toLowerCase();
            if (KNOWN_SECTIONS.contains(titleLower)) continue;

            Matcher idCheck = ISSUE_ID_RE.matcher(title);
            if (idCheck.find() && existingIds.contains(idCheck.group())) continue;

            int bodyStart = headings.get(i)[1] + 1;
            int bodyEnd = (i + 1 < headings.size()) ? headings.get(i + 1)[0] : content.length();
            String body = content.substring(bodyStart, bodyEnd).trim();

            String[] parts = SIGNAL_SEPARATOR.split(body, 2);
            body = parts[0].trim();

            String issueId = String.format("R%d-%02d", roundNum, seq++);
            issues.add(new ParsedIssue(issueId, title, body));
        }

        return issues;
    }

    // ── Response extraction ──────────────────────────────────────────────────

    static List<ParsedResponse> extractIssueResponses(String content) {
        List<ParsedResponse> responses = new ArrayList<>();
        Matcher m = ISSUE_RESPONSE_RE.matcher(content);
        List<int[]> matches = new ArrayList<>();
        List<String[]> parsed = new ArrayList<>();

        while (m.find()) {
            matches.add(new int[]{m.start(), m.end()});
            parsed.add(new String[]{
                    "R" + m.group(1) + "-" + String.format("%02d", Integer.parseInt(m.group(2))),
                    m.group(3).toUpperCase()
            });
        }

        for (int i = 0; i < matches.size(); i++) {
            int bodyStart = matches.get(i)[1] + 1;
            int bodyEnd = (i + 1 < matches.size()) ? matches.get(i + 1)[0] : content.length();
            String body = content.substring(bodyStart, bodyEnd).trim();

            String[] signalParts = SIGNAL_SEPARATOR.split(body, 2);
            body = signalParts[0].trim();

            body = Pattern.compile("^SETTLED:", Pattern.MULTILINE)
                    .split(body, 2)[0].trim();

            String sectionRef = null;
            Matcher refMatch = SECTION_REF_RE.matcher(body);
            if (refMatch.find()) {
                sectionRef = refMatch.group(1) != null ? refMatch.group(1) : refMatch.group(2);
            }

            String status = parsed.get(i)[1];
            String rationale = ("REJECTED".equals(status) || "ESCALATED".equals(status)) ? body : "";

            responses.add(new ParsedResponse(parsed.get(i)[0], status, sectionRef, rationale, body));
        }

        return responses;
    }

    // ── Confirmation extraction ──────────────────────────────────────────────

    static List<ParsedConfirmation> extractConfirmations(String content) {
        List<ParsedConfirmation> confirmations = new ArrayList<>();
        Matcher m = CONFIRMATION_RE.matcher(content);

        while (m.find()) {
            String issueId = "R" + m.group(1) + "-" + String.format("%02d", Integer.parseInt(m.group(2)));
            String statusText = m.group(3).toLowerCase();
            boolean resolved = statusText.contains("resolved") && !statusText.contains("still");
            boolean accepted = statusText.contains("accepted");

            String reason = "";
            if (!resolved && !accepted) {
                int afterMatch = m.end();
                int lineEnd = content.indexOf('\n', afterMatch);
                if (lineEnd < 0) lineEnd = content.length();
                reason = content.substring(afterMatch, lineEnd)
                        .replaceAll("^[\\s—\\-:]+", "").trim();
            }

            confirmations.add(new ParsedConfirmation(issueId, resolved, accepted, reason));
        }

        return confirmations;
    }

    // ── Signal extraction ────────────────────────────────────────────────────

    static String[] extractSignal(String content) {
        String[] lines = content.split("\n");
        int searchFrom = Math.max(0, lines.length - 10);
        String lastTenLines = String.join("\n", Arrays.copyOfRange(lines, searchFrom, lines.length));

        Matcher m = SIGNAL_RE.matcher(lastTenLines);
        String signal = "CONTINUE";
        String description = null;

        while (m.find()) {
            signal = m.group(1).toUpperCase();
            description = "DECISION_NEEDED".equals(signal) ? m.group(2) : null;
        }

        if ("CONTINUE".equals(signal)) {
            m = SIGNAL_RE.matcher(content);
            while (m.find()) {
                signal = m.group(1).toUpperCase();
                description = "DECISION_NEEDED".equals(signal) ? m.group(2) : null;
            }
        }

        return new String[]{signal, description};
    }

    // ── Assumption / settled extraction ───────────────────────────────────────

    static List<String> extractAssumptions(String content) {
        List<String> assumptions = new ArrayList<>();
        Matcher m = ASSUMPTION_RE.matcher(content);
        while (m.find()) assumptions.add(m.group(1).trim());
        return assumptions;
    }

    static List<ParsedSettledDecision> extractSettledDecisions(String content) {
        List<ParsedSettledDecision> decisions = new ArrayList<>();
        Matcher m = SETTLED_RE.matcher(content);
        while (m.find()) {
            decisions.add(new ParsedSettledDecision(
                    m.group(1).trim(),
                    m.group(2) != null ? m.group(2) : ""));
        }
        return decisions;
    }

    // ── Tracker parsing ──────────────────────────────────────────────────────

    static List<ParsedTrackerEntry> parseTracker(Path workspaceDir) {
        String content = readFileOrNull(workspaceDir.resolve("tracker.md"));
        if (content == null) return List.of();

        List<ParsedTrackerEntry> entries = new ArrayList<>();
        Matcher headingMatcher = TRACKER_HEADING_RE.matcher(content);

        List<int[]> headingPositions = new ArrayList<>();
        List<String[]> headingData = new ArrayList<>();
        while (headingMatcher.find()) {
            headingPositions.add(new int[]{headingMatcher.start(), headingMatcher.end()});
            headingData.add(new String[]{headingMatcher.group(1), headingMatcher.group(2).trim()});
        }

        for (int i = 0; i < headingPositions.size(); i++) {
            int sectionStart = headingPositions.get(i)[1];
            int sectionEnd = (i + 1 < headingPositions.size())
                    ? headingPositions.get(i + 1)[0] : content.length();
            String section = content.substring(sectionStart, sectionEnd);

            String status = null;
            Matcher sm = TRACKER_STATUS_RE.matcher(section);
            if (sm.find()) status = sm.group(1);

            String evidence = null;
            Matcher em = TRACKER_EVIDENCE_RE.matcher(section);
            if (em.find()) {
                String raw = em.group(1).trim();
                evidence = raw.isEmpty() ? null : raw;
            }

            entries.add(new ParsedTrackerEntry(
                    headingData.get(i)[0], headingData.get(i)[1], status, evidence));
        }

        return entries;
    }

    // ── Utilities ────────────────────────────────────────────────────────────

    private static String readFileOrNull(Path path) {
        try {
            return Files.exists(path) ? Files.readString(path) : null;
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=WorkspaceParserTest`
Expected: PASS — all tests green

- [ ] **Step 6: Commit**

```bash
git add server/runtime/src/main/java/io/casehub/drafthouse/debate/WorkspaceParser.java \
       server/runtime/src/test/java/io/casehub/drafthouse/debate/WorkspaceParserTest.java \
       server/runtime/src/test/resources/fixtures/workspace-replay/
git commit -m "feat(#95): workspace parser — regex extraction from design-review files

Ports 8 Python regex patterns to Java. Extracts issues, responses,
confirmations, assumptions, settled decisions, signals, and tracker
terminal statuses from workspace markdown files.

Refs #95"
```

---

### Task 3: WorkspaceReplayAdapter — channel dispatch orchestrator

**Files:**
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/debate/WorkspaceReplayAdapter.java`
- Create: `server/runtime/src/test/java/io/casehub/drafthouse/debate/WorkspaceReplayAdapterTest.java`

**Interfaces:**
- Consumes: `WorkspaceParser.WorkspaceParseResult`, `ChannelService.create()`, `MessageService.dispatch()`, `MessageService.pollAfter()`, `InstanceService.register()`, `WebSocketEventBus.pushDebateEntries()`, `DebateSessionRegistry`, `ChannelGateway.initChannel()`, `DebateStreamEntry.from(Message)`, `ChannelMessageMeta.encode()`, `DebateProtocol.META_SENTINEL`, `ConversationProtocol.*` constants
- Produces: `WorkspaceReplayAdapter.replay(DebateSession session, WorkspaceParseResult parseResult) → ReplayResult`; record `ReplayResult(int entryCount, Map<String, String> statusDistribution)`

- [ ] **Step 1: Write failing integration test**

Create `WorkspaceReplayAdapterTest.java`:

```java
package io.casehub.drafthouse.debate;

import io.casehub.blocks.conversation.ConversationState;
import io.casehub.drafthouse.DebateSession;
import io.casehub.drafthouse.DebateSessionRegistry;
import io.casehub.drafthouse.WebSocketEventBus;
import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelCreateRequest;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.instance.InstanceService;
import io.casehub.qhorus.runtime.message.MessageService;
import io.casehub.qhorus.runtime.projection.ProjectionService;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.nio.file.Path;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class WorkspaceReplayAdapterTest {

    @Inject ChannelService channelService;
    @Inject MessageService messageService;
    @Inject InstanceService instanceService;
    @Inject ChannelGateway channelGateway;
    @Inject ProjectionService projectionService;
    @Inject DebateChannelProjection debateProjection;
    @Inject DebateSessionRegistry registry;
    @Inject WebSocketEventBus eventBus;

    @Test
    void replay_creates_entries_with_correct_statuses() {
        Path fixture = Path.of("src/test/resources/fixtures/workspace-replay");
        var parseResult = WorkspaceParser.parse(fixture);

        String channelName = "drafthouse/debate/replay-test-" + System.nanoTime();
        Channel channel = channelService.create(ChannelCreateRequest.builder(channelName)
                .description("test replay").semantic(ChannelSemantic.APPEND).build());

        DebateSession session = new DebateSession(
                channel.id(), channel.id().toString(), channel.name(), null);

        var adapter = new WorkspaceReplayAdapter(
                messageService, instanceService, channelGateway, eventBus);

        var result = adapter.replay(session, parseResult);

        assertTrue(result.entryCount() > 0, "should have dispatched entries");

        var projected = projectionService.project(channel.id(), debateProjection);
        ConversationState state = projected.state();

        assertNotNull(state);
        assertFalse(state.points().isEmpty(), "should have conversation points");
        assertEquals(4, state.points().size(), "fixture has 4 issues");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=WorkspaceReplayAdapterTest -DfailIfNoTests=false`
Expected: FAIL — `WorkspaceReplayAdapter` does not exist

- [ ] **Step 3: Implement WorkspaceReplayAdapter**

Create `server/runtime/src/main/java/io/casehub/drafthouse/debate/WorkspaceReplayAdapter.java`:

```java
package io.casehub.drafthouse.debate;

import io.casehub.blocks.channel.ChannelMessageMeta;
import io.casehub.blocks.conversation.ConversationProtocol;
import io.casehub.drafthouse.DebateSession;
import io.casehub.drafthouse.WebSocketEventBus;
import io.casehub.qhorus.api.message.ActorType;
import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.instance.InstanceService;
import io.casehub.qhorus.runtime.message.MessageService;

import java.util.*;
import java.util.logging.Level;
import java.util.logging.Logger;

public class WorkspaceReplayAdapter {

    private static final Logger LOG = Logger.getLogger(WorkspaceReplayAdapter.class.getName());

    public record ReplayResult(int entryCount, Map<String, String> statusDistribution) {}

    private final MessageService messageService;
    private final InstanceService instanceService;
    private final ChannelGateway channelGateway;
    private final WebSocketEventBus eventBus;

    public WorkspaceReplayAdapter(MessageService messageService,
                                   InstanceService instanceService,
                                   ChannelGateway channelGateway,
                                   WebSocketEventBus eventBus) {
        this.messageService = messageService;
        this.instanceService = instanceService;
        this.channelGateway = channelGateway;
        this.eventBus = eventBus;
    }

    public ReplayResult replay(DebateSession session,
                                WorkspaceParser.WorkspaceParseResult parseResult) {

        UUID channelId = session.channelId();
        String revSender = registerSender(session, AgentType.REV);
        String impSender = registerSender(session, AgentType.IMP);

        channelGateway.initChannel(channelId,
                new io.casehub.qhorus.api.channel.ChannelRef(channelId, session.channelName()));

        Map<String, Long> raiseMessageIds = new HashMap<>();
        int entryCount = 0;

        for (var round : parseResult.rounds()) {
            int n = round.roundNumber();

            // 1. RAISE entries
            for (var issue : round.issues()) {
                String location = extractLocation(issue.body());
                if (location == null) {
                    location = findLocationFromResponses(issue.issueId(), round.responses());
                }

                var meta = buildMeta("RAISE", "REV", n, null, null, location);
                String encoded = ChannelMessageMeta.encode(
                        DebateProtocol.META_SENTINEL, meta, issue.title() + "\n\n" + issue.body());

                DispatchResult dr = messageService.dispatch(MessageDispatch.builder()
                        .channelId(channelId)
                        .sender(revSender)
                        .type(MessageType.QUERY)
                        .content(encoded)
                        .correlationId(issue.issueId())
                        .actorType(ActorType.AGENT)
                        .build());

                raiseMessageIds.put(issue.issueId(), dr.messageId());
                entryCount++;
            }

            // 2. QUALIFY / COUNTER / FLAG_HUMAN entries
            for (var resp : round.responses()) {
                String entryType = switch (resp.status()) {
                    case "FIXED" -> "QUALIFY";
                    case "REJECTED" -> "COUNTER";
                    case "ESCALATED" -> "FLAG_HUMAN";
                    default -> "QUALIFY";
                };
                MessageType msgType = switch (resp.status()) {
                    case "FIXED" -> MessageType.RESPONSE;
                    case "REJECTED" -> MessageType.RESPONSE;
                    case "ESCALATED" -> MessageType.HANDOFF;
                    default -> MessageType.RESPONSE;
                };

                var meta = buildMeta(entryType, "IMP", n, null, null, null);
                String content = resp.body().isEmpty() ? resp.rationale() : resp.body();
                String encoded = ChannelMessageMeta.encode(
                        DebateProtocol.META_SENTINEL, meta, content);

                Long inReplyTo = raiseMessageIds.get(resp.issueId());
                messageService.dispatch(MessageDispatch.builder()
                        .channelId(channelId)
                        .sender(impSender)
                        .type(msgType)
                        .content(encoded)
                        .correlationId(resp.issueId())
                        .inReplyTo(inReplyTo)
                        .actorType(ActorType.AGENT)
                        .build());
                entryCount++;
            }

            // 3. VERIFIED / DISPUTE / AGREE entries (from next round's reviewer)
            for (var conf : round.confirmations()) {
                String entryType;
                MessageType msgType;
                String content;

                if (conf.resolved()) {
                    entryType = "VERIFIED";
                    msgType = MessageType.DONE;
                    content = "Fix verified.";
                } else if (conf.accepted()) {
                    entryType = "AGREE";
                    msgType = MessageType.DONE;
                    content = "Rejection accepted.";
                } else {
                    entryType = "DISPUTE";
                    msgType = MessageType.DECLINE;
                    content = conf.reason().isEmpty() ? "Still open." : conf.reason();
                }

                var meta = buildMeta(entryType, "REV", n + 1, null, null, null);
                String encoded = ChannelMessageMeta.encode(
                        DebateProtocol.META_SENTINEL, meta, content);

                Long inReplyTo = raiseMessageIds.get(conf.issueId());
                messageService.dispatch(MessageDispatch.builder()
                        .channelId(channelId)
                        .sender(revSender)
                        .type(msgType)
                        .content(encoded)
                        .correlationId(conf.issueId())
                        .inReplyTo(inReplyTo)
                        .actorType(ActorType.AGENT)
                        .build());
                entryCount++;
            }

            // 4. MEMO entries (assumptions + settled decisions)
            for (String assumption : round.assumptions()) {
                entryCount += dispatchMemo(channelId, revSender, n,
                        "ASSUMPTION: " + assumption);
            }
            for (var sd : round.settledDecisions()) {
                String text = "SETTLED: " + sd.text();
                if (!sd.fromIssue().isEmpty()) text += " (from " + sd.fromIssue() + ")";
                entryCount += dispatchMemo(channelId, revSender, n, text);
            }
        }

        // 5. DEFERRED entries (from tracker terminal statuses)
        for (var te : parseResult.trackerStatuses()) {
            if ("DEFERRED".equals(te.status())) {
                int deferredRound = findDeferredRound(te.issueId(), parseResult);
                var meta = buildMeta("DEFERRED", "REV", deferredRound, null, null, null);
                String encoded = ChannelMessageMeta.encode(
                        DebateProtocol.META_SENTINEL, meta, "Issue deferred.");

                Long inReplyTo = raiseMessageIds.get(te.issueId());
                messageService.dispatch(MessageDispatch.builder()
                        .channelId(channelId)
                        .sender(revSender)
                        .type(MessageType.DECLINE)
                        .content(encoded)
                        .correlationId(te.issueId())
                        .inReplyTo(inReplyTo)
                        .actorType(ActorType.AGENT)
                        .build());
                entryCount++;
            }
        }

        // 6. Evidence MEMO entries
        for (var te : parseResult.trackerStatuses()) {
            if (te.evidence() != null) {
                int evidenceRound = findEvidenceRound(te.issueId(), parseResult);
                entryCount += dispatchMemo(channelId, revSender, evidenceRound,
                        te.issueId() + ": spec commit " + te.evidence());
            }
        }

        // Batch push to WebSocket
        var messages = messageService.pollAfter(channelId, 0L, Integer.MAX_VALUE);
        var entries = messages.stream()
                .map(DebateStreamEntry::from)
                .filter(Objects::nonNull)
                .toList();
        eventBus.pushDebateEntries(channelId, entries);

        Map<String, String> statusDist = new LinkedHashMap<>();
        for (var te : parseResult.trackerStatuses()) {
            statusDist.merge(te.status(), "1",
                    (a, b) -> String.valueOf(Integer.parseInt(a) + 1));
        }

        return new ReplayResult(entryCount, statusDist);
    }

    // ── Helpers ──────────────────────────────────────────────────────────────

    private String registerSender(DebateSession session, AgentType role) {
        return session.registerIfAbsent(role, () -> {
            String id = DebateSession.instanceId(role, session.debateSessionId());
            instanceService.register(id,
                    "DraftHouse replay " + role.name().toLowerCase() + " " + session.debateSessionId(),
                    List.of("document-debate-" + role.name().toLowerCase()));
            return id;
        });
    }

    private int dispatchMemo(UUID channelId, String sender, int round, String content) {
        var meta = buildMeta("MEMO", "REV", round, null, null, null);
        String encoded = ChannelMessageMeta.encode(DebateProtocol.META_SENTINEL, meta, content);
        messageService.dispatch(MessageDispatch.builder()
                .channelId(channelId)
                .sender(sender)
                .type(MessageType.STATUS)
                .content(encoded)
                .actorType(ActorType.AGENT)
                .build());
        return 1;
    }

    private static Map<String, String> buildMeta(String entryType, String role, int round,
                                                  String priority, String scope, String location) {
        Map<String, String> meta = new LinkedHashMap<>();
        meta.put(ConversationProtocol.ENTRY_TYPE, entryType);
        meta.put(ConversationProtocol.ROLE, role);
        meta.put(ConversationProtocol.ROUND, String.valueOf(round));
        if (priority != null) meta.put(ConversationProtocol.PRIORITY, priority);
        if (scope != null) meta.put(ConversationProtocol.SCOPE, scope);
        if (location != null) meta.put(ConversationProtocol.LOCATION, location);
        return meta;
    }

    private static String extractLocation(String body) {
        var m = java.util.regex.Pattern.compile("§(\\d+(?:\\.\\d+)*)|[Ss]ection\\s+(\\d+(?:\\.\\d+)*)")
                .matcher(body);
        if (m.find()) {
            String ref = m.group(1) != null ? m.group(1) : m.group(2);
            return "§" + ref;
        }
        return null;
    }

    private static String findLocationFromResponses(String issueId,
                                                     List<WorkspaceParser.ParsedResponse> responses) {
        for (var r : responses) {
            if (r.issueId().equals(issueId) && r.sectionRef() != null) {
                return "§" + r.sectionRef();
            }
        }
        return null;
    }

    private static int findDeferredRound(String issueId,
                                          WorkspaceParser.WorkspaceParseResult result) {
        for (var round : result.rounds()) {
            for (var conf : round.confirmations()) {
                if (conf.issueId().equals(issueId) && !conf.resolved() && !conf.accepted()) {
                    return round.roundNumber() + 1;
                }
            }
        }
        for (var round : result.rounds()) {
            for (var issue : round.issues()) {
                if (issue.issueId().equals(issueId)) return round.roundNumber();
            }
        }
        return 1;
    }

    private static int findEvidenceRound(String issueId,
                                          WorkspaceParser.WorkspaceParseResult result) {
        for (var round : result.rounds()) {
            for (var resp : round.responses()) {
                if (resp.issueId().equals(issueId) && "FIXED".equals(resp.status())) {
                    return round.roundNumber();
                }
            }
        }
        for (var round : result.rounds()) {
            for (var issue : round.issues()) {
                if (issue.issueId().equals(issueId)) return round.roundNumber();
            }
        }
        return 1;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=WorkspaceReplayAdapterTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add server/runtime/src/main/java/io/casehub/drafthouse/debate/WorkspaceReplayAdapter.java \
       server/runtime/src/test/java/io/casehub/drafthouse/debate/WorkspaceReplayAdapterTest.java
git commit -m "feat(#95): workspace replay adapter — channel dispatch orchestrator

Creates Qhorus channel, encodes entries with DHMETA: sentinel, dispatches
via MessageService, batch-pushes to WebSocket. Handles DEFERRED and
evidence MEMO emission from tracker.md terminal statuses.

Refs #95"
```

---

### Task 4: MCP tool — load_workspace on DebateMcpTools

**Files:**
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/DebateMcpTools.java`
- Create: `server/runtime/src/test/java/io/casehub/drafthouse/debate/LoadWorkspaceTest.java`

**Interfaces:**
- Consumes: `WorkspaceParser.parse()`, `WorkspaceReplayAdapter.replay()`, `ChannelService.create()`, `DebateSessionRegistry`, `WebSocketEventBus.broadcast()`
- Produces: `@Tool("load_workspace") String loadWorkspace(String workspacePath)` — returns JSON summary or `"error: ..."` string

- [ ] **Step 1: Write failing test**

Create `LoadWorkspaceTest.java`:

```java
package io.casehub.drafthouse.debate;

import io.casehub.drafthouse.DebateMcpTools;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.nio.file.Path;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class LoadWorkspaceTest {

    @Inject DebateMcpTools tools;

    @Test
    void load_workspace_returns_summary() {
        String path = Path.of("src/test/resources/fixtures/workspace-replay")
                .toAbsolutePath().toString();
        String result = tools.loadWorkspace(path);

        assertFalse(result.startsWith("error:"), "should not be an error: " + result);
        assertTrue(result.contains("debateSessionId"), "should contain session ID");
        assertTrue(result.contains("entryCount"), "should contain entry count");
    }

    @Test
    void load_workspace_idempotent() {
        String path = Path.of("src/test/resources/fixtures/workspace-replay")
                .toAbsolutePath().toString();

        String first = tools.loadWorkspace(path);
        String second = tools.loadWorkspace(path);

        assertFalse(first.startsWith("error:"));
        assertFalse(second.startsWith("error:"));
    }

    @Test
    void load_workspace_invalid_path() {
        String result = tools.loadWorkspace("/nonexistent/workspace");
        assertTrue(result.startsWith("error:"));
    }

    @Test
    void load_workspace_missing_responses_dir() {
        String path = Path.of("src/test/resources/fixtures")
                .toAbsolutePath().toString();
        String result = tools.loadWorkspace(path);
        assertTrue(result.startsWith("error:"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=LoadWorkspaceTest -DfailIfNoTests=false`
Expected: FAIL — `loadWorkspace` method does not exist

- [ ] **Step 3: Add loadWorkspace method to DebateMcpTools**

Add the following method to `DebateMcpTools.java` (use `ide_insert_member` after `exportDebateSummary`):

```java
@Tool(name = "load_workspace",
      description = "Load a completed design-review workspace into DraftHouse as an interactive debate session. "
              + "The workspace directory must contain a responses/ folder with reviewer-N.md and implementor-N.md files.")
public String loadWorkspace(
        @ToolArg(description = "Absolute path to the design-review workspace directory") String workspacePath) {

    java.nio.file.Path wsPath = java.nio.file.Path.of(workspacePath);
    if (!java.nio.file.Files.isDirectory(wsPath)) {
        return "error: workspace directory not found: " + workspacePath;
    }
    if (!java.nio.file.Files.isDirectory(wsPath.resolve("responses"))) {
        return "error: workspace has no responses/ directory: " + workspacePath;
    }

    String workspaceDirName = wsPath.getFileName().toString();
    String channelName = "drafthouse/debate/replay-" + workspaceDirName;

    // Idempotency — check for existing session with same channel name
    for (var existing : registry.all()) {
        if (channelName.equals(existing.channelName())) {
            return "{\"debateSessionId\":\"" + existing.debateSessionId()
                    + "\",\"channel\":\"" + existing.channelName()
                    + "\",\"status\":\"already_loaded\"}";
        }
    }

    Channel channel = null;
    DebateSession session = null;
    try {
        var parseResult = WorkspaceParser.parse(wsPath);

        channel = channelService.create(io.casehub.qhorus.api.channel.ChannelCreateRequest.builder(channelName)
                .description("DraftHouse replay session")
                .semantic(io.casehub.qhorus.api.channel.ChannelSemantic.APPEND).build());

        String debateSessionId = channel.id().toString();
        session = new DebateSession(channel.id(), debateSessionId, channel.name(), null);

        if (parseResult.specPath() != null) {
            session.addDocument(parseResult.specPath(), "spec");
        }

        registry.put(session);

        var adapter = new WorkspaceReplayAdapter(messageService, instanceService, channelGateway, eventBus);
        var result = adapter.replay(session, parseResult);

        eventBus.broadcast("session-created", new DebateEventResource.SessionInfo(
                session.debateSessionId(), session.channelName(),
                parseResult.specPath(), null));

        return "{\"debateSessionId\":\"" + debateSessionId
                + "\",\"channel\":\"" + channel.name()
                + "\",\"entryCount\":" + result.entryCount()
                + ",\"issues\":" + parseResult.trackerStatuses().size()
                + ",\"rounds\":" + parseResult.rounds().size()
                + ",\"status\":\"loaded\"}";

    } catch (Exception e) {
        LOG.log(Level.WARNING, "load_workspace failed: " + e.getMessage(), e);
        if (channel != null) {
            if (session != null) {
                session.participants().values().forEach(id -> {
                    try { instanceService.deregister(id); } catch (Exception ce) { LOG.warning("cleanup: " + ce.getMessage()); }
                });
                try { registry.remove(channel.id()); } catch (Exception ce) { LOG.warning("cleanup: " + ce.getMessage()); }
            }
            try { channelService.delete(channel.id(), true); } catch (Exception ce) { LOG.warning("cleanup: " + ce.getMessage()); }
        }
        return "error: " + e.getMessage();
    }
}
```

Note: This requires importing `io.casehub.qhorus.api.channel.Channel` at the top of the file (it's already used by `startDebate`). Also add `import java.nio.file.*;` if not present.

- [ ] **Step 4: Add `all()` method to DebateSessionRegistry if missing**

Check the `DebateSessionRegistry` interface. If it doesn't have an `all()` method for iterating sessions, add one. The `loadWorkspace` idempotency check needs to search by channel name.

If the registry doesn't support iteration, use `channelService.findByName(channelName)` instead for the idempotency check — if the channel already exists, find the session by channel ID.

- [ ] **Step 5: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=LoadWorkspaceTest`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add server/runtime/src/main/java/io/casehub/drafthouse/DebateMcpTools.java \
       server/runtime/src/test/java/io/casehub/drafthouse/debate/LoadWorkspaceTest.java
git commit -m "feat(#95): load_workspace MCP tool — replay design-review workspaces

Thin entry point on DebateMcpTools. Validates path, creates channel with
deterministic name, parses workspace, dispatches entries, broadcasts
session-created. Idempotent — returns existing session if already loaded.

Refs #95"
```

---

### Task 5: E2E test — workspace replay rendering

**Files:**
- Create: `server/runtime/src/test/java/io/casehub/drafthouse/e2e/WorkspaceReplayE2ETest.java`

**Interfaces:**
- Consumes: `DebateMcpTools.loadWorkspace()`, `DebateE2EFixtures`, `PlaywrightFixtures`
- Produces: End-to-end verification of the full replay pipeline

- [ ] **Step 1: Write E2E test**

Create `WorkspaceReplayE2ETest.java`:

```java
package io.casehub.drafthouse.e2e;

import com.microsoft.playwright.BrowserContext;
import com.microsoft.playwright.Locator;
import com.microsoft.playwright.Page;
import io.casehub.drafthouse.DebateMcpTools;
import io.quarkiverse.playwright.InjectPlaywright;
import io.quarkiverse.playwright.WithPlaywright;
import io.quarkus.test.common.http.TestHTTPResource;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.net.URL;
import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;
import java.nio.file.Path;

import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
@WithPlaywright
class WorkspaceReplayE2ETest {

    @InjectPlaywright
    BrowserContext context;

    @TestHTTPResource("/")
    URL index;

    @Inject DebateMcpTools tools;

    private Page page;
    private String sessionId;

    @BeforeEach
    void setUp() {
        page = context.newPage();
    }

    @AfterEach
    void tearDown() {
        if (sessionId != null) {
            tools.endDebate(sessionId, false);
            sessionId = null;
        }
        if (page != null) page.close();
    }

    @Test
    void replay_shows_issues_in_tracker() {
        String wsPath = Path.of("src/test/resources/fixtures/workspace-replay")
                .toAbsolutePath().toString();
        String result = tools.loadWorkspace(wsPath);
        assertFalse(result.startsWith("error:"), result);

        sessionId = DebateE2EFixtures.extractSessionId(result);
        assertFalse(sessionId.isBlank());

        String a = URLEncoder.encode(PlaywrightFixtures.fixturePath("diff-a.md"), StandardCharsets.UTF_8);
        String b = URLEncoder.encode(PlaywrightFixtures.fixturePath("diff-b.md"), StandardCharsets.UTF_8);
        page.navigate(index + "?a=" + a + "&b=" + b + "&debate=" + sessionId);
        PlaywrightFixtures.waitForRender(page);

        DebateE2EFixtures.waitForTrackerPoints(page, 4);

        Locator points = page.locator("drafthouse-review-tracker .point-item");
        assertTrue(points.count() >= 4, "should show at least 4 review points");
    }

    @Test
    void replay_shows_verified_status_icon() {
        String wsPath = Path.of("src/test/resources/fixtures/workspace-replay")
                .toAbsolutePath().toString();
        String result = tools.loadWorkspace(wsPath);
        sessionId = DebateE2EFixtures.extractSessionId(result);

        String a = URLEncoder.encode(PlaywrightFixtures.fixturePath("diff-a.md"), StandardCharsets.UTF_8);
        String b = URLEncoder.encode(PlaywrightFixtures.fixturePath("diff-b.md"), StandardCharsets.UTF_8);
        page.navigate(index + "?a=" + a + "&b=" + b + "&debate=" + sessionId);
        PlaywrightFixtures.waitForRender(page);

        DebateE2EFixtures.waitForTrackerPoints(page, 4);

        Locator verifiedIcons = page.locator("drafthouse-review-tracker .status-icon:text('✓✓')");
        assertTrue(verifiedIcons.count() >= 1, "should show at least one VERIFIED icon");
    }

    @Test
    void replay_shows_debate_entries() {
        String wsPath = Path.of("src/test/resources/fixtures/workspace-replay")
                .toAbsolutePath().toString();
        String result = tools.loadWorkspace(wsPath);
        sessionId = DebateE2EFixtures.extractSessionId(result);

        String a = URLEncoder.encode(PlaywrightFixtures.fixturePath("diff-a.md"), StandardCharsets.UTF_8);
        String b = URLEncoder.encode(PlaywrightFixtures.fixturePath("diff-b.md"), StandardCharsets.UTF_8);
        page.navigate(index + "?a=" + a + "&b=" + b + "&debate=" + sessionId);
        PlaywrightFixtures.waitForRender(page);

        DebateE2EFixtures.waitForDebateEntries(page, 1);

        Locator entries = page.locator("drafthouse-debate .entry");
        assertTrue(entries.count() >= 4, "should show debate conversation entries");
    }
}
```

- [ ] **Step 2: Run E2E test**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=WorkspaceReplayE2ETest`
Expected: PASS — all three E2E tests green

If tests fail, debug by inspecting:
- Browser console errors (Playwright console messages)
- WebSocket connection establishment
- Session creation response from loadWorkspace

- [ ] **Step 3: Commit**

```bash
git add server/runtime/src/test/java/io/casehub/drafthouse/e2e/WorkspaceReplayE2ETest.java
git commit -m "test(#95): E2E tests for workspace replay — tracker, status icons, debate entries

Refs #95"
```

---

### Task 6: Run full test suite and fix any regressions

**Files:**
- Potentially any file modified in Tasks 1-5

**Interfaces:**
- Consumes: all changes from Tasks 1-5
- Produces: green test suite confirming no regressions

- [ ] **Step 1: Run the full test suite**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime`
Expected: All tests pass including existing E2E tests

- [ ] **Step 2: Fix any regressions**

If any existing tests fail due to the VERIFIED/DEFERRED additions or the new `Map.ofEntries()` calls, fix them. Common issues:
- Existing tests that assert exact status emoji map size
- Tests that assert the EntryType enum ordinal values
- Tests that assume only AGREED and DECLINED are resolved statuses

- [ ] **Step 3: Commit fixes if any**

```bash
git add -u
git commit -m "fix(#95): resolve test regressions from VERIFIED/DEFERRED additions

Refs #95"
```
