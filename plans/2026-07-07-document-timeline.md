# Document Timeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #98 — Document timeline — version navigation across review rounds
**Issue group:** #98

**Goal:** Add a document timeline showing how a document evolved across review
rounds, with clickable navigation to diff between any two snapshots.

**Architecture:** `ROUND_SNAPSHOT` entries flow through the existing channel →
projection → WebSocket → pages-event pipeline. The replay adapter emits snapshot
entries at round boundaries. A thin timeline strip above the diff panel consumes
these events client-side. A new REST endpoint serves pre-loaded document content
at each round. The projection intercepts `ROUND_SNAPSHOT` to prevent spurious
warnings.

**Tech Stack:** Java 21 (records, sealed interfaces), Quarkus 3.34.3,
casehub-qhorus 0.2-SNAPSHOT, casehub-blocks 0.2-SNAPSHOT, TypeScript (pages
workbench), vanilla JS Web Components (Shadow DOM, adoptedStyleSheets).

## Global Constraints

- `ChannelProjection.apply()` must never throw (PP-20260610-a47ef5)
- Sentinel encoding uses `DebateProtocol.META_SENTINEL` constant — never hardcoded
- `ROUND_SNAPSHOT` is a domain entry type added to `EntryType` enum (not infrastructure provenance)
- Module placement: new API types in `server/api/`, projection + adapter changes in `server/runtime/`
- Playwright E2E: `@WithPlaywright`, `@BeforeEach`/`@AfterEach` page lifecycle, wait on `[data-diff-chunk]`
- MCP tools return error strings — never throw
- `ProjectionResult.isEmpty()` is cursor-based — check domain state directly

---

### Task 1: API Model — SnapshotSource, DocumentSnapshot, DocumentTimeline

**Files:**
- Create: `server/api/src/main/java/io/casehub/drafthouse/debate/SnapshotSource.java`
- Create: `server/api/src/main/java/io/casehub/drafthouse/debate/DocumentSnapshot.java`
- Create: `server/api/src/main/java/io/casehub/drafthouse/debate/DocumentTimeline.java`
- Modify: `server/api/src/main/java/io/casehub/drafthouse/debate/EntryType.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/debate/DocumentTimelineModelTest.java`

**Interfaces:**
- Consumes: nothing (foundation types)
- Produces: `SnapshotSource` sealed interface with `GitCommit(String commitHash, Instant timestamp, int roundNumber)`, `DocumentSnapshot(String documentPath, String label, SnapshotSource source)`, `DocumentTimeline(String documentId, List<DocumentSnapshot> snapshots)`, `EntryType.ROUND_SNAPSHOT`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.drafthouse.debate;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class DocumentTimelineModelTest {

    @Test
    void snapshotSource_gitCommit_carries_round_number() {
        var source = new SnapshotSource.GitCommit("abc123", Instant.parse("2026-07-07T10:00:00Z"), 1);
        assertEquals("abc123", source.commitHash());
        assertEquals(1, source.roundNumber());
    }

    @Test
    void documentSnapshot_uses_label_for_display() {
        var source = new SnapshotSource.GitCommit("abc123", Instant.parse("2026-07-07T10:00:00Z"), 0);
        var snap = new DocumentSnapshot("docs/spec.md", "Round 0 (original)", source);
        assertEquals("Round 0 (original)", snap.label());
        assertNull(snap.documentPath()); // won't compile yet
    }

    @Test
    void documentTimeline_orders_snapshots() {
        var s0 = new DocumentSnapshot("spec.md", "Round 0", new SnapshotSource.GitCommit("aaa", Instant.EPOCH, 0));
        var s1 = new DocumentSnapshot("spec.md", "Round 1", new SnapshotSource.GitCommit("bbb", Instant.EPOCH.plusSeconds(60), 1));
        var timeline = new DocumentTimeline("spec.md", List.of(s0, s1));
        assertEquals(2, timeline.snapshots().size());
        assertEquals("Round 0", timeline.snapshots().get(0).label());
    }

    @Test
    void entryType_round_snapshot_exists() {
        assertNotNull(EntryType.valueOf("ROUND_SNAPSHOT"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DocumentTimelineModelTest -DfailIfNoTests=false`
Expected: FAIL — classes do not exist

- [ ] **Step 3: Create SnapshotSource sealed interface**

Create `server/api/src/main/java/io/casehub/drafthouse/debate/SnapshotSource.java`:

```java
package io.casehub.drafthouse.debate;

import java.time.Instant;

public sealed interface SnapshotSource {
    record GitCommit(String commitHash, Instant timestamp, int roundNumber) implements SnapshotSource {}
}
```

- [ ] **Step 4: Create DocumentSnapshot record**

Create `server/api/src/main/java/io/casehub/drafthouse/debate/DocumentSnapshot.java`:

```java
package io.casehub.drafthouse.debate;

public record DocumentSnapshot(
    String documentPath,
    String label,
    SnapshotSource source
) {}
```

- [ ] **Step 5: Create DocumentTimeline record**

Create `server/api/src/main/java/io/casehub/drafthouse/debate/DocumentTimeline.java`:

```java
package io.casehub.drafthouse.debate;

import java.util.List;

public record DocumentTimeline(
    String documentId,
    List<DocumentSnapshot> snapshots
) {}
```

- [ ] **Step 6: Add ROUND_SNAPSHOT to EntryType**

In `server/api/src/main/java/io/casehub/drafthouse/debate/EntryType.java`, add `ROUND_SNAPSHOT` after `RESTART_CONTEXT`:

```java
RESTART_CONTEXT,
ROUND_SNAPSHOT
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DocumentTimelineModelTest`
Expected: PASS (4 tests)

- [ ] **Step 8: Commit**

```
feat(#98): add DocumentSnapshot, SnapshotSource, DocumentTimeline API model

Sealed interface SnapshotSource with GitCommit variant. ROUND_SNAPSHOT
added to EntryType. Foundation types for document timeline feature.

Refs #98
```

---

### Task 2: DebateStreamEntry — add commitHash and documentPath fields

**Files:**
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/debate/DebateStreamEntry.java`
- Modify: `server/runtime/src/test/java/io/casehub/drafthouse/debate/DebateStreamEntryTest.java`

**Interfaces:**
- Consumes: `EntryType.ROUND_SNAPSHOT` from Task 1
- Produces: `DebateStreamEntry` with two new nullable fields `commitHash` (String) and `documentPath` (String), null-role guard updated for ROUND_SNAPSHOT

- [ ] **Step 1: Write the failing tests**

Add to `DebateStreamEntryTest.java`:

```java
@Test
void roundSnapshot_entry_parses_with_null_role_and_commit_fields() {
    String content = "DHMETA:entryType=ROUND_SNAPSHOT|round=2|commitHash=abc123|documentPath=docs/spec.md\n\nRound 2 snapshot";
    Message msg = buildMessage(content, null, null);
    DebateStreamEntry entry = DebateStreamEntry.from(msg);

    assertNotNull(entry);
    assertEquals(EntryType.ROUND_SNAPSHOT, entry.entryType());
    assertNull(entry.agentRole());
    assertEquals(2, entry.round());
    assertEquals("abc123", entry.commitHash());
    assertEquals("docs/spec.md", entry.documentPath());
}

@Test
void non_snapshot_entry_has_null_commit_fields() {
    String content = "DHMETA:entryType=RAISE|role=REV|round=1\n\nSome issue";
    Message msg = buildMessage(content, "R1-01", null);
    DebateStreamEntry entry = DebateStreamEntry.from(msg);

    assertNotNull(entry);
    assertEquals(EntryType.RAISE, entry.entryType());
    assertNull(entry.commitHash());
    assertNull(entry.documentPath());
}
```

Note: adapt `buildMessage` helper to match the existing test helper pattern — check the existing test file for the exact signature.

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DebateStreamEntryTest`
Expected: FAIL — `commitHash()` and `documentPath()` do not exist on the record

- [ ] **Step 3: Add fields to DebateStreamEntry record**

Modify the record declaration to add two new fields:

```java
public record DebateStreamEntry(
        EntryType entryType,
        AgentType agentRole,
        int round,
        String content,
        String pointId,
        String subTaskId,
        Priority priority,
        String scope,
        String location,
        String sender,
        Instant timestamp,
        String commitHash,
        String documentPath) {
```

- [ ] **Step 4: Update from(Message) factory method**

Update the null-role guard:

```java
} else if (entryType != EntryType.RESTART_CONTEXT
        && entryType != EntryType.ROUND_SNAPSHOT) {
    return null;
}
```

Extract commitHash and documentPath for ROUND_SNAPSHOT:

```java
String commitHash = null;
String documentPath = null;
if (entryType == EntryType.ROUND_SNAPSHOT) {
    commitHash = meta.get("commitHash");
    documentPath = meta.get("documentPath");
}
```

Update the constructor call to include the two new fields:

```java
return new DebateStreamEntry(
        entryType, agentRole, round, body,
        pointId, subTaskId,
        priority, scope,
        location != null && !location.isBlank() ? location : null,
        msg.sender(),
        msg.createdAt() != null ? msg.createdAt() : Instant.now(),
        commitHash, documentPath);
```

- [ ] **Step 5: Update from(OutboundMessage) factory method**

Apply the same three changes: null-role guard, commitHash/documentPath extraction, constructor call.

- [ ] **Step 6: Fix all existing callers**

Grep for `new DebateStreamEntry(` — any call site constructing the record directly needs the two new null args appended. These are likely only in tests.

- [ ] **Step 7: Run full test suite to verify**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```
feat(#98): add commitHash and documentPath to DebateStreamEntry

Nullable fields populated only for ROUND_SNAPSHOT entries. Null-role
guard updated to allow ROUND_SNAPSHOT alongside RESTART_CONTEXT.

Refs #98
```

---

### Task 3: DebateChannelProjection — intercept ROUND_SNAPSHOT

**Files:**
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/debate/DebateChannelProjection.java`
- Modify: `server/runtime/src/test/java/io/casehub/drafthouse/debate/DebateChannelProjectionTest.java`

**Interfaces:**
- Consumes: `EntryType.ROUND_SNAPSHOT`, `DebateProtocol.META_SENTINEL`, `ChannelMessageMeta.parseMeta()`
- Produces: `apply()` returns state unchanged for ROUND_SNAPSHOT (no warning, no state mutation)

- [ ] **Step 1: Write the failing test**

Add to `DebateChannelProjectionTest.java`:

```java
@Test
void roundSnapshot_entry_returns_state_unchanged() {
    var projection = new DebateChannelProjection();
    ConversationState initial = projection.identity();

    String content = ChannelMessageMeta.encode(
            DebateProtocol.META_SENTINEL,
            Map.of("entryType", "ROUND_SNAPSHOT", "round", "2",
                   "commitHash", "abc123", "documentPath", "spec.md"),
            "Round 2 snapshot — 3 raised, 2 fixed");

    MessageView msg = buildMessageView(content, null);
    ConversationState result = projection.apply(initial, msg);

    assertSame(initial, result);
}
```

Note: adapt `buildMessageView` to match the existing test helper.

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DebateChannelProjectionTest#roundSnapshot_entry_returns_state_unchanged`
Expected: FAIL — ROUND_SNAPSHOT falls through to base class dispatch, which logs warnings

- [ ] **Step 3: Override apply() in DebateChannelProjection**

Add the override with try-catch per PP-20260610-a47ef5:

```java
@Override
public ConversationState apply(ConversationState state, MessageView message) {
    try {
        Map<String, String> meta = ChannelMessageMeta.parseMeta(sentinel(), message.content());
        if ("ROUND_SNAPSHOT".equals(meta.get(ConversationProtocol.ENTRY_TYPE))) {
            return state;
        }
    } catch (Exception e) {
        LOG.log(Level.WARNING, "ROUND_SNAPSHOT check failed — delegating to base", e);
    }
    return super.apply(state, message);
}
```

Add the logger field:

```java
private static final java.util.logging.Logger LOG =
    java.util.logging.Logger.getLogger(DebateChannelProjection.class.getName());
```

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DebateChannelProjectionTest`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```
feat(#98): intercept ROUND_SNAPSHOT in DebateChannelProjection.apply()

Returns state unchanged for timeline markers. Try-catch preserves
PP-20260610-a47ef5 (apply must not throw). Double-parse deferred
to blocks#39 (skipEntryTypes hook).

Refs #98
```

---

### Task 4: WorkspaceParser — extract commitHash from tracker entries

**Files:**
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/debate/WorkspaceParser.java`
- Modify: `server/runtime/src/test/java/io/casehub/drafthouse/debate/WorkspaceParserTest.java`

**Interfaces:**
- Consumes: existing `TRACKER_EVIDENCE_RE` pattern
- Produces: `ParsedTrackerEntry` gains `commitHash` field (String, nullable). `WorkspaceParseResult` gains `projectRepoPath` field (String, nullable — from `.source-dirs` file).

- [ ] **Step 1: Write the failing tests**

Add to `WorkspaceParserTest.java`:

```java
@Test
void trackerEntry_extracts_commitHash_from_evidence() {
    // Write a tracker.md with "**Spec commit:** → abc123" format
    Path workspace = createTempWorkspace();
    Files.writeString(workspace.resolve("tracker.md"),
        "# Tracker\n\n### R1-01: Some issue\n- **Status:** VERIFIED\n- **Spec commit:**  → abc123\n");

    var result = WorkspaceParser.parse(workspace);
    assertEquals(1, result.trackerStatuses().size());
    assertEquals("abc123", result.trackerStatuses().get(0).commitHash());
}

@Test
void trackerEntry_null_commitHash_when_no_evidence() {
    Path workspace = createTempWorkspace();
    Files.writeString(workspace.resolve("tracker.md"),
        "# Tracker\n\n### R1-01: Some issue\n- **Status:** OPEN\n");

    var result = WorkspaceParser.parse(workspace);
    assertEquals(1, result.trackerStatuses().size());
    assertNull(result.trackerStatuses().get(0).commitHash());
}

@Test
void parseResult_reads_projectRepoPath_from_sourceDirs() {
    Path workspace = createTempWorkspace();
    Files.writeString(workspace.resolve(".source-dirs"),
        "/Users/dev/project\n/Users/dev/workspace\n");

    var result = WorkspaceParser.parse(workspace);
    assertEquals("/Users/dev/project", result.projectRepoPath());
}

@Test
void parseResult_null_projectRepoPath_when_no_sourceDirs() {
    Path workspace = createTempWorkspace();
    var result = WorkspaceParser.parse(workspace);
    assertNull(result.projectRepoPath());
}
```

Note: adapt `createTempWorkspace` to match existing test helper or use `@TempDir`.

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=WorkspaceParserTest`
Expected: FAIL — `commitHash()` does not exist on `ParsedTrackerEntry`, `projectRepoPath()` does not exist on `WorkspaceParseResult`

- [ ] **Step 3: Add commitHash to ParsedTrackerEntry**

Change the record:

```java
public record ParsedTrackerEntry(
        String issueId,
        String title,
        String status,
        String evidence,
        String commitHash) {}
```

- [ ] **Step 4: Extract commitHash in parseTracker()**

After extracting `evidence`, derive `commitHash`:

```java
String commitHash = null;
if (evidence != null) {
    String cleaned = evidence.replaceFirst("^.*→\\s*", "").trim();
    if (!cleaned.isEmpty()) commitHash = cleaned;
}

entries.add(new ParsedTrackerEntry(
        headingData.get(i)[0], headingData.get(i)[1], status, evidence, commitHash));
```

- [ ] **Step 5: Add projectRepoPath to WorkspaceParseResult**

Change the record:

```java
public record WorkspaceParseResult(
        String specPath,
        String mode,
        String contextNote,
        List<ParsedRound> rounds,
        List<ParsedTrackerEntry> trackerStatuses,
        String projectRepoPath) {}
```

- [ ] **Step 6: Read .source-dirs in parse()**

```java
String projectRepoPath = null;
String sourceDirs = readFileOrNull(workspaceDir.resolve(".source-dirs"));
if (sourceDirs != null) {
    String firstLine = sourceDirs.lines().findFirst().orElse(null);
    if (firstLine != null) projectRepoPath = firstLine.trim();
}

return new WorkspaceParseResult(
        specPath != null ? specPath.trim() : null,
        mode != null ? mode.trim() : null,
        contextNote,
        rounds,
        trackerStatuses,
        projectRepoPath);
```

- [ ] **Step 7: Fix all callers of ParsedTrackerEntry and WorkspaceParseResult**

Search for `new ParsedTrackerEntry(` and `new WorkspaceParseResult(` — update constructor calls in tests.

- [ ] **Step 8: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```
feat(#98): extract commitHash and projectRepoPath in WorkspaceParser

ParsedTrackerEntry gains commitHash parsed from "Spec commit: → hash".
WorkspaceParseResult gains projectRepoPath from .source-dirs file.

Refs #98
```

---

### Task 5: WorkspaceReplayAdapter — emit ROUND_SNAPSHOT entries and pre-load content

**Files:**
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/debate/WorkspaceReplayAdapter.java`
- Modify: `server/runtime/src/test/java/io/casehub/drafthouse/debate/WorkspaceReplayAdapterTest.java`

**Interfaces:**
- Consumes: `ParsedTrackerEntry.commitHash()`, `WorkspaceParseResult.projectRepoPath()`, `ChannelMessageMeta.encode()`, `DocumentTimeline`, `DocumentSnapshot`, `SnapshotSource.GitCommit`
- Produces: `ReplayResult(int entryCount, Map<String, String> statusDistribution, DocumentTimeline timeline, Map<Integer, String> snapshotContent)`. ROUND_SNAPSHOT entries dispatched at round boundaries.

- [ ] **Step 1: Write the failing tests**

Add to `WorkspaceReplayAdapterTest.java`:

```java
@Test
void replayResult_contains_timeline_and_snapshotContent() {
    // Set up a workspace parse result with tracker entries containing commitHash
    // and a projectRepoPath pointing to a git repo fixture
    // ... (adapt to existing test pattern — mock MessageService, InstanceService, etc.)

    ReplayResult result = adapter.replay(session, parseResult);

    assertNotNull(result.timeline());
    assertFalse(result.timeline().snapshots().isEmpty());
    assertFalse(result.snapshotContent().isEmpty());
}

@Test
void replayResult_emits_round_snapshot_entries() {
    // Verify that ROUND_SNAPSHOT entries appear in the dispatched messages
    // by checking the messageService.dispatch() calls
    // ... (use existing mock/capture pattern)
}
```

Note: this test depends on git fixture setup. If git access is impractical in unit tests, test with null projectRepoPath (no snapshots emitted) and test the snapshot emission logic separately with a real workspace in the integration test (`LoadWorkspaceTest`).

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=WorkspaceReplayAdapterTest`
Expected: FAIL — `ReplayResult` doesn't have `timeline()` or `snapshotContent()` fields

- [ ] **Step 3: Update ReplayResult record**

```java
public record ReplayResult(
    int entryCount,
    Map<String, String> statusDistribution,
    DocumentTimeline timeline,
    Map<Integer, String> snapshotContent) {}
```

- [ ] **Step 4: Add round-to-commit mapping logic**

Add a helper method:

```java
private static Map<Integer, String> buildRoundCommitMap(
        WorkspaceParser.WorkspaceParseResult parseResult) {
    Map<Integer, String> roundToCommit = new LinkedHashMap<>();
    for (var te : parseResult.trackerStatuses()) {
        if (te.commitHash() == null) continue;
        int fixRound = findEvidenceRound(te.issueId(), parseResult);
        roundToCommit.putIfAbsent(fixRound, te.commitHash());
    }
    return roundToCommit;
}
```

- [ ] **Step 5: Add git content pre-loading**

Add a helper that runs `git show <hash>:<path>` against the project repo:

```java
private static String gitShow(String repoPath, String commitHash, String filePath) {
    try {
        String relativePath = filePath;
        if (relativePath.startsWith(repoPath)) {
            relativePath = relativePath.substring(repoPath.length());
            if (relativePath.startsWith("/")) relativePath = relativePath.substring(1);
        }
        ProcessBuilder pb = new ProcessBuilder("git", "show", commitHash + ":" + relativePath);
        pb.directory(new java.io.File(repoPath));
        pb.redirectErrorStream(true);
        Process p = pb.start();
        String output = new String(p.getInputStream().readAllBytes());
        int exit = p.waitFor();
        return exit == 0 ? output : null;
    } catch (Exception e) {
        LOG.warning("git show failed for " + commitHash + ":" + filePath + " — " + e.getMessage());
        return null;
    }
}

private static String findInitialCommit(String repoPath, String filePath) {
    try {
        String relativePath = filePath;
        if (relativePath.startsWith(repoPath)) {
            relativePath = relativePath.substring(repoPath.length());
            if (relativePath.startsWith("/")) relativePath = relativePath.substring(1);
        }
        ProcessBuilder pb = new ProcessBuilder("git", "log", "--reverse", "--format=%H", "--", relativePath);
        pb.directory(new java.io.File(repoPath));
        Process p = pb.start();
        String output = new String(p.getInputStream().readAllBytes()).trim();
        int exit = p.waitFor();
        if (exit != 0 || output.isEmpty()) return null;
        return output.lines().findFirst().orElse(null);
    } catch (Exception e) {
        LOG.warning("git log --reverse failed for " + filePath + " — " + e.getMessage());
        return null;
    }
}
```

- [ ] **Step 6: Emit ROUND_SNAPSHOT entries in the replay loop**

After step 6 (evidence MEMOs) and before the batch WebSocket push, build the timeline and dispatch ROUND_SNAPSHOT entries:

```java
// 7. Build timeline and emit ROUND_SNAPSHOT entries
DocumentTimeline timeline = null;
Map<Integer, String> snapshotContent = new LinkedHashMap<>();
String specPath = parseResult.specPath();
String repoPath = parseResult.projectRepoPath();

if (specPath != null && repoPath != null) {
    Map<Integer, String> roundCommits = buildRoundCommitMap(parseResult);
    List<DocumentSnapshot> snapshots = new ArrayList<>();
    int snapshotIndex = 0;

    // Round 0 — original document
    String initialCommit = findInitialCommit(repoPath, specPath);
    if (initialCommit != null) {
        String content = gitShow(repoPath, initialCommit, specPath);
        if (content != null) {
            var source = new SnapshotSource.GitCommit(initialCommit, Instant.EPOCH, 0);
            snapshots.add(new DocumentSnapshot(specPath, "Round 0 (original)", source));
            snapshotContent.put(snapshotIndex, content);
            dispatchRoundSnapshot(channelId, revSender, 0, initialCommit, specPath,
                    "Round 0 snapshot — original document");
            entryCount++;
            snapshotIndex++;
        }
    }

    // Rounds with commits
    for (var entry : roundCommits.entrySet()) {
        int roundNum = entry.getKey();
        String commitHash = entry.getValue();
        String content = gitShow(repoPath, commitHash, specPath);
        if (content != null) {
            long issueCount = countIssuesInRound(roundNum, parseResult);
            long fixCount = countFixesInRound(roundNum, parseResult);
            String label = String.format("Round %d (+%d raised, %d fixed)", roundNum, issueCount, fixCount);
            var source = new SnapshotSource.GitCommit(commitHash, Instant.now(), roundNum);
            snapshots.add(new DocumentSnapshot(specPath, label, source));
            snapshotContent.put(snapshotIndex, content);
            dispatchRoundSnapshot(channelId, revSender, roundNum, commitHash, specPath,
                    String.format("Round %d snapshot — %d issues raised, %d fixed", roundNum, issueCount, fixCount));
            entryCount++;
            snapshotIndex++;
        }
    }

    timeline = new DocumentTimeline(specPath, snapshots);
}
```

- [ ] **Step 7: Add helper methods**

```java
private void dispatchRoundSnapshot(UUID channelId, String sender, int round,
                                    String commitHash, String documentPath, String body) {
    Map<String, String> meta = new LinkedHashMap<>();
    meta.put(ConversationProtocol.ENTRY_TYPE, "ROUND_SNAPSHOT");
    meta.put(ConversationProtocol.ROUND, String.valueOf(round));
    meta.put("commitHash", commitHash);
    meta.put("documentPath", documentPath);
    String encoded = ChannelMessageMeta.encode(DebateProtocol.META_SENTINEL, meta, body);
    messageService.dispatch(MessageDispatch.builder()
            .channelId(channelId)
            .sender(sender)
            .type(MessageType.STATUS)
            .content(encoded)
            .actorType(ActorType.AGENT)
            .build());
}

private static long countIssuesInRound(int round, WorkspaceParser.WorkspaceParseResult result) {
    return result.rounds().stream()
            .filter(r -> r.roundNumber() == round)
            .mapToLong(r -> r.issues().size())
            .sum();
}

private static long countFixesInRound(int round, WorkspaceParser.WorkspaceParseResult result) {
    return result.rounds().stream()
            .filter(r -> r.roundNumber() == round)
            .mapToLong(r -> r.responses().stream().filter(resp -> "FIXED".equals(resp.status())).count())
            .sum();
}
```

- [ ] **Step 8: Update the return statement**

```java
return new ReplayResult(entryCount, statusDist, timeline, snapshotContent);
```

- [ ] **Step 9: Fix all callers of ReplayResult**

Search for `ReplayResult` usage — `DebateMcpTools` and `LoadWorkspaceTest` likely reference `.entryCount()` and `.statusDistribution()`. The callers that construct `ReplayResult` in tests need the two new fields.

- [ ] **Step 10: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime`
Expected: ALL PASS

- [ ] **Step 11: Commit**

```
feat(#98): emit ROUND_SNAPSHOT entries and pre-load content in replay adapter

Builds DocumentTimeline with GitCommit snapshots. Pre-loads content via
git show. Dispatches ROUND_SNAPSHOT channel messages at round boundaries.

Refs #98
```

---

### Task 6: Snapshot REST endpoint + session content storage

**Files:**
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/DebateSession.java`
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/DebateEventResource.java` (or modify if endpoint class already exists)
- Test: add to existing integration test or create `server/runtime/src/test/java/io/casehub/drafthouse/SnapshotEndpointTest.java`

**Interfaces:**
- Consumes: `ReplayResult.snapshotContent()`, `ReplayResult.timeline()`, `DebateSession`
- Produces: `GET /api/debate/{id}/snapshot/{index}` — returns plain text content, 404 if not found. `DebateSession.snapshotContent()` accessor.

- [ ] **Step 1: Write the failing test**

```java
@Test
void snapshotEndpoint_returns_content_for_valid_index() {
    // Start a debate session, populate snapshot content
    // GET /api/debate/{id}/snapshot/0 → 200, plain text body
}

@Test
void snapshotEndpoint_returns_404_for_invalid_index() {
    // GET /api/debate/{id}/snapshot/99 → 404
}
```

- [ ] **Step 2: Add snapshotContent storage to DebateSession**

```java
private volatile Map<Integer, String> snapshotContent = Map.of();
private volatile DocumentTimeline timeline;

public void setSnapshotContent(Map<Integer, String> content) {
    this.snapshotContent = Map.copyOf(content);
}

public String snapshotContentAt(int index) {
    return snapshotContent.get(index);
}

public void setTimeline(DocumentTimeline timeline) {
    this.timeline = timeline;
}

public DocumentTimeline timeline() {
    return timeline;
}
```

- [ ] **Step 3: Store snapshot data after replay**

In the MCP tool layer where `replay()` is called (likely `DebateMcpTools` `load_workspace`), after getting the `ReplayResult`:

```java
if (result.timeline() != null) {
    session.setTimeline(result.timeline());
    session.setSnapshotContent(result.snapshotContent());
}
```

- [ ] **Step 4: Add the REST endpoint**

Check if `DebateEventResource.java` already exists and add to it, or create a new resource:

```java
@GET
@Path("/debate/{id}/snapshot/{index}")
@Produces(MediaType.TEXT_PLAIN)
public Response snapshot(@PathParam("id") String sessionId, @PathParam("index") int index) {
    DebateSession session = resolveSession(sessionId);
    if (session == null) return Response.status(404).entity("Session not found").build();
    String content = session.snapshotContentAt(index);
    if (content == null) return Response.status(404).entity("Snapshot not found").build();
    return Response.ok(content).build();
}
```

- [ ] **Step 5: Run tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```
feat(#98): add snapshot REST endpoint and session content storage

GET /api/debate/{id}/snapshot/{index} serves pre-loaded document content.
DebateSession stores snapshot content and timeline from ReplayResult.

Refs #98
```

---

### Task 7: Timeline panel (`<drafthouse-timeline>`) + diff panel integration

**Files:**
- Create: `server/runtime/src/main/webui/src/panels/drafthouse-timeline.js`
- Modify: `server/runtime/src/main/webui/src/panels/drafthouse-diff.js`
- Modify: `server/runtime/src/main/webui/src/panels/drafthouse-review-tracker.js`
- Modify: `server/runtime/src/main/webui/src/index.ts`

**Interfaces:**
- Consumes: `debate-entries` pages-event (filters ROUND_SNAPSHOT), `point-selected`/`point-deselected` custom events, `GET /api/debate/{id}/snapshot/{index}`
- Produces: `timeline-comparison-changed` custom event with `{ sessionId, indexA, indexB, labelA, labelB }`, `loadContent(panel, content, label)` on diff panel

- [ ] **Step 1: Create the timeline panel**

Create `server/runtime/src/main/webui/src/panels/drafthouse-timeline.js`:

```javascript
const styles = new CSSStyleSheet();
styles.replaceSync(`
  :host {
    display: block;
    height: 40px;
    background: var(--chrome, #ede7d9);
    border-bottom: 1px solid var(--border, #c8baa0);
    padding: 4px 12px;
    font-family: 'SFMono-Regular', Consolas, monospace;
    font-size: 11px;
  }

  .timeline-track {
    display: flex;
    align-items: center;
    height: 100%;
    gap: 0;
    overflow-x: auto;
    overflow-y: hidden;
  }

  .marker {
    display: flex;
    flex-direction: column;
    align-items: center;
    cursor: pointer;
    padding: 2px 8px;
    border-radius: 4px;
    transition: background 0.15s;
    white-space: nowrap;
    flex-shrink: 0;
  }

  .marker:hover { background: var(--bg, #f4f0e8); }
  .marker.selected { background: var(--accent-bg, #dbe4ee); }
  .marker.trail-fix { font-weight: 700; }
  .marker.trail-raise, .marker.trail-verify {
    opacity: 0.6;
  }
  .marker.trail-raise::after, .marker.trail-verify::after {
    content: '';
    display: block;
    width: 4px;
    height: 4px;
    border-radius: 50%;
    background: var(--accent, #4a6a8a);
    margin-top: 2px;
  }

  .connector {
    flex: 1;
    min-width: 16px;
    max-width: 60px;
    height: 2px;
    background: var(--border, #c8baa0);
  }

  .connector.active {
    background: var(--accent, #4a6a8a);
    height: 3px;
  }

  .marker-dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    background: var(--muted, #8a7a5a);
    border: 2px solid var(--chrome, #ede7d9);
  }
  .marker.selected .marker-dot {
    background: var(--accent, #4a6a8a);
  }

  .marker-label {
    font-size: 10px;
    color: var(--muted, #8a7a5a);
    margin-top: 1px;
  }
  .marker.selected .marker-label {
    color: var(--ink, #2a2218);
    font-weight: 600;
  }

  .hidden { display: none; }
`);

class DraftHouseTimeline extends HTMLElement {
  #shadow;
  #snapshots = [];       // { label, round, commitHash, documentPath }
  #selectedA = null;     // index
  #selectedB = null;     // index
  #sessionId = null;
  #trailHighlight = null; // { raiseRound, fixRound, verifyRound }

  constructor() {
    super();
    this.#shadow = this.attachShadow({ mode: 'open' });
    this.#shadow.adoptedStyleSheets = [styles];
  }

  configure(props) {
    if (props.sessionId) this.#sessionId = props.sessionId;
    this.#initialize();
  }

  #initialize() {
    const listener = (e) => {
      const { topic, payload } = e.detail;
      if (topic === 'debate-entries') {
        this.#handleEntries(Array.isArray(payload) ? payload : [payload]);
      } else if (topic === 'reconnected') {
        this.#snapshots = [];
        this.#selectedA = null;
        this.#selectedB = null;
        this.#render();
      }
    };
    document.addEventListener('pages-event', listener);

    // Listen for point-selected/deselected from review tracker
    document.addEventListener('point-selected', (e) => {
      this.#trailHighlight = {
        raiseRound: e.detail.raiseRound,
        fixRound: e.detail.fixRound,
        verifyRound: e.detail.verifyRound,
      };
      // Jump diff to the fix round
      if (e.detail.fixRound != null) {
        const fixIndex = this.#snapshots.findIndex(
          s => s.round === e.detail.fixRound);
        if (fixIndex >= 0) {
          const prevIndex = Math.max(0, fixIndex - 1);
          this.#selectedA = prevIndex;
          this.#selectedB = fixIndex;
          this.#emitComparison();
        }
      }
      this.#render();
    });

    document.addEventListener('point-deselected', () => {
      this.#trailHighlight = null;
      this.#render();
    });

    this.#render();
  }

  #handleEntries(entries) {
    let added = false;
    for (const entry of entries) {
      if (entry.entryType === 'ROUND_SNAPSHOT') {
        this.#snapshots.push({
          label: entry.content,
          round: entry.round,
          commitHash: entry.commitHash,
          documentPath: entry.documentPath,
        });
        added = true;
      }
    }
    if (added) {
      // Default: select last two for adjacent comparison
      if (this.#snapshots.length >= 2 && this.#selectedA == null) {
        this.#selectedA = this.#snapshots.length - 2;
        this.#selectedB = this.#snapshots.length - 1;
        this.#emitComparison();
      } else if (this.#snapshots.length === 1 && this.#selectedA == null) {
        this.#selectedA = 0;
        this.#selectedB = 0;
      }
      this.#render();
    }
  }

  #handleClick(index, shiftKey) {
    if (shiftKey && this.#selectedA != null) {
      // Shift-click: set second endpoint
      this.#selectedB = index;
    } else if (this.#selectedA === index && this.#selectedB != null) {
      // Clicking already-selected marker clears selection
      this.#selectedA = null;
      this.#selectedB = null;
    } else {
      // Single click: set as one end, pair with adjacent
      this.#selectedA = index;
      this.#selectedB = Math.min(index + 1, this.#snapshots.length - 1);
      if (this.#selectedA === this.#selectedB && index > 0) {
        this.#selectedA = index - 1;
        this.#selectedB = index;
      }
    }
    // Ensure A < B
    if (this.#selectedA != null && this.#selectedB != null && this.#selectedA > this.#selectedB) {
      [this.#selectedA, this.#selectedB] = [this.#selectedB, this.#selectedA];
    }
    this.#emitComparison();
    this.#render();
  }

  #emitComparison() {
    if (this.#selectedA == null || this.#selectedB == null) return;
    this.dispatchEvent(new CustomEvent('timeline-comparison-changed', {
      bubbles: true,
      composed: true,
      detail: {
        sessionId: this.#sessionId,
        indexA: this.#selectedA,
        indexB: this.#selectedB,
        labelA: this.#snapshots[this.#selectedA]?.label || `Snapshot ${this.#selectedA}`,
        labelB: this.#snapshots[this.#selectedB]?.label || `Snapshot ${this.#selectedB}`,
      }
    }));
  }

  #render() {
    if (this.#snapshots.length === 0) {
      this.#shadow.innerHTML = '';
      this.classList.add('hidden');
      return;
    }
    this.classList.remove('hidden');

    const track = document.createElement('div');
    track.className = 'timeline-track';

    this.#snapshots.forEach((snap, i) => {
      if (i > 0) {
        const conn = document.createElement('div');
        conn.className = 'connector';
        if (this.#selectedA != null && this.#selectedB != null
            && i > this.#selectedA && i <= this.#selectedB) {
          conn.classList.add('active');
        }
        track.appendChild(conn);
      }

      const marker = document.createElement('div');
      marker.className = 'marker';
      if (i === this.#selectedA || i === this.#selectedB) marker.classList.add('selected');

      // Trail highlight
      if (this.#trailHighlight) {
        if (snap.round === this.#trailHighlight.fixRound) marker.classList.add('trail-fix');
        if (snap.round === this.#trailHighlight.raiseRound) marker.classList.add('trail-raise');
        if (snap.round === this.#trailHighlight.verifyRound) marker.classList.add('trail-verify');
      }

      const dot = document.createElement('div');
      dot.className = 'marker-dot';
      marker.appendChild(dot);

      const label = document.createElement('div');
      label.className = 'marker-label';
      // Use the label from the entry content (human-readable summary)
      label.textContent = snap.label.split(' — ')[0] || `Round ${snap.round}`;
      marker.appendChild(label);

      marker.addEventListener('click', (e) => this.#handleClick(i, e.shiftKey));
      track.appendChild(marker);
    });

    this.#shadow.innerHTML = '';
    this.#shadow.appendChild(track);
  }
}

customElements.define('drafthouse-timeline', DraftHouseTimeline);
```

- [ ] **Step 2: Add loadContent() to the diff panel**

In `drafthouse-diff.js`, add after the `loadFile` method:

```javascript
loadContent(panel, content, label) {
  this._panels[panel].path = null;
  this._panels[panel].label = label || 'Snapshot';
  this._panels[panel].content = content;
  this._syncPanelMeta(panel);
  this._syncPanelContent(panel);
  if (panel === 'a') this.#pathA = null;
  if (panel === 'b') this.#pathB = null;
  this._updateSwapButton();
  this._updateDiffMap();
}
```

Also update `_syncPanelMeta` to handle the null path case — the path display should show the label when no path is set:

```javascript
_syncPanelMeta(panel) {
  const s = this._panels[panel];
  this._$(`label-${panel}`).value = s.label;
  this._$(`path-${panel}`).textContent = s.path || s.label || 'No file selected';
  this._$(`path-${panel}`).classList.toggle('loaded', !!(s.path || s.content));
}
```

- [ ] **Step 3: Add timeline-comparison-changed listener to diff panel**

In the diff panel's `configure()` or `#initialize()` method, add:

```javascript
document.addEventListener('timeline-comparison-changed', async (e) => {
  const { sessionId, indexA, indexB, labelA, labelB } = e.detail;
  try {
    const [contentA, contentB] = await Promise.all([
      fetch(this._apiUrl(`/api/debate/${sessionId}/snapshot/${indexA}`)).then(r => r.ok ? r.text() : null),
      fetch(this._apiUrl(`/api/debate/${sessionId}/snapshot/${indexB}`)).then(r => r.ok ? r.text() : null),
    ]);
    if (contentA != null) this.loadContent('a', contentA, labelA);
    if (contentB != null) this.loadContent('b', contentB, labelB);
  } catch (err) {
    console.error('Timeline snapshot fetch failed:', err);
  }
});
```

- [ ] **Step 4: Enrich point-selected event in review tracker**

In `drafthouse-review-tracker.js`, update `#derivePoints()` to compute per-phase rounds:

```javascript
// In the per-pointId loop, after building the trail:
const raiseEntry = entries.find(e => e.entryType === 'RAISE');
const fixEntry = entries.find(e => e.entryType === 'QUALIFY' || e.entryType === 'COUNTER');
const verifyEntry = entries.find(e => e.entryType === 'VERIFIED' || e.entryType === 'AGREE');

points.push({
  pointId,
  status,
  summary,
  location: raiseEntry?.location || lastEntry.location,
  round: lastEntry.round,
  raiseRound: raiseEntry?.round ?? null,
  fixRound: fixEntry?.round ?? null,
  verifyRound: verifyEntry?.round ?? null,
  trail,
  isQualifyActive,
});
```

Update the event dispatch to include the new fields:

```javascript
this.dispatchEvent(new CustomEvent(eventName, {
  bubbles: true,
  detail: {
    pointId: point.pointId,
    round: point.round,
    raiseRound: point.raiseRound,
    fixRound: point.fixRound,
    verifyRound: point.verifyRound,
    location: point.location
  }
}));
```

- [ ] **Step 5: Register and wire timeline in index.ts**

In `index.ts`:

```typescript
import "./panels/drafthouse-timeline.js";

registerPanel("document-timeline", "drafthouse-timeline");
```

Update the layout to insert the timeline strip above the diff viewer. Replace:

```typescript
hostPanel("diff-viewer", { pathA, pathB }),
```

With:

```typescript
rows(
  hostPanel("document-timeline", { sessionId: debateParam || "" }),
  hostPanel("diff-viewer", { pathA, pathB }),
),
```

- [ ] **Step 6: Manual verification**

Build and run the app:
```bash
/opt/homebrew/bin/mvn -f server/pom.xml package -DskipTests
java -jar server/runtime/target/drafthouse-server-runner.jar
```

Load a workspace via the MCP `load_workspace` tool and verify:
- Timeline strip appears above the diff panel with round markers
- Clicking markers updates the diff panel
- Shift-click for non-adjacent comparison works
- Clicking an issue in the review tracker highlights the round on the timeline

- [ ] **Step 7: Commit**

```
feat(#98): add drafthouse-timeline panel with diff and tracker integration

Timeline strip renders ROUND_SNAPSHOT entries as clickable markers.
Adjacent comparison default, shift-click for non-adjacent. Diff panel
gains loadContent() for snapshot content. Review tracker enriches
point-selected with raiseRound/fixRound/verifyRound for trail highlight.

Refs #98
```

---

### Task 8: E2E Tests

**Files:**
- Create: `server/runtime/src/test/java/io/casehub/drafthouse/e2e/TimelineE2ETest.java`
- Create: test fixture workspace (git repo with rounds, tracker with commit hashes)

**Interfaces:**
- Consumes: `/api/debate/{id}/snapshot/{index}`, `load_workspace` MCP tool, timeline panel DOM, diff panel DOM
- Produces: E2E verification of the full pipeline

- [ ] **Step 1: Create test fixture workspace**

Create a fixture directory at `server/runtime/src/test/resources/fixtures/timeline-workspace/`:
- `.spec-path` pointing to a spec file
- `.source-dirs` pointing to the fixture project repo
- `tracker.md` with 2-3 issues with `**Spec commit:**` fields
- `responses/reviewer-1.md`, `responses/implementor-1.md`, `responses/reviewer-2.md`
- A small git repo containing the spec file at different commits

This fixture setup is complex — the git repo needs real commits. Consider creating it programmatically in `@BeforeAll` using JGit or `ProcessBuilder("git", ...)`.

- [ ] **Step 2: Write the E2E test class**

```java
@QuarkusTest
@WithPlaywright
class TimelineE2ETest {

    @BeforeEach
    void openPage() { /* standard page lifecycle */ }

    @AfterEach
    void closePage() { page.close(); }

    @Test
    void timeline_renders_after_workspace_replay() {
        // Load workspace via MCP or REST
        // Wait for timeline markers to appear
        // Assert correct number of markers
    }

    @Test
    void clicking_markers_updates_diff_panel() {
        // Load workspace
        // Click two non-adjacent markers
        // Verify diff panel labels show "Round N" format
        // Verify diff panel content changes
    }

    @Test
    void review_tracker_click_highlights_timeline() {
        // Load workspace
        // Click an issue in the review tracker
        // Verify corresponding timeline marker gets trail-fix class
    }
}
```

- [ ] **Step 3: Run E2E tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=TimelineE2ETest`
Expected: ALL PASS

- [ ] **Step 4: Commit**

```
test(#98): E2E tests for document timeline — render, navigation, tracker integration

Fixture workspace with git history exercises the full pipeline:
replay adapter → ROUND_SNAPSHOT → timeline panel → diff panel.

Refs #98
```

---

## Task Dependencies

```
Task 1 (API model)
  ↓
Task 2 (DebateStreamEntry)     Task 4 (WorkspaceParser)
  ↓                              ↓
Task 3 (Projection)            Task 5 (ReplayAdapter) ← depends on Task 2 + Task 4
                                 ↓
                               Task 6 (REST endpoint + session storage)
                                 ↓
                               Task 7 (Timeline panel + diff integration) ← depends on Task 3 + Task 6
                                 ↓
                               Task 8 (E2E tests) ← depends on all above
```

Tasks 2 and 4 can run in parallel after Task 1. Task 3 depends only on Task 1. Task 5 depends on Tasks 2 and 4. Tasks 6-8 are sequential.
