# Live Workspace Watching Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #99 — Live workspace watching adapter — real-time design-review monitoring
**Issue group:** #99

**Goal:** Extend `load_workspace` to auto-detect running reviews and watch for new
files in near-real-time, parsing new rounds as they arrive and pushing progress
metadata to the browser.

**Architecture:** `WorkspaceWatcher` holds a `DirectoryWatcher` handle, tracks parse
state (`lastReplayedRound`, `processedFiles`, `progressLogOffset`), and dispatches
entries through extracted `WorkspaceReplayAdapter` methods shared with one-shot replay.
Progress.log tailing provides agent status/cost/timing metadata on a separate topic
(`workspace-progress`). A `<workspace-status>` topbar element renders progress.

**Tech Stack:** Java 21, Quarkus 3.34.3, `io.methvin:directory-watcher:0.18.0`,
Lit (LitElement), TypeScript, `@casehubio/pages-component`

## Global Constraints

- `io.methvin:directory-watcher:0.18.0` — pinned version, new dependency
- All `MessageDispatch` builders on the watcher thread must include `.tenancyId(capturedTenancyId)` — the watcher thread has no CDI request context
- CDI request context must be activated around each watcher callback via `Arc.container().requestContext()`
- `WorkspaceWatcher` methods exposed for watcher use are package-visible, not public
- Panels follow casehub-pages convention: LitElement, Shadow DOM, `onPagesEvent()`, `_cleanups[]`
- Existing tests must continue to pass after refactoring

---

### Task 1: Foundation — dependency, parser visibility, test fixture

**Files:**
- Modify: `server/runtime/pom.xml` (add directory-watcher dependency)
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/debate/WorkspaceParser.java`
  (change 4 methods from `private` to package-visible)
- Create: `server/runtime/src/test/resources/fixtures/workspace-watching/` (test fixture directory)
- Create: `server/runtime/src/test/resources/fixtures/workspace-watching/responses/reviewer-1.md`
- Create: `server/runtime/src/test/resources/fixtures/workspace-watching/responses/implementor-1.md`
- Create: `server/runtime/src/test/resources/fixtures/workspace-watching/tracker.md`
- Create: `server/runtime/src/test/resources/fixtures/workspace-watching/progress.log`
- Create: `server/runtime/src/test/resources/fixtures/workspace-watching/.spec-path`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/debate/WorkspaceParserVisibilityTest.java`

**Interfaces:**
- Produces: `WorkspaceParser.parseRoundFromMarkdown(Path, int, Set<String>)` → `ParsedRound` (package-visible)
- Produces: `WorkspaceParser.parseRoundFromJsonl(Path, int)` → `ParsedRound` (package-visible)
- Produces: `WorkspaceParser.discoverMaxRound(Path)` → `int` (package-visible)
- Produces: `WorkspaceParser.parseTracker(Path)` → `List<ParsedTrackerEntry>` (already static, now package-visible)
- Produces: Test fixture at `src/test/resources/fixtures/workspace-watching/`

- [ ] **Step 1: Add directory-watcher dependency to pom.xml**

Open `server/runtime/pom.xml`. Add after the pty4j dependency (before test dependencies):

```xml
    <!-- directory-watcher — native macOS FSEvents via JNA, cross-platform recursive watching -->
    <dependency>
      <groupId>io.methvin</groupId>
      <artifactId>directory-watcher</artifactId>
      <version>0.18.0</version>
    </dependency>
```

Use `ide_read_file` to find the exact insertion point (after pty4j, before `<!-- Test dependencies -->`),
then use `ide_insert_member` or the Edit tool on pom.xml.

- [ ] **Step 2: Write test verifying per-round parsing is accessible**

Create `WorkspaceParserVisibilityTest.java`:

```java
package io.casehub.drafthouse.debate;

import org.junit.jupiter.api.Test;
import java.nio.file.Path;
import java.util.HashSet;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.*;

class WorkspaceParserVisibilityTest {

    private final Path fixture = Path.of("src/test/resources/fixtures/workspace-replay");

    @Test
    void discoverMaxRound_finds_highest_round() {
        int max = WorkspaceParser.discoverMaxRound(fixture.resolve("responses"));
        assertTrue(max >= 1, "fixture should have at least 1 round");
    }

    @Test
    void parseRoundFromMarkdown_returns_issues_for_round_1() {
        Set<String> existing = new HashSet<>();
        var round = WorkspaceParser.parseRoundFromMarkdown(
                fixture.resolve("responses"), 1, existing);
        assertNotNull(round);
        assertEquals(1, round.roundNumber());
        assertFalse(round.issues().isEmpty(), "round 1 should have issues");
    }

    @Test
    void parseTracker_returns_entries() {
        var entries = WorkspaceParser.parseTracker(fixture);
        assertNotNull(entries);
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=WorkspaceParserVisibilityTest`

Expected: compilation error — `parseRoundFromMarkdown`, `discoverMaxRound` are private.

- [ ] **Step 4: Change method visibility in WorkspaceParser**

In `WorkspaceParser.java`, change these 4 methods from `private static` to `static`
(package-visible):

1. `discoverMaxRound(Path responsesDir)` — remove `private`
2. `parseRoundFromMarkdown(Path responsesDir, int roundNum, Set<String> existingIds)` — remove `private`
3. `parseRoundFromJsonl(Path responsesDir, int roundNum)` — remove `private`
4. `parseTracker(Path workspaceDir)` — already static, change from `static` to `static`
   (verify it's accessible from same package — it should be, since it has no access modifier)

Use `ide_edit_member` for each method, replacing the full signature line.

- [ ] **Step 5: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=WorkspaceParserVisibilityTest`

Expected: PASS — all 3 tests green.

- [ ] **Step 6: Create workspace-watching test fixture**

Create the fixture directory structure. These are test resource files, not source code,
so use bash/Write:

`server/runtime/src/test/resources/fixtures/workspace-watching/.spec-path`:
```
/tmp/test-spec.md
```

`server/runtime/src/test/resources/fixtures/workspace-watching/responses/reviewer-1.md`:
```markdown
## R1-01: Missing error handling for null input

The `processInput()` method does not check for null values.
This will cause a NullPointerException at runtime.

SIGNAL: CONTINUE
```

`server/runtime/src/test/resources/fixtures/workspace-watching/responses/implementor-1.md`:
```markdown
## R1-01: FIXED

Added null check to `processInput()`. The method now returns early
with an empty result when input is null.

§3.2

SIGNAL: CONTINUE
```

`server/runtime/src/test/resources/fixtures/workspace-watching/tracker.md`:
```markdown
# Design Review Tracker

## Issues

### R1-01: Missing error handling for null input
- **Status:** FIXED
- **Spec commit:** abc123 → def456
```

`server/runtime/src/test/resources/fixtures/workspace-watching/progress.log`:
```
[10:00:00] Review (spec-review): /tmp/test-workspace
[10:00:00] Mode: spec-review
[10:00:00]
============================================================
[10:00:00]   ROUND 1
[10:00:00] ============================================================
[10:00:00]   Reviewer (fresh session)... (this may take 1-2 minutes)
[10:00:30]     [30s] reviewer: Reading spec and exploring codebase
[10:01:00]   Reviewer done ($1.50)
[10:01:00]   1 new issue(s) raised
[10:01:00]   Implementor (fresh session)... (this may take 1-2 minutes)
[10:01:30]     [30s] implementor: Verifying reviewer claims
[10:02:00]   Implementor done ($1.20)
[10:02:00]   Round 1 complete — ~$2.70/round, $2.70 cumulative
```

Note: this progress.log has NO terminal line — it represents a review still in progress.

- [ ] **Step 7: Run all existing tests to verify nothing is broken**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime`

Expected: all existing tests pass.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/drafthouse add server/runtime/pom.xml \
  server/runtime/src/main/java/io/casehub/drafthouse/debate/WorkspaceParser.java \
  server/runtime/src/test/java/io/casehub/drafthouse/debate/WorkspaceParserVisibilityTest.java \
  server/runtime/src/test/resources/fixtures/workspace-watching/
git -C /Users/mdproctor/claude/casehub/drafthouse commit -m "feat(#99): add directory-watcher dependency, expose parser round methods, add watching fixture"
```

---

### Task 2: Refactor WorkspaceReplayAdapter — extract dispatch methods, extend ReplayResult

**Files:**
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/debate/WorkspaceReplayAdapter.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/debate/WorkspaceReplayAdapterTest.java` (existing — must still pass)

**Interfaces:**
- Consumes: `WorkspaceParser.WorkspaceParseResult`, `WorkspaceParser.ParsedRound`,
  `DebateProtocol`, `ChannelMessageMeta`, `ConversationProtocol`
- Produces:
  - `dispatchIssues(UUID channelId, String sender, ParsedRound round, Map<String, Long> raiseMessageIds)` → `int` (entry count)
  - `dispatchResponses(UUID channelId, String sender, ParsedRound round, Map<String, Long> raiseMessageIds)` → `int`
  - `dispatchConfirmations(UUID channelId, String sender, ParsedRound round, Map<String, Long> raiseMessageIds)` → `int`
  - `dispatchMemos(UUID channelId, String sender, int roundNum, List<String> assumptions, List<ParsedSettledDecision> settled)` → `int`
  - `dispatchDeferred(UUID channelId, String sender, List<ParsedTrackerEntry> trackerStatuses, Map<String, Long> raiseMessageIds, WorkspaceParseResult parseResult)` → `int`
  - `dispatchEvidence(UUID channelId, String sender, List<ParsedTrackerEntry> trackerStatuses, WorkspaceParseResult parseResult)` → `int`
  - `dispatchRoundSnapshot(UUID channelId, String sender, int round, String commitHash, String documentPath, String label, Instant timestamp, String body)` (existing private → package-visible)
  - `ReplayResult(int entryCount, Map<String, String> statusDistribution, DocumentTimeline timeline, Map<Integer, String> snapshotContent, Map<String, Long> raiseMessageIds, long lastMessageId)` — extended record

- [ ] **Step 1: Run existing tests to establish baseline**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=WorkspaceReplayAdapterTest`

Expected: 3 tests PASS.

- [ ] **Step 2: Extend ReplayResult record**

In `WorkspaceReplayAdapter.java`, modify the `ReplayResult` record (line ~31) to add
two new fields:

```java
public record ReplayResult(
        int entryCount,
        Map<String, String> statusDistribution,
        DocumentTimeline timeline,
        Map<Integer, String> snapshotContent,
        Map<String, Long> raiseMessageIds,
        long lastMessageId) {}
```

Use `ide_edit_member` with `class=WorkspaceReplayAdapter`, `member=ReplayResult`.

- [ ] **Step 3: Add tenancyId field to WorkspaceReplayAdapter**

Add an optional `tenancyId` field. When non-null (watcher path), all
`MessageDispatch` builders include `.tenancyId(tenancyId)`. When null (replay path
called from MCP tool thread), the builder omits it and `MessageService` falls back to
`CurrentPrincipal`. Add a constructor overload:

```java
private final String tenancyId;

public WorkspaceReplayAdapter(MessageService messageService,
                               InstanceService instanceService,
                               ChannelGateway channelGateway,
                               WebSocketEventBus eventBus) {
    this(messageService, instanceService, channelGateway, eventBus, null);
}

public WorkspaceReplayAdapter(MessageService messageService,
                               InstanceService instanceService,
                               ChannelGateway channelGateway,
                               WebSocketEventBus eventBus,
                               String tenancyId) {
    this.messageService = messageService;
    this.instanceService = instanceService;
    this.channelGateway = channelGateway;
    this.eventBus = eventBus;
    this.tenancyId = tenancyId;
}
```

Add a helper used by all dispatch methods:

```java
private MessageDispatch.Builder dispatch() {
    var b = MessageDispatch.builder();
    if (tenancyId != null) b.tenancyId(tenancyId);
    return b;
}
```

All extracted dispatch methods use `dispatch()` instead of `MessageDispatch.builder()`.

- [ ] **Step 4: Extract dispatchIssues method**

Extract the RAISE entries loop (the `// 1. RAISE entries` block inside `replay()`)
into a package-visible method. Use `ide_insert_member` to add after the `replay()` method:

```java
int dispatchIssues(UUID channelId, String sender, WorkspaceParser.ParsedRound round,
                   Map<String, Long> raiseMessageIds) {
    int count = 0;
    int n = round.roundNumber();
    for (var issue : round.issues()) {
        String location = issue.location();
        if (location == null) {
            location = extractLocation(issue.body());
            if (location == null) {
                location = findLocationFromResponses(issue.issueId(), round.responses());
            }
        }
        String priority = issue.priority();
        var meta = buildMeta("RAISE", "REV", n, priority, null, location);
        String encoded = ChannelMessageMeta.encode(
                DebateProtocol.META_SENTINEL, meta, issue.title() + "\n\n" + issue.body());
        DispatchResult dr = messageService.dispatch(dispatch()
                .channelId(channelId).sender(sender).type(MessageType.QUERY)
                .content(encoded).correlationId(issue.issueId())
                .actorType(ActorType.AGENT).build());
        raiseMessageIds.put(issue.issueId(), dr.messageId());
        count++;
    }
    return count;
}
```

- [ ] **Step 5: Extract dispatchResponses method**

```java
int dispatchResponses(UUID channelId, String sender, WorkspaceParser.ParsedRound round,
                      Map<String, Long> raiseMessageIds) {
    int count = 0;
    int n = round.roundNumber();
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
        var dispatchBuilder = dispatch()
                .channelId(channelId).sender(sender).type(msgType)
                .content(encoded).correlationId(resp.issueId())
                .inReplyTo(inReplyTo).actorType(ActorType.AGENT);
        if (msgType == MessageType.HANDOFF) {
            dispatchBuilder.target(HUMAN_INSTANCE_ID);
        }
        messageService.dispatch(dispatchBuilder.build());
        count++;
    }
    return count;
}
```

- [ ] **Step 6: Extract dispatchConfirmations method**

```java
int dispatchConfirmations(UUID channelId, String sender, WorkspaceParser.ParsedRound round,
                          Map<String, Long> raiseMessageIds) {
    int count = 0;
    int n = round.roundNumber();
    for (var conf : round.confirmations()) {
        String entryType;
        MessageType msgType;
        String content;
        switch (conf.verdict()) {
            case "resolved" -> {
                entryType = "VERIFIED"; msgType = MessageType.DONE; content = "Fix verified.";
            }
            case "accepted" -> {
                entryType = "AGREE"; msgType = MessageType.DONE; content = "Rejection accepted.";
            }
            default -> {
                entryType = "DISPUTE"; msgType = MessageType.DECLINE;
                content = conf.reason().isEmpty() ? "Still open." : conf.reason();
            }
        }
        var meta = buildMeta(entryType, "REV", n, null, null, null);
        String encoded = ChannelMessageMeta.encode(
                DebateProtocol.META_SENTINEL, meta, content);
        Long inReplyTo = raiseMessageIds.get(conf.issueId());
        messageService.dispatch(dispatch()
                .channelId(channelId).sender(sender).type(msgType)
                .content(encoded).correlationId(conf.issueId())
                .inReplyTo(inReplyTo).actorType(ActorType.AGENT).build());
        count++;
    }
    return count;
}
```

- [ ] **Step 7: Extract dispatchMemos method**

```java
int dispatchMemos(UUID channelId, String sender, int roundNum,
                  List<String> assumptions,
                  List<WorkspaceParser.ParsedSettledDecision> settledDecisions) {
    int count = 0;
    for (String assumption : assumptions) {
        count += dispatchMemo(channelId, sender, roundNum, "ASSUMPTION: " + assumption);
    }
    for (var sd : settledDecisions) {
        String text = "SETTLED: " + sd.text();
        if (!sd.fromIssue().isEmpty()) { text += " (from " + sd.fromIssue() + ")"; }
        count += dispatchMemo(channelId, sender, roundNum, text);
    }
    return count;
}
```

- [ ] **Step 8: Extract dispatchDeferred and dispatchEvidence methods**

```java
int dispatchDeferred(UUID channelId, String sender,
                     List<WorkspaceParser.ParsedTrackerEntry> trackerStatuses,
                     Map<String, Long> raiseMessageIds,
                     WorkspaceParser.WorkspaceParseResult parseResult) {
    int count = 0;
    for (var te : trackerStatuses) {
        if ("DEFERRED".equals(te.status())) {
            int deferredRound = findDeferredRound(te.issueId(), parseResult);
            var meta = buildMeta("DEFERRED", "REV", deferredRound, null, null, null);
            String encoded = ChannelMessageMeta.encode(
                    DebateProtocol.META_SENTINEL, meta, "Issue deferred.");
            Long inReplyTo = raiseMessageIds.get(te.issueId());
            messageService.dispatch(dispatch()
                    .channelId(channelId).sender(sender).type(MessageType.DECLINE)
                    .content(encoded).correlationId(te.issueId())
                    .inReplyTo(inReplyTo).actorType(ActorType.AGENT).build());
            count++;
        }
    }
    return count;
}

int dispatchEvidence(UUID channelId, String sender,
                     List<WorkspaceParser.ParsedTrackerEntry> trackerStatuses,
                     WorkspaceParser.WorkspaceParseResult parseResult) {
    int count = 0;
    for (var te : trackerStatuses) {
        if (te.evidence() != null) {
            int evidenceRound = findEvidenceRound(te.issueId(), parseResult);
            count += dispatchMemo(channelId, sender, evidenceRound,
                    te.issueId() + ": spec commit " + te.evidence());
        }
    }
    return count;
}
```

- [ ] **Step 9: Promote dispatchRoundSnapshot to package-visible**

Change the existing `dispatchRoundSnapshot` method from `private` to package-visible
(remove the `private` modifier). Use `ide_edit_member`.

- [ ] **Step 10: Rewrite replay() as a thin loop over extracted methods**

Replace the body of `replay()` with calls to the extracted methods. The `replay()` method
becomes:

```java
public ReplayResult replay(DebateSession session,
                            WorkspaceParser.WorkspaceParseResult parseResult) {
    UUID   channelId = session.channelId();
    String revSender = registerSender(session, AgentType.REV);
    String impSender = registerSender(session, AgentType.IMP);

    channelGateway.initChannel(channelId,
                               new ChannelRef(channelId, session.channelName()));

    Map<String, Long> raiseMessageIds = new HashMap<>();
    int               entryCount      = 0;

    for (var round : parseResult.rounds()) {
        entryCount += dispatchIssues(channelId, revSender, round, raiseMessageIds);
        entryCount += dispatchResponses(channelId, impSender, round, raiseMessageIds);
        entryCount += dispatchConfirmations(channelId, revSender, round, raiseMessageIds);
        entryCount += dispatchMemos(channelId, revSender, round.roundNumber(),
                round.assumptions(), round.settledDecisions());
    }

    entryCount += dispatchDeferred(channelId, revSender,
            parseResult.trackerStatuses(), raiseMessageIds, parseResult);
    entryCount += dispatchEvidence(channelId, revSender,
            parseResult.trackerStatuses(), parseResult);

    // Timeline + ROUND_SNAPSHOT entries
    DocumentTimeline     timeline        = null;
    Map<Integer, String> snapshotContent = new LinkedHashMap<>();
    String               specPath        = parseResult.specPath();
    String               repoPath        = parseResult.projectRepoPath();

    if (specPath != null && repoPath != null) {
        Map<Integer, String>   roundCommits  = buildRoundCommitMap(parseResult);
        List<DocumentSnapshot> snapshots     = new ArrayList<>();
        int                    snapshotIndex = 0;

        String initialCommit = findInitialCommit(repoPath, specPath);
        if (initialCommit != null) {
            String content = gitShow(repoPath, initialCommit, specPath);
            if (content != null) {
                Instant commitTs = gitCommitTimestamp(repoPath, initialCommit);
                String  label    = "Round 0 (original)";
                var     source   = new SnapshotSource.GitCommit(initialCommit, commitTs, 0);
                snapshots.add(new DocumentSnapshot(specPath, label, source));
                snapshotContent.put(snapshotIndex, content);
                dispatchRoundSnapshot(channelId, revSender, 0, initialCommit, specPath,
                                      label, commitTs, label);
                entryCount++;
                snapshotIndex++;
            }
        }

        for (var entry : roundCommits.entrySet()) {
            int    roundNum   = entry.getKey();
            String commitHash = entry.getValue();
            String content    = gitShow(repoPath, commitHash, specPath);
            if (content != null) {
                long    issueCount = countIssuesInRound(roundNum, parseResult);
                long    fixCount   = countFixesInRound(roundNum, parseResult);
                String  label      = String.format("Round %d (+%d raised, %d fixed)",
                        roundNum, issueCount, fixCount);
                Instant commitTs   = gitCommitTimestamp(repoPath, commitHash);
                var     source     = new SnapshotSource.GitCommit(commitHash, commitTs, roundNum);
                snapshots.add(new DocumentSnapshot(specPath, label, source));
                snapshotContent.put(snapshotIndex, content);
                dispatchRoundSnapshot(channelId, revSender, roundNum, commitHash, specPath,
                                      label, commitTs, label);
                entryCount++;
                snapshotIndex++;
            }
        }

        if (!snapshots.isEmpty()) {
            timeline = new DocumentTimeline(specPath, snapshots);
        }
    }

    // Batch push to WebSocket
    var messages = messageService.pollAfter(channelId, 0L, Integer.MAX_VALUE);
    var entries = messages.stream()
                          .map(DebateStreamEntry::from)
                          .filter(Objects::nonNull)
                          .toList();
    eventBus.pushDebateEntries(channelId, entries);

    long lastMessageId = messages.isEmpty() ? 0L
            : messages.get(messages.size() - 1).id();

    Map<String, String> statusDist = new LinkedHashMap<>();
    for (var te : parseResult.trackerStatuses()) {
        statusDist.merge(te.status(), "1",
                         (a, b) -> String.valueOf(Integer.parseInt(a) + 1));
    }

    return new ReplayResult(entryCount, statusDist, timeline, snapshotContent,
            raiseMessageIds, lastMessageId);
}
```

Use `ide_replace_member` with `class=WorkspaceReplayAdapter`, `member=replay`.

- [ ] **Step 11: Run existing tests to verify refactoring is correct**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=WorkspaceReplayAdapterTest`

Expected: all 3 existing tests PASS — refactoring is behaviour-preserving.

- [ ] **Step 12: Run LoadWorkspaceTest to verify end-to-end**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=LoadWorkspaceTest`

Expected: all 4 tests PASS.

- [ ] **Step 13: Verify with ide_diagnostics**

Run `ide_diagnostics` on `WorkspaceReplayAdapter.java` to check for compilation errors.

- [ ] **Step 14: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/drafthouse add \
  server/runtime/src/main/java/io/casehub/drafthouse/debate/WorkspaceReplayAdapter.java
git -C /Users/mdproctor/claude/casehub/drafthouse commit -m "refactor(#99): extract per-entry dispatch methods from WorkspaceReplayAdapter, extend ReplayResult"
```

---

### Task 3: ProgressLogParser — parse progress.log lines into typed events

**Files:**
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/debate/ProgressLogParser.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/debate/ProgressLogParserTest.java`

**Interfaces:**
- Produces: `ProgressLogParser.parse(String line)` → `ProgressEvent` (sealed interface)
- Produces: `ProgressLogParser.isTerminal(String line)` → `boolean`
- Produces: `ProgressLogParser.terminalState(String line)` → `String` (DONE/PAUSED/FAILED/etc)
- Produces: `ProgressEvent` sealed interface with subtypes:
  `AgentStart`, `AgentStatus`, `AgentComplete`, `IssuesRaised`, `RoundComplete`, `ReviewTerminal`

- [ ] **Step 1: Write failing tests for ProgressLogParser**

Create `ProgressLogParserTest.java`:

```java
package io.casehub.drafthouse.debate;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

import static org.junit.jupiter.api.Assertions.*;

class ProgressLogParserTest {

    @Test
    void parse_agent_start_reviewer() {
        var event = ProgressLogParser.parse(
                "[10:00:00]   Reviewer (fresh session)... (this may take 1-2 minutes)");
        assertInstanceOf(ProgressLogParser.AgentStart.class, event);
        var start = (ProgressLogParser.AgentStart) event;
        assertEquals("reviewer", start.agent());
        assertEquals(false, start.cached());
    }

    @Test
    void parse_agent_start_implementor_cached() {
        var event = ProgressLogParser.parse(
                "[10:00:00]   Implementor (continued — cached context)... (this may take 1-2 minutes)");
        assertInstanceOf(ProgressLogParser.AgentStart.class, event);
        var start = (ProgressLogParser.AgentStart) event;
        assertEquals("implementor", start.agent());
        assertEquals(true, start.cached());
    }

    @Test
    void parse_agent_status() {
        var event = ProgressLogParser.parse(
                "[10:00:30]     [30s] reviewer: Reading spec and exploring codebase");
        assertInstanceOf(ProgressLogParser.AgentStatus.class, event);
        var status = (ProgressLogParser.AgentStatus) event;
        assertEquals("reviewer", status.agent());
        assertEquals(30, status.elapsedSeconds());
        assertEquals("Reading spec and exploring codebase", status.message());
    }

    @Test
    void parse_agent_complete() {
        var event = ProgressLogParser.parse("[10:01:00]   Reviewer done ($1.50)");
        assertInstanceOf(ProgressLogParser.AgentComplete.class, event);
        var complete = (ProgressLogParser.AgentComplete) event;
        assertEquals("reviewer", complete.agent());
        assertEquals(1.50, complete.cost(), 0.001);
    }

    @Test
    void parse_issues_raised() {
        var event = ProgressLogParser.parse("[10:01:00]   13 new issue(s) raised");
        assertInstanceOf(ProgressLogParser.IssuesRaised.class, event);
        assertEquals(13, ((ProgressLogParser.IssuesRaised) event).count());
    }

    @Test
    void parse_round_complete() {
        var event = ProgressLogParser.parse(
                "[10:02:00]   Round 1 complete — ~$2.70/round, $4.50 cumulative");
        assertInstanceOf(ProgressLogParser.RoundComplete.class, event);
        var rc = (ProgressLogParser.RoundComplete) event;
        assertEquals(1, rc.round());
        assertEquals(2.70, rc.roundCost(), 0.001);
        assertEquals(4.50, rc.cumulativeCost(), 0.001);
    }

    @ParameterizedTest
    @ValueSource(strings = {
            "REVIEW DONE",
            "REVIEW PAUSED",
            "REVIEW PAUSED: received SIGTERM — parent session likely ended",
            "REVIEW FAILED (exit 1)",
            "REVIEW ABORTED",
            "REVIEW CRASHED: java.lang.OutOfMemoryError",
            "REVIEW INTERRUPTED"
    })
    void parse_terminal_states(String line) {
        var event = ProgressLogParser.parse(line);
        assertInstanceOf(ProgressLogParser.ReviewTerminal.class, event);
        assertTrue(ProgressLogParser.isTerminal(line));
    }

    @Test
    void parse_terminal_extracts_state() {
        assertEquals("DONE", ProgressLogParser.terminalState("REVIEW DONE"));
        assertEquals("PAUSED", ProgressLogParser.terminalState(
                "REVIEW PAUSED: received SIGTERM — parent session likely ended"));
        assertEquals("FAILED", ProgressLogParser.terminalState("REVIEW FAILED (exit 1)"));
    }

    @Test
    void parse_unrecognised_line_returns_null() {
        assertNull(ProgressLogParser.parse("[10:00:00] Mode: spec-review"));
        assertNull(ProgressLogParser.parse("============================================================"));
        assertNull(ProgressLogParser.parse("[10:00:00]     [60s] no output file yet — agent may be reading/exploring"));
    }

    @Test
    void isTerminal_returns_false_for_non_terminal() {
        assertFalse(ProgressLogParser.isTerminal("[10:00:00]   Round 1 complete — ~$2.70/round, $4.50 cumulative"));
        assertFalse(ProgressLogParser.isTerminal(""));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=ProgressLogParserTest`

Expected: compilation error — `ProgressLogParser` does not exist.

- [ ] **Step 3: Implement ProgressLogParser**

Create `ProgressLogParser.java` using `ide_create_file`:

```java
package io.casehub.drafthouse.debate;

import java.util.regex.Matcher;
import java.util.regex.Pattern;

public final class ProgressLogParser {

    public sealed interface ProgressEvent {}
    public record AgentStart(String agent, boolean cached) implements ProgressEvent {}
    public record AgentStatus(String agent, int elapsedSeconds, String message) implements ProgressEvent {}
    public record AgentComplete(String agent, double cost) implements ProgressEvent {}
    public record IssuesRaised(int count) implements ProgressEvent {}
    public record RoundComplete(int round, double roundCost, double cumulativeCost) implements ProgressEvent {}
    public record ReviewTerminal(String finalState) implements ProgressEvent {}

    private static final Pattern AGENT_START = Pattern.compile(
            "\\[\\d{2}:\\d{2}:\\d{2}]\\s+(Reviewer|Implementor)\\s+\\((fresh session|continued .*cached.*)\\)");
    private static final Pattern AGENT_STATUS = Pattern.compile(
            "\\[\\d{2}:\\d{2}:\\d{2}]\\s+\\[(\\d+)s]\\s+(reviewer|implementor):\\s+(.+)");
    private static final Pattern AGENT_COMPLETE = Pattern.compile(
            "\\[\\d{2}:\\d{2}:\\d{2}]\\s+(Reviewer|Implementor)\\s+done\\s+\\(\\$(\\d+\\.\\d+)\\)");
    private static final Pattern ISSUES_RAISED = Pattern.compile(
            "\\[\\d{2}:\\d{2}:\\d{2}]\\s+(\\d+)\\s+new\\s+issue\\(s\\)\\s+raised");
    private static final Pattern ROUND_COMPLETE = Pattern.compile(
            "\\[\\d{2}:\\d{2}:\\d{2}]\\s+Round\\s+(\\d+)\\s+complete\\s+—\\s+~\\$(\\d+\\.\\d+)/round,\\s+\\$(\\d+\\.\\d+)\\s+cumulative");
    private static final Pattern TERMINAL = Pattern.compile(
            "REVIEW\\s+(DONE|PAUSED|FAILED|ABORTED|CRASHED|INTERRUPTED)\\b");

    private ProgressLogParser() {}

    public static ProgressEvent parse(String line) {
        if (line == null || line.isBlank()) return null;

        Matcher m;

        m = TERMINAL.matcher(line);
        if (m.find()) return new ReviewTerminal(m.group(1));

        m = AGENT_START.matcher(line);
        if (m.find()) return new AgentStart(m.group(1).toLowerCase(), m.group(2).contains("cached"));

        m = AGENT_STATUS.matcher(line);
        if (m.find()) return new AgentStatus(m.group(2), Integer.parseInt(m.group(1)), m.group(3).trim());

        m = AGENT_COMPLETE.matcher(line);
        if (m.find()) return new AgentComplete(m.group(1).toLowerCase(), Double.parseDouble(m.group(2)));

        m = ISSUES_RAISED.matcher(line);
        if (m.find()) return new IssuesRaised(Integer.parseInt(m.group(1)));

        m = ROUND_COMPLETE.matcher(line);
        if (m.find()) return new RoundComplete(
                Integer.parseInt(m.group(1)),
                Double.parseDouble(m.group(2)),
                Double.parseDouble(m.group(3)));

        return null;
    }

    public static boolean isTerminal(String line) {
        return line != null && TERMINAL.matcher(line).find();
    }

    public static String terminalState(String line) {
        if (line == null) return null;
        Matcher m = TERMINAL.matcher(line);
        return m.find() ? m.group(1) : null;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=ProgressLogParserTest`

Expected: all 10 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/drafthouse add \
  server/runtime/src/main/java/io/casehub/drafthouse/debate/ProgressLogParser.java \
  server/runtime/src/test/java/io/casehub/drafthouse/debate/ProgressLogParserTest.java
git -C /Users/mdproctor/claude/casehub/drafthouse commit -m "feat(#99): add ProgressLogParser for progress.log line parsing"
```

---

### Task 4: WorkspaceWatcher — core file watching + incremental dispatch

**Files:**
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/debate/WorkspaceWatcher.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/debate/WorkspaceWatcherTest.java`

**Interfaces:**
- Consumes: `WorkspaceReplayAdapter` (dispatch methods from Task 2), `WorkspaceParser` (round
  parsing from Task 1), `ProgressLogParser` (from Task 3), `WebSocketEventBus`, `DebateSession`,
  `MessageService`, `DirectoryWatcher` (`io.methvin:directory-watcher`)
- Produces:
  - `WorkspaceWatcher(WorkspaceReplayAdapter adapter, WebSocketEventBus eventBus, DebateSession session, MessageService messageService, String tenancyId, Runnable onComplete)` — constructor
  - `start(Path workspacePath, int startFromRound, Set<String> existingIssueIds, Map<String, Long> raiseMessageIds, long lastMessageId, String projectRepoPath, String specPath)` — begin watching
  - `stop()` — close DirectoryWatcher
  - Implements `Closeable`

- [ ] **Step 1: Write failing tests for WorkspaceWatcher**

Create `WorkspaceWatcherTest.java`. This test creates a temp directory, starts the watcher,
writes files, and asserts that dispatch methods are called. Use `@QuarkusTest` to get CDI beans:

```java
package io.casehub.drafthouse.debate;

import io.casehub.drafthouse.DebateSession;
import io.casehub.drafthouse.WebSocketEventBus;
import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelCreateRequest;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;
import io.casehub.qhorus.runtime.instance.InstanceService;
import io.casehub.qhorus.runtime.message.MessageService;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Objects;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class WorkspaceWatcherTest {

    @Inject ChannelService channelService;
    @Inject MessageService messageService;
    @Inject InstanceService instanceService;
    @Inject ChannelGateway channelGateway;
    @Inject WebSocketEventBus eventBus;

    @Test
    void watcher_dispatches_entries_for_new_reviewer_file(@TempDir Path tmpDir) throws Exception {
        Path responsesDir = tmpDir.resolve("responses");
        Files.createDirectories(responsesDir);
        Files.writeString(tmpDir.resolve(".spec-path"), "/tmp/test.md");

        Channel channel = channelService.create(ChannelCreateRequest.builder(
                "drafthouse/debate/watcher-test-" + System.nanoTime())
                .description("watcher test").semantic(ChannelSemantic.APPEND).build());

        DebateSession session = new DebateSession(
                channel.id(), channel.id().toString(), channel.name(), null);

        channelGateway.initChannel(channel.id(),
                new ChannelRef(channel.id(), channel.name()));

        String revId = DebateSession.instanceId(AgentType.REV, session.debateSessionId());
        instanceService.register(revId, "test rev", java.util.List.of("document-debate-rev"));
        session.registerIfAbsent(AgentType.REV, () -> revId);

        String impId = DebateSession.instanceId(AgentType.IMP, session.debateSessionId());
        instanceService.register(impId, "test imp", java.util.List.of("document-debate-imp"));
        session.registerIfAbsent(AgentType.IMP, () -> impId);

        var adapter = new WorkspaceReplayAdapter(
                messageService, instanceService, channelGateway, eventBus);

        var watcher = new WorkspaceWatcher(
                adapter, eventBus, session, messageService, null, () -> {});
        watcher.start(tmpDir, 0, new HashSet<>(), new HashMap<>(), 0L, null, null);

        Thread.sleep(500);

        Files.writeString(responsesDir.resolve("reviewer-1.md"),
                "## R1-01: Missing validation\n\nNo input validation.\n\nSIGNAL: CONTINUE\n");

        Thread.sleep(2000);

        var messages = messageService.pollAfter(channel.id(), 0L, Integer.MAX_VALUE);
        var entries = messages.stream()
                .map(DebateStreamEntry::from).filter(Objects::nonNull).toList();

        assertFalse(entries.isEmpty(), "watcher should have dispatched entries for reviewer-1.md");
        assertTrue(entries.stream().anyMatch(e -> e.entryType() == EntryType.RAISE),
                "should contain a RAISE entry");

        watcher.stop();
    }

    @Test
    void watcher_stops_on_terminal_state(@TempDir Path tmpDir) throws Exception {
        Path responsesDir = tmpDir.resolve("responses");
        Files.createDirectories(responsesDir);
        Files.writeString(tmpDir.resolve(".spec-path"), "/tmp/test.md");

        Channel channel = channelService.create(ChannelCreateRequest.builder(
                "drafthouse/debate/watcher-term-" + System.nanoTime())
                .description("watcher terminal test").semantic(ChannelSemantic.APPEND).build());

        DebateSession session = new DebateSession(
                channel.id(), channel.id().toString(), channel.name(), null);

        channelGateway.initChannel(channel.id(),
                new ChannelRef(channel.id(), channel.name()));

        String revId = DebateSession.instanceId(AgentType.REV, session.debateSessionId());
        instanceService.register(revId, "test rev", java.util.List.of("document-debate-rev"));
        session.registerIfAbsent(AgentType.REV, () -> revId);

        var adapter = new WorkspaceReplayAdapter(
                messageService, instanceService, channelGateway, eventBus);

        CountDownLatch completed = new CountDownLatch(1);
        var watcher = new WorkspaceWatcher(
                adapter, eventBus, session, messageService, null, completed::countDown);
        watcher.start(tmpDir, 0, new HashSet<>(), new HashMap<>(), 0L, null, null);

        Thread.sleep(500);

        Files.writeString(tmpDir.resolve("progress.log"), "REVIEW DONE\n");

        assertTrue(completed.await(5, TimeUnit.SECONDS),
                "watcher should invoke onComplete after terminal state");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=WorkspaceWatcherTest`

Expected: compilation error — `WorkspaceWatcher` does not exist.

- [ ] **Step 3: Implement WorkspaceWatcher**

Create `WorkspaceWatcher.java` using `ide_create_file`:

```java
package io.casehub.drafthouse.debate;

import io.casehub.drafthouse.DebateSession;
import io.casehub.drafthouse.WebSocketEventBus;
import io.casehub.qhorus.runtime.message.MessageService;
import io.methvin.watcher.DirectoryChangeEvent;
import io.methvin.watcher.DirectoryWatcher;

import java.io.Closeable;
import java.io.IOException;
import java.io.RandomAccessFile;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.logging.Logger;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class WorkspaceWatcher implements Closeable {

    private static final Logger LOG = Logger.getLogger(WorkspaceWatcher.class.getName());
    private static final Pattern RESPONSE_FILE = Pattern.compile(
            "(reviewer|implementor)-(\\d+)\\.(md|jsonl)$");

    private final WorkspaceReplayAdapter adapter;
    private final WebSocketEventBus eventBus;
    private final DebateSession session;
    private final MessageService messageService;
    private final String tenancyId;
    private final Runnable onComplete;

    private DirectoryWatcher directoryWatcher;
    private final AtomicBoolean stopped = new AtomicBoolean(false);
    private final Set<String> processedFiles = ConcurrentHashMap.newKeySet();
    private final Set<String> existingIssueIds = ConcurrentHashMap.newKeySet();
    private final Map<String, Long> raiseMessageIds = new ConcurrentHashMap<>();
    private volatile long lastMessageId;
    private volatile int lastReplayedRound;
    private volatile long progressLogOffset;
    private String projectRepoPath;
    private String specPath;
    private Path workspacePath;

    public WorkspaceWatcher(WorkspaceReplayAdapter adapter,
                            WebSocketEventBus eventBus,
                            DebateSession session,
                            MessageService messageService,
                            String tenancyId,
                            Runnable onComplete) {
        this.adapter = adapter;
        this.eventBus = eventBus;
        this.session = session;
        this.messageService = messageService;
        this.tenancyId = tenancyId;
        this.onComplete = onComplete;
    }

    public void start(Path workspacePath, int startFromRound,
                      Set<String> existingIssueIds,
                      Map<String, Long> raiseMessageIds,
                      long lastMessageId,
                      String projectRepoPath, String specPath) throws IOException {
        this.workspacePath = workspacePath;
        this.lastReplayedRound = startFromRound;
        this.existingIssueIds.addAll(existingIssueIds);
        this.raiseMessageIds.putAll(raiseMessageIds);
        this.lastMessageId = lastMessageId;
        this.projectRepoPath = projectRepoPath;
        this.specPath = specPath;

        Path progressLog = workspacePath.resolve("progress.log");
        this.progressLogOffset = Files.exists(progressLog) ? Files.size(progressLog) : 0;

        this.directoryWatcher = DirectoryWatcher.builder()
                .path(workspacePath)
                .listener(this::onEvent)
                .build();
        this.directoryWatcher.watchAsync();

        catchUpReconciliation();
    }

    @Override
    public void close() {
        stop();
    }

    public void stop() {
        if (stopped.compareAndSet(false, true) && directoryWatcher != null) {
            try {
                directoryWatcher.close();
            } catch (IOException e) {
                LOG.warning("Failed to close DirectoryWatcher: " + e.getMessage());
            }
        }
    }

    private void catchUpReconciliation() {
        Path responsesDir = workspacePath.resolve("responses");
        if (!Files.isDirectory(responsesDir)) return;

        int maxRound = WorkspaceParser.discoverMaxRound(responsesDir);
        for (int n = lastReplayedRound + 1; n <= maxRound; n++) {
            processReviewerFile(responsesDir, n);
            processImplementorFile(responsesDir, n);
        }
        if (lastReplayedRound < maxRound) {
            processPartialRound(responsesDir, maxRound);
        }
    }

    private void processPartialRound(Path responsesDir, int roundNum) {
        String reviewerStem = "reviewer-" + roundNum;
        if (Files.exists(responsesDir.resolve(reviewerStem + ".md"))
                || Files.exists(responsesDir.resolve(reviewerStem + ".jsonl"))) {
            processReviewerFile(responsesDir, roundNum);
        }
    }

    private void onEvent(DirectoryChangeEvent event) {
        if (stopped.get()) return;
        if (event.eventType() != DirectoryChangeEvent.EventType.CREATE
                && event.eventType() != DirectoryChangeEvent.EventType.MODIFY) return;

        Path path = event.path();
        String fileName = path.getFileName().toString();

        if (fileName.equals("progress.log")) {
            tailProgressLog();
            return;
        }

        if (fileName.equals("tracker.md")) {
            return;
        }

        Matcher m = RESPONSE_FILE.matcher(fileName);
        if (!m.matches()) return;

        String role = m.group(1);
        int roundNum = Integer.parseInt(m.group(2));
        Path responsesDir = path.getParent();

        // Activate CDI request context — watcher thread has none
        var rc = io.quarkus.arc.Arc.container().requestContext();
        rc.activate();
        try {
            if ("reviewer".equals(role)) {
                processReviewerFile(responsesDir, roundNum);
            } else {
                processImplementorFile(responsesDir, roundNum);
            }
        } finally {
            rc.deactivate();
        }
    }

    private void processReviewerFile(Path responsesDir, int roundNum) {
        String stem = "reviewer-" + roundNum;
        if (!processedFiles.add(stem)) return;

        if (!waitForFile(responsesDir, stem)) return;

        try {
            boolean hasJsonl = Files.exists(responsesDir.resolve(stem + ".jsonl"));
            WorkspaceParser.ParsedRound round = hasJsonl
                    ? WorkspaceParser.parseRoundFromJsonl(responsesDir, roundNum)
                    : WorkspaceParser.parseRoundFromMarkdown(
                            responsesDir, roundNum, existingIssueIds);

            String revSender = session.instanceIdFor(AgentType.REV);
            java.util.UUID channelId = session.channelId();

            int count = 0;
            count += adapter.dispatchIssues(channelId, revSender, round, raiseMessageIds);
            count += adapter.dispatchConfirmations(channelId, revSender, round, raiseMessageIds);
            count += adapter.dispatchMemos(channelId, revSender, round.roundNumber(),
                    round.assumptions(), round.settledDecisions());

            round.issues().forEach(i -> existingIssueIds.add(i.issueId()));

            if (count > 0) pushNewEntries();

            LOG.info("Watcher processed " + stem + ": " + count + " entries dispatched");
        } catch (Exception e) {
            LOG.warning("Failed to process " + stem + ": " + e.getMessage());
        }
    }

    private void processImplementorFile(Path responsesDir, int roundNum) {
        String stem = "implementor-" + roundNum;
        if (!processedFiles.add(stem)) return;

        if (!waitForFile(responsesDir, stem)) return;

        try {
            boolean hasJsonl = Files.exists(responsesDir.resolve(stem + ".jsonl"));
            WorkspaceParser.ParsedRound round = hasJsonl
                    ? WorkspaceParser.parseRoundFromJsonl(responsesDir, roundNum)
                    : WorkspaceParser.parseRoundFromMarkdown(
                            responsesDir, roundNum, existingIssueIds);

            String impSender = session.instanceIdFor(AgentType.IMP);
            java.util.UUID channelId = session.channelId();

            int count = 0;
            count += adapter.dispatchResponses(channelId, impSender, round, raiseMessageIds);

            if (count > 0) pushNewEntries();

            lastReplayedRound = roundNum;

            LOG.info("Watcher processed " + stem + ": " + count + " entries dispatched");
        } catch (Exception e) {
            LOG.warning("Failed to process " + stem + ": " + e.getMessage());
        }
    }

    private boolean waitForFile(Path dir, String stem) {
        Path md = dir.resolve(stem + ".md");
        Path jsonl = dir.resolve(stem + ".jsonl");
        for (int i = 0; i < 3; i++) {
            if (Files.exists(jsonl)) return true;
            if (Files.exists(md)) {
                try {
                    String content = Files.readString(md);
                    if (content.contains("SIGNAL:")) return true;
                } catch (IOException ignored) {}
            }
            try { Thread.sleep(500); } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return false;
            }
        }
        return Files.exists(jsonl) || Files.exists(md);
    }

    private void pushNewEntries() {
        java.util.UUID channelId = session.channelId();
        var newMessages = messageService.pollAfter(channelId, lastMessageId, Integer.MAX_VALUE);
        var newEntries = newMessages.stream()
                .map(DebateStreamEntry::from).filter(Objects::nonNull).toList();
        if (!newEntries.isEmpty()) {
            eventBus.pushDebateEntries(channelId, newEntries);
            lastMessageId = newMessages.get(newMessages.size() - 1).id();
        }
    }

    private void tailProgressLog() {
        Path logPath = workspacePath.resolve("progress.log");
        if (!Files.exists(logPath)) return;

        try {
            long fileSize = Files.size(logPath);
            if (fileSize <= progressLogOffset) return;

            String newContent;
            try (var raf = new RandomAccessFile(logPath.toFile(), "r")) {
                raf.seek(progressLogOffset);
                byte[] bytes = new byte[(int) (fileSize - progressLogOffset)];
                raf.readFully(bytes);
                newContent = new String(bytes);
            }
            progressLogOffset = fileSize;

            for (String line : newContent.split("\n")) {
                var event = ProgressLogParser.parse(line.trim());
                if (event == null) continue;

                Map<String, Object> payload = toPayload(event);
                eventBus.pushMetadata(session.channelId(), "workspace-progress", payload);

                if (event instanceof ProgressLogParser.ReviewTerminal terminal) {
                    LOG.info("Watcher detected terminal state: " + terminal.finalState());
                    stop();
                    onComplete.run();
                    return;
                }
            }
        } catch (IOException e) {
            LOG.warning("Failed to tail progress.log: " + e.getMessage());
        }
    }

    private static Map<String, Object> toPayload(ProgressLogParser.ProgressEvent event) {
        Map<String, Object> payload = new LinkedHashMap<>();
        switch (event) {
            case ProgressLogParser.AgentStart s -> {
                payload.put("type", "AGENT_START");
                payload.put("agent", s.agent());
                payload.put("cached", s.cached());
            }
            case ProgressLogParser.AgentStatus s -> {
                payload.put("type", "AGENT_STATUS");
                payload.put("agent", s.agent());
                payload.put("elapsed", s.elapsedSeconds());
                payload.put("message", s.message());
            }
            case ProgressLogParser.AgentComplete c -> {
                payload.put("type", "AGENT_COMPLETE");
                payload.put("agent", c.agent());
                payload.put("cost", c.cost());
            }
            case ProgressLogParser.IssuesRaised r -> {
                payload.put("type", "ISSUES_RAISED");
                payload.put("count", r.count());
            }
            case ProgressLogParser.RoundComplete r -> {
                payload.put("type", "ROUND_COMPLETE");
                payload.put("round", r.round());
                payload.put("cost", r.roundCost());
                payload.put("cumulativeCost", r.cumulativeCost());
            }
            case ProgressLogParser.ReviewTerminal t -> {
                payload.put("type", "REVIEW_TERMINAL");
                payload.put("finalState", t.finalState());
            }
        }
        return payload;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=WorkspaceWatcherTest`

Expected: 2 tests PASS.

- [ ] **Step 5: Verify with ide_diagnostics**

Run `ide_diagnostics` on `WorkspaceWatcher.java`.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/drafthouse add \
  server/runtime/src/main/java/io/casehub/drafthouse/debate/WorkspaceWatcher.java \
  server/runtime/src/test/java/io/casehub/drafthouse/debate/WorkspaceWatcherTest.java
git -C /Users/mdproctor/claude/casehub/drafthouse commit -m "feat(#99): add WorkspaceWatcher — file watching with incremental round dispatch"
```

---

### Task 5: DebateMcpTools integration — active watcher registry, loadWorkspace watching mode

**Files:**
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/DebateMcpTools.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/debate/LoadWorkspaceTest.java` (add new tests)

**Interfaces:**
- Consumes: `WorkspaceWatcher` (Task 4), `WorkspaceReplayAdapter.ReplayResult` (Task 2),
  `ProgressLogParser.isTerminal()` (Task 3)
- Produces:
  - `loadWorkspace()` returns `"status":"watching"` when review is still running
  - `activeWatchers` field — `ConcurrentHashMap<String, WorkspaceWatcher>`
  - `endDebate()` stops any active watcher before cleanup
  - `@PreDestroy shutdown()` closes all watchers

- [ ] **Step 1: Write failing tests for watching mode**

Add tests to `LoadWorkspaceTest.java`:

```java
@Test
void load_workspace_detects_in_progress_review() {
    String path = Path.of("src/test/resources/fixtures/workspace-watching")
            .toAbsolutePath().toString();
    String result = tools.loadWorkspace(path);

    assertFalse(result.startsWith("error:"), "should not be error: " + result);
    assertTrue(result.contains("\"status\":\"watching\"")
            || result.contains("\"status\":\"already_watching\""),
            "in-progress workspace should get watching status: " + result);
}

@Test
void load_workspace_completed_review_gets_loaded_status() {
    String path = Path.of("src/test/resources/fixtures/workspace-replay")
            .toAbsolutePath().toString();
    String result = tools.loadWorkspace(path);

    assertFalse(result.startsWith("error:"), "should not be error: " + result);
    // workspace-replay fixture has no progress.log → treated as complete
    assertTrue(result.contains("entryCount") || result.contains("already_loaded"),
            "completed workspace should get loaded/already_loaded: " + result);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=LoadWorkspaceTest`

Expected: `load_workspace_detects_in_progress_review` fails — current code always returns
`"status":"loaded"`.

- [ ] **Step 3: Add activeWatchers field and shutdown method to DebateMcpTools**

Add field after the `eventBus` injection:

```java
private final ConcurrentHashMap<String, WorkspaceWatcher> activeWatchers = new ConcurrentHashMap<>();
```

Add `@PreDestroy` method:

```java
@jakarta.annotation.PreDestroy
void shutdown() {
    activeWatchers.values().forEach(w -> {
        try { w.stop(); } catch (Exception e) { LOG.warning("shutdown: " + e.getMessage()); }
    });
    activeWatchers.clear();
}
```

Use `ide_insert_member` for both.

- [ ] **Step 4: Add isReviewComplete helper method**

```java
private static boolean isReviewComplete(Path wsPath) {
    Path progressLog = wsPath.resolve("progress.log");
    if (!java.nio.file.Files.exists(progressLog)) return true;
    try {
        List<String> lines = java.nio.file.Files.readAllLines(progressLog);
        for (int i = lines.size() - 1; i >= Math.max(0, lines.size() - 20); i--) {
            if (ProgressLogParser.isTerminal(lines.get(i).trim())) return true;
        }
        return false;
    } catch (IOException e) {
        LOG.warning("Could not read progress.log: " + e.getMessage());
        return true;
    }
}
```

Also add a helper to collect all issue IDs from a parse result:

```java
private static Set<String> collectIssueIds(WorkspaceParser.WorkspaceParseResult parseResult) {
    Set<String> ids = new java.util.HashSet<>();
    for (var round : parseResult.rounds()) {
        for (var issue : round.issues()) {
            ids.add(issue.issueId());
        }
    }
    return ids;
}
```

Use `ide_insert_member` for both, placing them near the existing private helpers.

- [ ] **Step 5: Modify loadWorkspace to start watcher when review is in progress**

In the `loadWorkspace()` method, after the existing replay logic (after `eventBus.broadcast("session-created", ...)`
and before the return statement), add watching logic.

The return block currently builds a JSON string. Replace the return section with:

```java
boolean reviewComplete = isReviewComplete(wsPath);

if (!reviewComplete) {
    var existingWatcher = activeWatchers.get(session.debateSessionId());
    if (existingWatcher != null) {
        return "{\"debateSessionId\":\"" + debateSessionId
               + "\",\"channel\":\"" + channel.name()
               + "\",\"status\":\"already_watching\"}";
    }

    var watchAdapter = new WorkspaceReplayAdapter(
            messageService, instanceService, channelGateway, eventBus);
    var watcher = new WorkspaceWatcher(watchAdapter, eventBus, session,
            messageService, channel.tenancyId(),
            () -> activeWatchers.remove(session.debateSessionId()));
    try {
        Set<String> existingIds = collectIssueIds(parseResult);
        watcher.start(wsPath, parseResult.rounds().size(), existingIds,
                result.raiseMessageIds(), result.lastMessageId(),
                parseResult.projectRepoPath(), parseResult.specPath());
        activeWatchers.put(session.debateSessionId(), watcher);
    } catch (IOException e) {
        LOG.warning("Failed to start workspace watcher: " + e.getMessage());
    }

    return "{\"debateSessionId\":\"" + debateSessionId
           + "\",\"channel\":\"" + channel.name()
           + "\",\"entryCount\":" + result.entryCount()
           + ",\"issues\":" + parseResult.trackerStatuses().size()
           + ",\"rounds\":" + parseResult.rounds().size()
           + ",\"status\":\"watching\"}";
}

return "{\"debateSessionId\":\"" + debateSessionId
       + "\",\"channel\":\"" + channel.name()
       + "\",\"entryCount\":" + result.entryCount()
       + ",\"issues\":" + parseResult.trackerStatuses().size()
       + ",\"rounds\":" + parseResult.rounds().size()
       + ",\"status\":\"loaded\"}";
```

Use `ide_replace_member` with `member=loadWorkspace` to replace the full method body,
incorporating the existing logic plus the new watching code.

- [ ] **Step 6: Modify endDebate to stop active watcher**

In `endDebate()`, add watcher cleanup before `registry.remove(channelId)`:

```java
var watcher = activeWatchers.remove(debateSessionId);
if (watcher != null) {
    try { watcher.stop(); }
    catch (Exception e) { LOG.warning("endDebate: watcher stop failed: " + e.getMessage()); }
}
```

Use `ide_read_file` to see the exact location in `endDebate()`, then use `ide_replace_member`.

- [ ] **Step 7: Run tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=LoadWorkspaceTest`

Expected: all tests PASS including the new watching mode test.

- [ ] **Step 8: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime`

Expected: all tests PASS.

- [ ] **Step 9: Verify with ide_diagnostics**

Run `ide_diagnostics` on `DebateMcpTools.java`.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/drafthouse add \
  server/runtime/src/main/java/io/casehub/drafthouse/DebateMcpTools.java \
  server/runtime/src/test/java/io/casehub/drafthouse/debate/LoadWorkspaceTest.java \
  server/runtime/src/test/resources/fixtures/workspace-watching/
git -C /Users/mdproctor/claude/casehub/drafthouse commit -m "feat(#99): integrate WorkspaceWatcher into load_workspace — auto-detect running reviews"
```

---

### Task 6: Workspace Status UI — `<workspace-status>` topbar element

**Files:**
- Create: `server/runtime/src/main/webui/src/panels/workspace-status.ts`
- Modify: `server/runtime/src/main/webui/src/index.ts` (add element to topbar)

**Interfaces:**
- Consumes: `workspace-progress` pages events from `WebSocketEventBus.pushMetadata()`
- Produces: `<workspace-status>` custom element registered in the browser

- [ ] **Step 1: Create workspace-status.ts**

Create the file using the Write tool (new file, not in IntelliJ):

`server/runtime/src/main/webui/src/panels/workspace-status.ts`:

```typescript
import { LitElement, html, css } from 'lit';
import { customElement, state } from 'lit/decorators.js';
import { onPagesEvent } from '@casehubio/pages-component';

interface WorkspaceProgressPayload {
  type: string;
  agent?: string;
  message?: string;
  elapsed?: number;
  cost?: number;
  round?: number;
  cumulativeCost?: number;
  count?: number;
  cached?: boolean;
  finalState?: string;
}

@customElement('workspace-status')
export class WorkspaceStatus extends LitElement {
  @state() private _visible = false;
  @state() private _text = '';
  @state() private _elapsed = 0;
  @state() private _terminal = false;

  private _cleanups: (() => void)[] = [];
  private _timer: ReturnType<typeof setInterval> | null = null;
  private _currentAgent = '';

  configure(_props: Record<string, unknown>): void {}

  override connectedCallback(): void {
    super.connectedCallback();
    this._cleanups.push(
      onPagesEvent<WorkspaceProgressPayload>(document, 'workspace-progress', (p) => {
        this._handleProgress(p);
      }),
      onPagesEvent(document, 'reconnected', () => {
        this._reset();
      }),
    );
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    this._cleanups.forEach(fn => fn());
    this._cleanups = [];
    this._stopTimer();
  }

  private _handleProgress(p: WorkspaceProgressPayload): void {
    this._visible = true;

    switch (p.type) {
      case 'AGENT_START':
        this._currentAgent = p.agent ?? '';
        this._elapsed = 0;
        this._terminal = false;
        this._text = `${this._currentAgent}${p.cached ? ' (cached)' : ''}...`;
        this._startTimer();
        break;

      case 'AGENT_STATUS':
        this._currentAgent = p.agent ?? this._currentAgent;
        this._elapsed = p.elapsed ?? this._elapsed;
        this._text = `${this._currentAgent}: ${p.message ?? ''}`;
        break;

      case 'AGENT_COMPLETE':
        this._stopTimer();
        this._text = `${p.agent ?? this._currentAgent} done ($${(p.cost ?? 0).toFixed(2)})`;
        break;

      case 'ISSUES_RAISED':
        this._text += ` — ${p.count} issue(s) raised`;
        break;

      case 'ROUND_COMPLETE':
        this._stopTimer();
        this._text = `Round ${p.round} complete — $${(p.cumulativeCost ?? 0).toFixed(2)} cumulative`;
        break;

      case 'REVIEW_TERMINAL':
        this._stopTimer();
        this._terminal = true;
        this._text = `Review ${(p.finalState ?? 'DONE').toLowerCase()}`;
        break;
    }
  }

  private _startTimer(): void {
    this._stopTimer();
    this._timer = setInterval(() => {
      this._elapsed++;
      this.requestUpdate();
    }, 1000);
  }

  private _stopTimer(): void {
    if (this._timer !== null) {
      clearInterval(this._timer);
      this._timer = null;
    }
  }

  private _reset(): void {
    this._visible = false;
    this._text = '';
    this._elapsed = 0;
    this._terminal = false;
    this._stopTimer();
  }

  private _formatElapsed(): string {
    const m = Math.floor(this._elapsed / 60);
    const s = this._elapsed % 60;
    return m > 0 ? `${m}m ${s}s` : `${s}s`;
  }

  static override styles = css`
    :host {
      display: inline-flex;
      align-items: center;
      gap: 6px;
      font-size: 11px;
      color: var(--muted, #888);
    }
    :host([hidden]) { display: none; }
    .dot {
      width: 6px; height: 6px; border-radius: 50%;
      background: var(--accent, #4a9eff);
      animation: pulse 1.5s ease-in-out infinite;
    }
    .dot.terminal { animation: none; background: var(--success, #4caf50); }
    .dot.failed { animation: none; background: var(--error, #f44336); }
    @keyframes pulse {
      0%, 100% { opacity: 1; }
      50% { opacity: 0.3; }
    }
    .elapsed { font-variant-numeric: tabular-nums; }
  `;

  override render() {
    if (!this._visible) return html``;
    const dotClass = this._terminal
      ? (this._text.includes('failed') || this._text.includes('crashed') ? 'dot failed' : 'dot terminal')
      : 'dot';
    return html`
      <span class="${dotClass}"></span>
      <span>${this._text}</span>
      ${this._timer !== null
        ? html`<span class="elapsed">(${this._formatElapsed()})</span>`
        : ''}
    `;
  }
}
```

- [ ] **Step 2: Wire into workbench topbar in index.ts**

In `server/runtime/src/main/webui/src/index.ts`, add the import at the top:

```typescript
import './panels/workspace-status.js';
```

Then in the topbar HTML template (inside the `<div id="topbar">`), add the element
before the `<span style="flex:1" id="topbar-spacer">`:

```html
<workspace-status></workspace-status>
```

Use the Edit tool on `index.ts` — this is a template string, not structural TypeScript.

- [ ] **Step 3: Build webui to verify TypeScript compiles**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml package -DskipTests -pl runtime`

Expected: build succeeds (Quinoa compiles TypeScript).

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/drafthouse add \
  server/runtime/src/main/webui/src/panels/workspace-status.ts \
  server/runtime/src/main/webui/src/index.ts
git -C /Users/mdproctor/claude/casehub/drafthouse commit -m "feat(#99): add workspace-status topbar element for live progress display"
```

---

### Task 7: Full integration test and final verification

**Files:**
- Modify: `server/runtime/src/test/java/io/casehub/drafthouse/debate/WorkspaceWatcherTest.java`
  (add implementor dispatch test)
- No new files

**Interfaces:**
- Consumes: all previous tasks

- [ ] **Step 1: Add test for implementor file dispatch**

Add to `WorkspaceWatcherTest.java`:

```java
@Test
void watcher_dispatches_responses_for_implementor_file(@TempDir Path tmpDir) throws Exception {
    Path responsesDir = tmpDir.resolve("responses");
    Files.createDirectories(responsesDir);
    Files.writeString(tmpDir.resolve(".spec-path"), "/tmp/test.md");

    Files.writeString(responsesDir.resolve("reviewer-1.md"),
            "## R1-01: Missing validation\n\nNo input validation.\n\nSIGNAL: CONTINUE\n");

    Channel channel = channelService.create(ChannelCreateRequest.builder(
            "drafthouse/debate/watcher-imp-" + System.nanoTime())
            .description("watcher implementor test").semantic(ChannelSemantic.APPEND).build());

    DebateSession session = new DebateSession(
            channel.id(), channel.id().toString(), channel.name(), null);

    channelGateway.initChannel(channel.id(),
            new ChannelRef(channel.id(), channel.name()));

    String revId = DebateSession.instanceId(AgentType.REV, session.debateSessionId());
    instanceService.register(revId, "test rev", java.util.List.of("document-debate-rev"));
    session.registerIfAbsent(AgentType.REV, () -> revId);

    String impId = DebateSession.instanceId(AgentType.IMP, session.debateSessionId());
    instanceService.register(impId, "test imp", java.util.List.of("document-debate-imp"));
    session.registerIfAbsent(AgentType.IMP, () -> impId);

    var adapter = new WorkspaceReplayAdapter(
            messageService, instanceService, channelGateway, eventBus);

    var existingIds = new HashSet<String>();
    var raiseIds = new HashMap<String, Long>();

    // Pre-process reviewer-1 as if it was replayed
    var round1 = WorkspaceParser.parseRoundFromMarkdown(responsesDir, 1, existingIds);
    adapter.dispatchIssues(channel.id(),
            session.instanceIdFor(AgentType.REV), round1, raiseIds);
    round1.issues().forEach(i -> existingIds.add(i.issueId()));
    long lastMsgId = messageService.pollAfter(channel.id(), 0L, Integer.MAX_VALUE)
            .stream().mapToLong(m -> m.id()).max().orElse(0L);

    var watcher = new WorkspaceWatcher(
            adapter, eventBus, session, messageService, null, () -> {});
    watcher.start(tmpDir, 0, existingIds, raiseIds, lastMsgId, null, null);

    Thread.sleep(500);

    Files.writeString(responsesDir.resolve("implementor-1.md"),
            "## R1-01: FIXED\n\nAdded null check.\n\n§3.2\n\nSIGNAL: CONTINUE\n");

    Thread.sleep(2000);

    var messages = messageService.pollAfter(channel.id(), lastMsgId, Integer.MAX_VALUE);
    var entries = messages.stream()
            .map(DebateStreamEntry::from).filter(Objects::nonNull).toList();

    assertTrue(entries.stream().anyMatch(e -> e.entryType() == EntryType.QUALIFY),
            "should contain a QUALIFY entry for the implementor response");

    watcher.stop();
}
```

- [ ] **Step 2: Run all tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime`

Expected: all tests PASS.

- [ ] **Step 3: Run ide_diagnostics on all modified files**

Check `WorkspaceReplayAdapter.java`, `WorkspaceWatcher.java`, `DebateMcpTools.java`,
`ProgressLogParser.java` for any compilation errors.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/drafthouse add \
  server/runtime/src/test/java/io/casehub/drafthouse/debate/WorkspaceWatcherTest.java
git -C /Users/mdproctor/claude/casehub/drafthouse commit -m "test(#99): add implementor dispatch integration test for WorkspaceWatcher"
```

- [ ] **Step 5: Push all commits**

```bash
git -C /Users/mdproctor/claude/casehub/drafthouse push
```
