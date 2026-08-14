# Review Pipeline Orchestration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #72 — Review pipeline orchestration — sequential multi-perspective review sessions
**Issue group:** #72

**Goal:** Build a DraftHouse browser panel that visualizes multi-dimension review pipelines in real time — pipeline progress, per-dimension findings, HIL checkpoint indicators, and decision validation.

**Architecture:** Server-side `PipelineSession` (data model in `server/api/`) with `PipelineOrchestrator` (state machine in `server/runtime/`) coordinates multiple `PipelineWatcher` instances (one per dimension, each tailing `progress.log`). Three new MCP tools (`start_pipeline`, `update_pipeline`, `load_decisions`) let the calling session create and update pipeline state. A new `review-pipeline` LitElement panel in `blocks-ui-document-workbench` subscribes to `pipeline-progress` and `pipeline-decisions` WebSocket events.

**Tech Stack:** Java 21 (sealed interfaces, records, pattern matching), Quarkus 3.34.3 (MCP tools, CDI), LitElement/TypeScript (browser panel), Playwright (E2E), io.methvin DirectoryWatcher (file watching)

## Global Constraints

- All new Java types in `server/api/` must be pure Java — no Quarkus or runtime dependencies
- `ProgressLogParser.ProgressEvent` is a sealed interface — adding records requires exhaustive switch updates in `WorkspaceWatcher.toPayload()`
- review.py EVENT format: `[HH:MM:SS] EVENT: {"type": "...", ...}` — the `EVENT:` prefix appears after the timestamp+space, embedded in a plain-text log line
- Panel tag names: bare descriptive names, no `drafthouse-` prefix (blocks-ui convention)
- Thread safety: multiple `PipelineWatcher` threads deliver events concurrently — all `PipelineSession` mutations must be synchronized

---

### Task 1: ProgressLogParser — dimension-level event parsing

**Files:**
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/debate/ProgressLogParser.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/debate/ProgressLogParserTest.java`
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/debate/WorkspaceWatcher.java` (toPayload switch)

**Interfaces:**
- Consumes: nothing (first task)
- Produces: `ProgressLogParser.DimensionStart`, `.RoundFindings`, `.RoundEnd`, `.DimensionDone` — all implement sealed `ProgressEvent`. Used by Task 3 (PipelineOrchestrator) and Task 2 (PipelineWatcher).

- [ ] **Step 1: Write failing tests for DimensionStart parsing**

```java
@Test
void parse_dimension_start() {
    var event = ProgressLogParser.parse(
            "[14:45:40] EVENT: {\"type\": \"dimension_start\", \"dimension\": \"coherence\", \"degree\": \"standard\", \"phase\": 1}");
    assertInstanceOf(ProgressLogParser.DimensionStart.class, event);
    var ds = (ProgressLogParser.DimensionStart) event;
    assertEquals("coherence", ds.dimension());
    assertEquals("standard", ds.degree());
    assertEquals(1, ds.phase());
}
```

- [ ] **Step 2: Write failing tests for RoundFindings, RoundEnd, DimensionDone**

```java
@Test
void parse_round_findings() {
    var event = ProgressLogParser.parse(
            "[14:46:54] EVENT: {\"type\": \"round_findings\", \"dimension\": \"structure\", \"round_number\": 1, \"issues\": {\"HIGH\": 3, \"MEDIUM\": 2, \"LOW\": 1}}");
    assertInstanceOf(ProgressLogParser.RoundFindings.class, event);
    var rf = (ProgressLogParser.RoundFindings) event;
    assertEquals("structure", rf.dimension());
    assertEquals(1, rf.roundNumber());
    assertEquals(6, rf.issueCount());
    assertEquals(Map.of("HIGH", 3, "MEDIUM", 2, "LOW", 1), rf.byPriority());
}

@Test
void parse_round_end() {
    var event = ProgressLogParser.parse(
            "[14:50:00] EVENT: {\"type\": \"round_end\", \"dimension\": \"robustness\", \"round_number\": 2, \"cost\": 1.50}");
    assertInstanceOf(ProgressLogParser.RoundEnd.class, event);
    var re = (ProgressLogParser.RoundEnd) event;
    assertEquals("robustness", re.dimension());
    assertEquals(2, re.roundNumber());
    assertEquals(1.50, re.cost(), 0.001);
}

@Test
void parse_dimension_done() {
    var event = ProgressLogParser.parse(
            "[14:55:00] EVENT: {\"type\": \"dimension_done\", \"dimension\": \"coherence\", \"total_rounds\": 3, \"cost\": 4.20, \"issues\": 8, \"verified\": 3, \"accepted\": 2, \"deferred\": 1, \"unresolved\": 2}");
    assertInstanceOf(ProgressLogParser.DimensionDone.class, event);
    var dd = (ProgressLogParser.DimensionDone) event;
    assertEquals("coherence", dd.dimension());
    assertEquals(3, dd.totalRounds());
    assertEquals(4.20, dd.cost(), 0.001);
    assertEquals(8, dd.issues());
}

@Test
void parse_unknown_event_type_returns_null() {
    var event = ProgressLogParser.parse(
            "[14:55:00] EVENT: {\"type\": \"unknown_future_event\", \"data\": 42}");
    assertNull(event);
}

@Test
void parse_malformed_json_after_event_prefix_returns_null() {
    var event = ProgressLogParser.parse("[14:55:00] EVENT: {not valid json}");
    assertNull(event);
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=ProgressLogParserTest -DfailIfNoTests=false`
Expected: compilation errors — new record types don't exist yet

- [ ] **Step 4: Add new record types and parseJsonEvent method**

Add four new records inside `ProgressLogParser`:

```java
public record DimensionStart(String dimension, String degree, int phase) implements ProgressEvent {}
public record RoundFindings(String dimension, int roundNumber, int issueCount,
                            Map<String, Integer> byPriority) implements ProgressEvent {}
public record RoundEnd(String dimension, int roundNumber, double cost) implements ProgressEvent {}
public record DimensionDone(String dimension, int totalRounds, double cost, int issues) implements ProgressEvent {}
```

Add a `parseJsonEvent` method and update `parse()` with an early check:

```java
private static final Pattern EVENT_LINE = Pattern.compile("EVENT:\\s+(\\{.+})");

public static ProgressEvent parse(String line) {
    if (line == null || line.isBlank()) return null;

    // JSON events first — they have a structured prefix
    Matcher em = EVENT_LINE.matcher(line);
    if (em.find()) return parseJsonEvent(em.group(1));

    // Existing regex parsing below...
    Matcher m;
    m = TERMINAL.matcher(line);
    // ... rest unchanged
}

private static ProgressEvent parseJsonEvent(String json) {
    try {
        var node = new com.fasterxml.jackson.databind.ObjectMapper().readTree(json);
        String type = node.path("type").asText("");
        return switch (type) {
            case "dimension_start" -> new DimensionStart(
                    node.path("dimension").asText(),
                    node.path("degree").asText(),
                    node.path("phase").asInt());
            case "round_findings" -> {
                Map<String, Integer> byPriority = new java.util.LinkedHashMap<>();
                node.path("issues").fields().forEachRemaining(
                        e -> byPriority.put(e.getKey(), e.getValue().asInt()));
                yield new RoundFindings(
                        node.path("dimension").asText(),
                        node.path("round_number").asInt(),
                        byPriority.values().stream().mapToInt(Integer::intValue).sum(),
                        byPriority);
            }
            case "round_end" -> new RoundEnd(
                    node.path("dimension").asText(),
                    node.path("round_number").asInt(),
                    node.path("cost").asDouble());
            case "dimension_done" -> new DimensionDone(
                    node.path("dimension").asText(),
                    node.path("total_rounds").asInt(),
                    node.path("cost").asDouble(),
                    node.path("issues").asInt());
            default -> null;
        };
    } catch (Exception e) {
        return null;
    }
}
```

- [ ] **Step 5: Update WorkspaceWatcher.toPayload() exhaustive switch**

Add cases for the new event types in the `toPayload` method:

```java
case ProgressLogParser.DimensionStart ds -> {
    payload.put("type", "DIMENSION_START");
    payload.put("dimension", ds.dimension());
    payload.put("degree", ds.degree());
    payload.put("phase", ds.phase());
}
case ProgressLogParser.RoundFindings rf -> {
    payload.put("type", "ROUND_FINDINGS");
    payload.put("dimension", rf.dimension());
    payload.put("round", rf.roundNumber());
    payload.put("issueCount", rf.issueCount());
    payload.put("byPriority", rf.byPriority());
}
case ProgressLogParser.RoundEnd re -> {
    payload.put("type", "ROUND_END");
    payload.put("dimension", re.dimension());
    payload.put("round", re.roundNumber());
    payload.put("cost", re.cost());
}
case ProgressLogParser.DimensionDone dd -> {
    payload.put("type", "DIMENSION_DONE");
    payload.put("dimension", dd.dimension());
    payload.put("totalRounds", dd.totalRounds());
    payload.put("cost", dd.cost());
    payload.put("issues", dd.issues());
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=ProgressLogParserTest`
Expected: all tests PASS

- [ ] **Step 7: Commit**

```bash
git -C server add runtime/src/main/java/io/casehub/drafthouse/debate/ProgressLogParser.java runtime/src/test/java/io/casehub/drafthouse/debate/ProgressLogParserTest.java runtime/src/main/java/io/casehub/drafthouse/debate/WorkspaceWatcher.java
git commit -m "feat(#72): add dimension-level event parsing to ProgressLogParser

Refs #72"
```

---

### Task 2: PipelineSession domain model and PipelineDecisionParser

**Files:**
- Create: `server/api/src/main/java/io/casehub/drafthouse/debate/PipelineSession.java`
- Create: `server/api/src/main/java/io/casehub/drafthouse/debate/DimensionDescriptor.java`
- Create: `server/api/src/main/java/io/casehub/drafthouse/debate/PipelinePhase.java`
- Create: `server/api/src/main/java/io/casehub/drafthouse/debate/CheckpointStatus.java`
- Create: `server/api/src/main/java/io/casehub/drafthouse/debate/DimensionStatus.java`
- Create: `server/api/src/main/java/io/casehub/drafthouse/debate/PipelineDecision.java`
- Create: `server/api/src/main/java/io/casehub/drafthouse/debate/PipelineFinding.java`
- Create: `server/api/src/main/java/io/casehub/drafthouse/debate/PipelineDecisionParser.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/debate/PipelineSessionTest.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/debate/PipelineDecisionParserTest.java`

**Interfaces:**
- Consumes: nothing (pure data model)
- Produces:
  - `PipelineSession` — mutable, synchronized. Fields: `pipelineId()`, `debateSessionId()`, `dimensions()`, `ordered()`, `specPath()`, `currentPhase()`, `checkpointStatus()`, `decisions()`. Mutators: `advanceDimension(String name, DimensionStatus status)`, `setPhase(PipelinePhase)`, `setCheckpoint(CheckpointStatus)`, `updateDimensionRound(String name, int round)`, `updateDimensionIssues(String name, Map<String, Integer> byPriority)`, `updateDimensionCost(String name, double cost)`, `addFinding(String dimension, PipelineFinding finding)`, `setDecisions(List<PipelineDecision>)`, `toSnapshot()` (returns immutable state map for WebSocket push).
  - `PipelineDecisionParser.parse(String markdown)` → `List<PipelineDecision>`
  - Enums: `PipelinePhase`, `CheckpointStatus`, `DimensionStatus`
  - Records: `DimensionDescriptor`, `PipelineDecision`, `PipelineFinding`

- [ ] **Step 1: Write failing tests for PipelineSession**

```java
package io.casehub.drafthouse.debate;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class PipelineSessionTest {

    @Test
    void create_with_dimensions_all_pending() {
        var dims = List.of(
                new DimensionDescriptor("coherence", "/tmp/ws/coherence", "standard"),
                new DimensionDescriptor("structure", "/tmp/ws/structure", "standard"));
        var session = new PipelineSession("p1", "d1", dims, false, "/tmp/spec.md");
        assertEquals(PipelinePhase.ROUND_1, session.currentPhase());
        assertEquals(CheckpointStatus.NONE, session.checkpointStatus());
        session.dimensions().forEach(d ->
                assertEquals(DimensionStatus.PENDING, d.status()));
    }

    @Test
    void advance_dimension_to_running() {
        var dims = List.of(new DimensionDescriptor("coherence", "/tmp/ws", "standard"));
        var session = new PipelineSession("p1", "d1", dims, false, "/tmp/spec.md");
        session.advanceDimension("coherence", DimensionStatus.RUNNING);
        assertEquals(DimensionStatus.RUNNING, session.dimensions().get(0).status());
    }

    @Test
    void advance_dimension_to_killed() {
        var dims = List.of(new DimensionDescriptor("coherence", "/tmp/ws", "standard"));
        var session = new PipelineSession("p1", "d1", dims, false, "/tmp/spec.md");
        session.advanceDimension("coherence", DimensionStatus.RUNNING);
        session.advanceDimension("coherence", DimensionStatus.KILLED);
        assertEquals(DimensionStatus.KILLED, session.dimensions().get(0).status());
    }

    @Test
    void advance_dimension_to_failed() {
        var dims = List.of(new DimensionDescriptor("coherence", "/tmp/ws", "standard"));
        var session = new PipelineSession("p1", "d1", dims, false, "/tmp/spec.md");
        session.advanceDimension("coherence", DimensionStatus.RUNNING);
        session.advanceDimension("coherence", DimensionStatus.FAILED);
        assertEquals(DimensionStatus.FAILED, session.dimensions().get(0).status());
    }

    @Test
    void phase_and_checkpoint_transitions() {
        var dims = List.of(new DimensionDescriptor("coherence", "/tmp/ws", "standard"));
        var session = new PipelineSession("p1", "d1", dims, false, "/tmp/spec.md");
        session.setPhase(PipelinePhase.HIL_CHECKPOINT_1);
        session.setCheckpoint(CheckpointStatus.PENDING);
        assertEquals(PipelinePhase.HIL_CHECKPOINT_1, session.currentPhase());
        assertEquals(CheckpointStatus.PENDING, session.checkpointStatus());
        session.setCheckpoint(CheckpointStatus.RESOLVED);
        assertEquals(CheckpointStatus.RESOLVED, session.checkpointStatus());
    }

    @Test
    void update_dimension_round_and_issues() {
        var dims = List.of(new DimensionDescriptor("coherence", "/tmp/ws", "standard"));
        var session = new PipelineSession("p1", "d1", dims, false, "/tmp/spec.md");
        session.updateDimensionRound("coherence", 2);
        session.updateDimensionIssues("coherence", Map.of("HIGH", 3, "MEDIUM", 1));
        var dim = session.dimensions().get(0);
        assertEquals(2, dim.currentRound());
        assertEquals(Map.of("HIGH", 3, "MEDIUM", 1), dim.issuesByPriority());
    }

    @Test
    void to_snapshot_returns_map() {
        var dims = List.of(new DimensionDescriptor("coherence", "/tmp/ws", "standard"));
        var session = new PipelineSession("p1", "d1", dims, false, "/tmp/spec.md");
        var snap = session.toSnapshot();
        assertEquals("p1", snap.get("pipelineId"));
        assertEquals("ROUND_1", snap.get("phase"));
        assertInstanceOf(List.class, snap.get("dimensions"));
    }
}
```

- [ ] **Step 2: Write failing tests for PipelineDecisionParser**

```java
package io.casehub.drafthouse.debate;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class PipelineDecisionParserTest {

    private static final String DECISIONS_MD = """
            ## D1: Architecture choice

            **Choice:** Thin registry
            **Alternatives:**
            - Heavy orchestrator — duplicates logic
            - Middleware — stateless, no checkpoint tracking
            **Rationale:** Preserves separation of concerns
            **Trade-offs:** Extra MCP calls needed
            **Exploration:** quick
            **Status:** captured

            ## D2: Event model

            **Choice:** Extend ProgressLogParser
            **Alternatives:**
            - JSONL bypass — breaks single-parser contract
            **Rationale:** Incremental extension of existing patterns
            **Trade-offs:** More parser types to maintain
            **Exploration:** deep-analysis
            **Depends on:** D1 (Architecture choice)
            **Status:** confirmed
            """;

    @Test
    void parse_two_decisions() {
        var decisions = PipelineDecisionParser.parse(DECISIONS_MD);
        assertEquals(2, decisions.size());
    }

    @Test
    void parse_first_decision_fields() {
        var d = PipelineDecisionParser.parse(DECISIONS_MD).get(0);
        assertEquals("D1", d.id());
        assertEquals("Architecture choice", d.title());
        assertEquals("Thin registry", d.choice());
        assertEquals(2, d.alternatives().size());
        assertEquals("Preserves separation of concerns", d.rationale());
        assertEquals("Extra MCP calls needed", d.tradeoffs());
        assertEquals("quick", d.explorationDepth());
        assertEquals("captured", d.status());
        assertNull(d.dependsOn());
    }

    @Test
    void parse_depends_on() {
        var d = PipelineDecisionParser.parse(DECISIONS_MD).get(1);
        assertEquals("D2", d.id());
        assertEquals("D1 (Architecture choice)", d.dependsOn());
        assertEquals("confirmed", d.status());
    }

    @Test
    void parse_empty_input() {
        var decisions = PipelineDecisionParser.parse("");
        assertTrue(decisions.isEmpty());
    }

    @Test
    void parse_null_input() {
        var decisions = PipelineDecisionParser.parse(null);
        assertTrue(decisions.isEmpty());
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest="PipelineSessionTest,PipelineDecisionParserTest" -DfailIfNoTests=false`
Expected: compilation errors — types don't exist yet

- [ ] **Step 4: Create enum types**

Create `PipelinePhase.java`:
```java
package io.casehub.drafthouse.debate;

public enum PipelinePhase {
    ROUND_1, HIL_CHECKPOINT_1, REMAINING_ROUNDS, HIL_CHECKPOINT_2, CROSS_CUTTING, COMPLETE
}
```

Create `CheckpointStatus.java`:
```java
package io.casehub.drafthouse.debate;

public enum CheckpointStatus { NONE, PENDING, RESOLVED }
```

Create `DimensionStatus.java`:
```java
package io.casehub.drafthouse.debate;

public enum DimensionStatus { PENDING, RUNNING, DONE, KILLED, FAILED }
```

- [ ] **Step 5: Create record types**

Create `PipelineDecision.java`:
```java
package io.casehub.drafthouse.debate;

import java.util.List;

public record PipelineDecision(
        String id, String title, String choice,
        List<String> alternatives, String rationale,
        String tradeoffs, String status,
        String explorationDepth, String dependsOn) {}
```

Create `PipelineFinding.java`:
```java
package io.casehub.drafthouse.debate;

public record PipelineFinding(
        String dimension, String issueId, String priority,
        String summary, String status, String location) {}
```

- [ ] **Step 6: Create DimensionDescriptor**

```java
package io.casehub.drafthouse.debate;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public final class DimensionDescriptor {
    private final String name;
    private final String workspacePath;
    private final String degree;
    private volatile DimensionStatus status = DimensionStatus.PENDING;
    private volatile int currentRound;
    private volatile int totalRounds;
    private volatile double cost;
    private volatile int elapsedSeconds;
    private final Map<String, Integer> issuesByPriority = new LinkedHashMap<>();
    private final List<PipelineFinding> findings = new ArrayList<>();

    public DimensionDescriptor(String name, String workspacePath, String degree) {
        this.name = name;
        this.workspacePath = workspacePath;
        this.degree = degree;
    }

    // Getters
    public String name() { return name; }
    public String workspacePath() { return workspacePath; }
    public String degree() { return degree; }
    public DimensionStatus status() { return status; }
    public int currentRound() { return currentRound; }
    public int totalRounds() { return totalRounds; }
    public double cost() { return cost; }
    public int elapsedSeconds() { return elapsedSeconds; }
    public Map<String, Integer> issuesByPriority() { return Map.copyOf(issuesByPriority); }
    public List<PipelineFinding> findings() { return List.copyOf(findings); }

    // Mutators (called under PipelineSession's synchronized block)
    void setStatus(DimensionStatus status) { this.status = status; }
    void setCurrentRound(int round) { this.currentRound = round; }
    void setTotalRounds(int total) { this.totalRounds = total; }
    void setCost(double cost) { this.cost = cost; }
    void setElapsedSeconds(int seconds) { this.elapsedSeconds = seconds; }
    void setIssuesByPriority(Map<String, Integer> byPriority) {
        this.issuesByPriority.clear();
        this.issuesByPriority.putAll(byPriority);
    }
    void addFinding(PipelineFinding finding) { this.findings.add(finding); }

    Map<String, Object> toMap() {
        var map = new LinkedHashMap<String, Object>();
        map.put("name", name);
        map.put("status", status.name());
        map.put("currentRound", currentRound);
        map.put("totalRounds", totalRounds);
        map.put("degree", degree);
        map.put("issuesByPriority", Map.copyOf(issuesByPriority));
        map.put("cost", cost);
        map.put("elapsed", elapsedSeconds);
        map.put("findingsCount", findings.size());
        return map;
    }
}
```

- [ ] **Step 7: Create PipelineSession**

```java
package io.casehub.drafthouse.debate;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public final class PipelineSession {
    private final String pipelineId;
    private final String debateSessionId;
    private final List<DimensionDescriptor> dimensions;
    private final boolean ordered;
    private final String specPath;
    private volatile PipelinePhase currentPhase = PipelinePhase.ROUND_1;
    private volatile CheckpointStatus checkpointStatus = CheckpointStatus.NONE;
    private volatile List<PipelineDecision> decisions = List.of();

    public PipelineSession(String pipelineId, String debateSessionId,
                           List<DimensionDescriptor> dimensions,
                           boolean ordered, String specPath) {
        this.pipelineId = pipelineId;
        this.debateSessionId = debateSessionId;
        this.dimensions = new ArrayList<>(dimensions);
        this.ordered = ordered;
        this.specPath = specPath;
    }

    public String pipelineId() { return pipelineId; }
    public String debateSessionId() { return debateSessionId; }
    public List<DimensionDescriptor> dimensions() { return List.copyOf(dimensions); }
    public boolean ordered() { return ordered; }
    public String specPath() { return specPath; }
    public PipelinePhase currentPhase() { return currentPhase; }
    public CheckpointStatus checkpointStatus() { return checkpointStatus; }
    public List<PipelineDecision> decisions() { return decisions; }

    public synchronized void advanceDimension(String name, DimensionStatus status) {
        findDimension(name).setStatus(status);
    }

    public synchronized void setPhase(PipelinePhase phase) { this.currentPhase = phase; }
    public synchronized void setCheckpoint(CheckpointStatus status) { this.checkpointStatus = status; }

    public synchronized void updateDimensionRound(String name, int round) {
        findDimension(name).setCurrentRound(round);
    }

    public synchronized void updateDimensionIssues(String name, Map<String, Integer> byPriority) {
        findDimension(name).setIssuesByPriority(byPriority);
    }

    public synchronized void updateDimensionCost(String name, double cost) {
        findDimension(name).setCost(cost);
    }

    public synchronized void addFinding(String dimension, PipelineFinding finding) {
        findDimension(dimension).addFinding(finding);
    }

    public synchronized void setDecisions(List<PipelineDecision> decisions) {
        this.decisions = List.copyOf(decisions);
    }

    public synchronized Map<String, Object> toSnapshot() {
        var map = new LinkedHashMap<String, Object>();
        map.put("pipelineId", pipelineId);
        map.put("phase", currentPhase.name());
        map.put("checkpointStatus", checkpointStatus.name());
        map.put("ordered", ordered);
        map.put("dimensions", dimensions.stream().map(DimensionDescriptor::toMap).toList());
        return map;
    }

    private DimensionDescriptor findDimension(String name) {
        return dimensions.stream()
                .filter(d -> d.name().equals(name))
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException("Unknown dimension: " + name));
    }
}
```

- [ ] **Step 8: Create PipelineDecisionParser**

```java
package io.casehub.drafthouse.debate;

import java.util.ArrayList;
import java.util.List;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public final class PipelineDecisionParser {
    private static final Pattern DECISION_HEADER = Pattern.compile("^## (D\\d+):\\s+(.+)$", Pattern.MULTILINE);

    private PipelineDecisionParser() {}

    public static List<PipelineDecision> parse(String markdown) {
        if (markdown == null || markdown.isBlank()) return List.of();
        var decisions = new ArrayList<PipelineDecision>();
        Matcher m = DECISION_HEADER.matcher(markdown);
        var starts = new ArrayList<int[]>();
        while (m.find()) starts.add(new int[]{m.start(), m.end()});

        for (int i = 0; i < starts.size(); i++) {
            int sectionStart = starts.get(i)[0];
            int sectionEnd = i + 1 < starts.size() ? starts.get(i + 1)[0] : markdown.length();
            String header = markdown.substring(starts.get(i)[0], starts.get(i)[1]);
            Matcher hm = DECISION_HEADER.matcher(header);
            if (!hm.find()) continue;
            String id = hm.group(1);
            String title = hm.group(2).trim();
            String section = markdown.substring(sectionStart, sectionEnd);

            decisions.add(new PipelineDecision(
                    id, title,
                    extractField(section, "Choice"),
                    extractAlternatives(section),
                    extractField(section, "Rationale"),
                    extractField(section, "Trade-offs"),
                    extractField(section, "Status"),
                    extractField(section, "Exploration"),
                    extractField(section, "Depends on")));
        }
        return decisions;
    }

    private static String extractField(String section, String fieldName) {
        Pattern p = Pattern.compile("\\*\\*" + fieldName + ":\\*\\*\\s*(.+)$", Pattern.MULTILINE);
        Matcher m = p.matcher(section);
        return m.find() ? m.group(1).trim() : null;
    }

    private static List<String> extractAlternatives(String section) {
        int idx = section.indexOf("**Alternatives:**");
        if (idx < 0) return List.of();
        var alts = new ArrayList<String>();
        String[] lines = section.substring(idx).split("\n");
        for (int i = 1; i < lines.length; i++) {
            String line = lines[i].trim();
            if (line.startsWith("- ")) alts.add(line.substring(2).trim());
            else if (line.startsWith("**")) break;
            else if (!line.isEmpty()) break;
        }
        return alts;
    }
}
```

- [ ] **Step 9: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest="PipelineSessionTest,PipelineDecisionParserTest"`
Expected: all tests PASS

- [ ] **Step 10: Commit**

```bash
git add server/api/src/main/java/io/casehub/drafthouse/debate/PipelineSession.java server/api/src/main/java/io/casehub/drafthouse/debate/DimensionDescriptor.java server/api/src/main/java/io/casehub/drafthouse/debate/PipelinePhase.java server/api/src/main/java/io/casehub/drafthouse/debate/CheckpointStatus.java server/api/src/main/java/io/casehub/drafthouse/debate/DimensionStatus.java server/api/src/main/java/io/casehub/drafthouse/debate/PipelineDecision.java server/api/src/main/java/io/casehub/drafthouse/debate/PipelineFinding.java server/api/src/main/java/io/casehub/drafthouse/debate/PipelineDecisionParser.java server/runtime/src/test/java/io/casehub/drafthouse/debate/PipelineSessionTest.java server/runtime/src/test/java/io/casehub/drafthouse/debate/PipelineDecisionParserTest.java
git commit -m "feat(#72): PipelineSession domain model and PipelineDecisionParser

Refs #72"
```

---

### Task 3: PipelineWatcher and PipelineOrchestrator

**Files:**
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/debate/PipelineWatcher.java`
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/debate/PipelineOrchestrator.java`
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/debate/PipelineSessionRegistry.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/debate/PipelineWatcherTest.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/debate/PipelineOrchestratorTest.java`

**Interfaces:**
- Consumes: `ProgressLogParser.parse()` (Task 1), `PipelineSession` + enums (Task 2)
- Produces:
  - `PipelineWatcher(Path workspacePath, Consumer<ProgressLogParser.ProgressEvent> listener)` — `start()`, `stop()`, `close()`
  - `PipelineOrchestrator` — `@ApplicationScoped`, `onEvent(PipelineSession session, ProgressLogParser.ProgressEvent event)` → mutates session, returns `boolean pushNeeded`
  - `PipelineSessionRegistry` — `@ApplicationScoped`, `create(PipelineSession)`, `get(String)`, `remove(String)`

- [ ] **Step 1: Write failing test for PipelineWatcher**

```java
package io.casehub.drafthouse.debate;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import static org.junit.jupiter.api.Assertions.*;

class PipelineWatcherTest {

    @Test
    void tails_progress_log_and_delivers_events(@TempDir Path tmpDir) throws Exception {
        var events = new ArrayList<ProgressLogParser.ProgressEvent>();
        var latch = new CountDownLatch(1);
        var watcher = new PipelineWatcher(tmpDir, event -> {
            events.add(event);
            if (event instanceof ProgressLogParser.ReviewTerminal) latch.countDown();
        });
        watcher.start();

        Path log = tmpDir.resolve("progress.log");
        Files.writeString(log, "[10:00:00]   Reviewer (fresh session)\n");
        Thread.sleep(500);
        Files.writeString(log,
                Files.readString(log) + "REVIEW DONE\n");

        assertTrue(latch.await(5, TimeUnit.SECONDS));
        watcher.stop();

        assertTrue(events.stream().anyMatch(e -> e instanceof ProgressLogParser.AgentStart));
        assertTrue(events.stream().anyMatch(e -> e instanceof ProgressLogParser.ReviewTerminal));
    }
}
```

- [ ] **Step 2: Write failing test for PipelineOrchestrator**

```java
package io.casehub.drafthouse.debate;

import org.junit.jupiter.api.Test;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

class PipelineOrchestratorTest {

    private PipelineOrchestrator orchestrator = new PipelineOrchestrator();

    @Test
    void dimension_start_sets_running() {
        var session = makeSession();
        orchestrator.onEvent(session,
                new ProgressLogParser.DimensionStart("coherence", "standard", 1));
        assertEquals(DimensionStatus.RUNNING, session.dimensions().get(0).status());
    }

    @Test
    void dimension_done_sets_done() {
        var session = makeSession();
        orchestrator.onEvent(session,
                new ProgressLogParser.DimensionStart("coherence", "standard", 1));
        orchestrator.onEvent(session,
                new ProgressLogParser.DimensionDone("coherence", 3, 4.20, 8));
        assertEquals(DimensionStatus.DONE, session.dimensions().get(0).status());
        assertEquals(4.20, session.dimensions().get(0).cost(), 0.001);
    }

    @Test
    void review_terminal_failed_sets_failed() {
        var session = makeSession();
        session.advanceDimension("coherence", DimensionStatus.RUNNING);
        orchestrator.onEvent(session,
                new ProgressLogParser.ReviewTerminal("FAILED"));
        // ReviewTerminal has no dimension — orchestrator sets the currently-running dimension to FAILED
        assertEquals(DimensionStatus.FAILED, session.dimensions().get(0).status());
    }

    @Test
    void all_round_1_complete_advances_to_checkpoint() {
        var session = makeSession("coherence", "structure");
        orchestrator.onEvent(session, new ProgressLogParser.DimensionStart("coherence", "standard", 1));
        orchestrator.onEvent(session, new ProgressLogParser.DimensionStart("structure", "standard", 1));
        orchestrator.onEvent(session, new ProgressLogParser.RoundEnd("coherence", 1, 1.0));
        assertEquals(PipelinePhase.ROUND_1, session.currentPhase());
        orchestrator.onEvent(session, new ProgressLogParser.RoundEnd("structure", 1, 1.0));
        assertEquals(PipelinePhase.HIL_CHECKPOINT_1, session.currentPhase());
    }

    @Test
    void all_done_advances_to_checkpoint_2() {
        var session = makeSession("coherence", "structure");
        session.advanceDimension("coherence", DimensionStatus.RUNNING);
        session.advanceDimension("structure", DimensionStatus.RUNNING);
        orchestrator.onEvent(session, new ProgressLogParser.DimensionDone("coherence", 3, 4.0, 5));
        assertEquals(PipelinePhase.ROUND_1, session.currentPhase());
        orchestrator.onEvent(session, new ProgressLogParser.DimensionDone("structure", 3, 3.0, 4));
        assertEquals(PipelinePhase.HIL_CHECKPOINT_2, session.currentPhase());
    }

    private PipelineSession makeSession(String... names) {
        if (names.length == 0) names = new String[]{"coherence"};
        var dims = java.util.Arrays.stream(names)
                .map(n -> new DimensionDescriptor(n, "/tmp/" + n, "standard"))
                .toList();
        return new PipelineSession("p1", "d1", dims, false, "/tmp/spec.md");
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest="PipelineWatcherTest,PipelineOrchestratorTest" -DfailIfNoTests=false`
Expected: compilation errors

- [ ] **Step 4: Implement PipelineWatcher**

```java
package io.casehub.drafthouse.debate;

import io.methvin.watcher.DirectoryChangeEvent;
import io.methvin.watcher.DirectoryWatcher;

import java.io.Closeable;
import java.io.IOException;
import java.io.RandomAccessFile;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.function.Consumer;
import java.util.logging.Logger;

public final class PipelineWatcher implements Closeable {
    private static final Logger LOG = Logger.getLogger(PipelineWatcher.class.getName());

    private final Path workspacePath;
    private final Consumer<ProgressLogParser.ProgressEvent> listener;
    private final AtomicBoolean stopped = new AtomicBoolean(false);
    private DirectoryWatcher directoryWatcher;
    private volatile long offset;

    public PipelineWatcher(Path workspacePath,
                           Consumer<ProgressLogParser.ProgressEvent> listener) {
        this.workspacePath = workspacePath;
        this.listener = listener;
    }

    public void start() throws IOException {
        Path logPath = workspacePath.resolve("progress.log");
        this.offset = Files.exists(logPath) ? Files.size(logPath) : 0;
        this.directoryWatcher = DirectoryWatcher.builder()
                .path(workspacePath)
                .listener(this::onFileEvent)
                .build();
        this.directoryWatcher.watchAsync();
    }

    public void stop() {
        if (stopped.compareAndSet(false, true) && directoryWatcher != null) {
            try { directoryWatcher.close(); }
            catch (IOException e) { LOG.warning("Failed to close watcher: " + e.getMessage()); }
        }
    }

    @Override
    public void close() { stop(); }

    private void onFileEvent(DirectoryChangeEvent event) {
        if (stopped.get()) return;
        if (!event.path().getFileName().toString().equals("progress.log")) return;
        tailProgressLog();
    }

    private void tailProgressLog() {
        Path logPath = workspacePath.resolve("progress.log");
        if (!Files.exists(logPath)) return;
        try {
            long fileSize = Files.size(logPath);
            if (fileSize <= offset) return;
            String newContent;
            try (var raf = new RandomAccessFile(logPath.toFile(), "r")) {
                raf.seek(offset);
                byte[] bytes = new byte[(int) (fileSize - offset)];
                raf.readFully(bytes);
                newContent = new String(bytes);
            }
            offset = fileSize;
            for (String line : newContent.split("\n")) {
                var parsed = ProgressLogParser.parse(line.trim());
                if (parsed != null) listener.accept(parsed);
            }
        } catch (IOException e) {
            LOG.warning("Failed to tail progress.log: " + e.getMessage());
        }
    }
}
```

- [ ] **Step 5: Implement PipelineOrchestrator**

```java
package io.casehub.drafthouse.debate;

import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class PipelineOrchestrator {

    public void onEvent(PipelineSession session,
                        ProgressLogParser.ProgressEvent event) {
        synchronized (session) {
            switch (event) {
                case ProgressLogParser.DimensionStart ds -> {
                    session.advanceDimension(ds.dimension(), DimensionStatus.RUNNING);
                    session.updateDimensionRound(ds.dimension(), 0);
                }
                case ProgressLogParser.RoundFindings rf -> {
                    session.updateDimensionRound(rf.dimension(), rf.roundNumber());
                    session.updateDimensionIssues(rf.dimension(), rf.byPriority());
                }
                case ProgressLogParser.RoundEnd re -> {
                    session.updateDimensionRound(re.dimension(), re.roundNumber());
                    session.updateDimensionCost(re.dimension(), re.cost());
                    checkRound1Complete(session);
                }
                case ProgressLogParser.DimensionDone dd -> {
                    session.advanceDimension(dd.dimension(), DimensionStatus.DONE);
                    session.updateDimensionCost(dd.dimension(), dd.cost());
                    checkAllDimensionsDone(session);
                }
                case ProgressLogParser.ReviewTerminal rt -> {
                    if ("DONE".equals(rt.finalState())) {
                        setRunningDimensionStatus(session, DimensionStatus.DONE);
                    } else {
                        setRunningDimensionStatus(session, DimensionStatus.FAILED);
                    }
                    checkAllDimensionsDone(session);
                }
                case ProgressLogParser.RoundComplete rc ->
                        session.updateDimensionRound(findRunningDimension(session), rc.round());
                default -> {} // AgentStart, AgentStatus, AgentComplete, IssuesRaised — no pipeline-level action
            }
        }
    }

    private void checkRound1Complete(PipelineSession session) {
        if (session.currentPhase() != PipelinePhase.ROUND_1) return;
        boolean allPastRound1 = session.dimensions().stream()
                .filter(d -> !d.name().equals("crosscutting"))
                .allMatch(d -> d.currentRound() >= 1);
        if (allPastRound1) session.setPhase(PipelinePhase.HIL_CHECKPOINT_1);
    }

    private void checkAllDimensionsDone(PipelineSession session) {
        if (session.currentPhase() == PipelinePhase.COMPLETE) return;
        boolean allTerminal = session.dimensions().stream()
                .filter(d -> !d.name().equals("crosscutting"))
                .allMatch(d -> d.status() == DimensionStatus.DONE
                        || d.status() == DimensionStatus.KILLED
                        || d.status() == DimensionStatus.FAILED);
        if (allTerminal && session.currentPhase().ordinal() < PipelinePhase.HIL_CHECKPOINT_2.ordinal()) {
            session.setPhase(PipelinePhase.HIL_CHECKPOINT_2);
        }
    }

    private void setRunningDimensionStatus(PipelineSession session, DimensionStatus status) {
        session.dimensions().stream()
                .filter(d -> d.status() == DimensionStatus.RUNNING)
                .findFirst()
                .ifPresent(d -> session.advanceDimension(d.name(), status));
    }

    private String findRunningDimension(PipelineSession session) {
        return session.dimensions().stream()
                .filter(d -> d.status() == DimensionStatus.RUNNING)
                .findFirst()
                .map(DimensionDescriptor::name)
                .orElse("unknown");
    }
}
```

- [ ] **Step 6: Implement PipelineSessionRegistry**

```java
package io.casehub.drafthouse.debate;

import jakarta.enterprise.context.ApplicationScoped;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class PipelineSessionRegistry {
    private final ConcurrentHashMap<String, PipelineSession> sessions = new ConcurrentHashMap<>();

    public void create(PipelineSession session) {
        sessions.put(session.pipelineId(), session);
    }

    public PipelineSession get(String pipelineId) {
        return sessions.get(pipelineId);
    }

    public void remove(String pipelineId) {
        sessions.remove(pipelineId);
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest="PipelineWatcherTest,PipelineOrchestratorTest"`
Expected: all tests PASS

- [ ] **Step 8: Commit**

```bash
git add server/runtime/src/main/java/io/casehub/drafthouse/debate/PipelineWatcher.java server/runtime/src/main/java/io/casehub/drafthouse/debate/PipelineOrchestrator.java server/runtime/src/main/java/io/casehub/drafthouse/debate/PipelineSessionRegistry.java server/runtime/src/test/java/io/casehub/drafthouse/debate/PipelineWatcherTest.java server/runtime/src/test/java/io/casehub/drafthouse/debate/PipelineOrchestratorTest.java
git commit -m "feat(#72): PipelineWatcher, PipelineOrchestrator, PipelineSessionRegistry

Refs #72"
```

---

### Task 4: PipelineMcpTools — start_pipeline, update_pipeline, load_decisions

**Files:**
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/PipelineMcpTools.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/PipelineMcpToolsTest.java`

**Interfaces:**
- Consumes: `PipelineSessionRegistry` (Task 3), `PipelineOrchestrator` (Task 3), `PipelineSession` (Task 2), `PipelineDecisionParser` (Task 2), `PipelineWatcher` (Task 3), `WebSocketEventBus.pushMetadata(UUID, String, Object)` (existing)
- Produces: Three `@Tool` methods. `start_pipeline(debateSessionId, dimensions, ordered, specPath) → String JSON`, `update_pipeline(pipelineId, action, dimension) → String JSON`, `load_decisions(pipelineId, decisionsPath) → String JSON`

- [ ] **Step 1: Write failing integration test**

```java
package io.casehub.drafthouse;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class PipelineMcpToolsTest {

    @Inject PipelineMcpTools tools;

    @Test
    void start_pipeline_creates_session() {
        String result = tools.startPipeline("test-debate",
                "[{\"name\":\"coherence\",\"workspacePath\":\"/tmp/test-ws\",\"degree\":\"light\"}]",
                false, "/tmp/spec.md");
        assertNotNull(result);
        assertTrue(result.contains("pipelineId"));
        assertTrue(result.contains("coherence"));
    }

    @Test
    void update_pipeline_unknown_id_returns_error() {
        String result = tools.updatePipeline("nonexistent", "pipeline_complete", null);
        assertTrue(result.contains("error"));
    }

    @Test
    void update_pipeline_idempotent() {
        String r1 = tools.startPipeline("test-debate2",
                "[{\"name\":\"structure\",\"workspacePath\":\"/tmp/test-ws2\",\"degree\":\"light\"}]",
                false, "/tmp/spec.md");
        // Extract pipelineId from JSON
        String pipelineId = extractPipelineId(r1);
        String r2 = tools.updatePipeline(pipelineId, "checkpoint_reached", null);
        String r3 = tools.updatePipeline(pipelineId, "checkpoint_reached", null);
        assertEquals(r2, r3);
        tools.updatePipeline(pipelineId, "pipeline_complete", null);
    }

    @Test
    void load_decisions_parses_file() throws Exception {
        String r1 = tools.startPipeline("test-debate3",
                "[{\"name\":\"coherence\",\"workspacePath\":\"/tmp/test-ws3\",\"degree\":\"light\"}]",
                false, "/tmp/spec.md");
        String pipelineId = extractPipelineId(r1);
        // Use the actual decisions.md from our workspace
        String result = tools.loadDecisions(pipelineId,
                System.getProperty("user.home") + "/claude/public/casehub/drafthouse/specs/issue-72-review-pipeline-orch/decisions.md");
        assertTrue(result.contains("decisions"));
        assertTrue(result.contains("D1"));
        tools.updatePipeline(pipelineId, "pipeline_complete", null);
    }

    private String extractPipelineId(String json) {
        int idx = json.indexOf("\"pipelineId\":\"") + 14;
        return json.substring(idx, json.indexOf("\"", idx));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=PipelineMcpToolsTest -DfailIfNoTests=false`
Expected: compilation error — PipelineMcpTools doesn't exist

- [ ] **Step 3: Implement PipelineMcpTools**

```java
package io.casehub.drafthouse;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.drafthouse.debate.*;
import io.quarkiverse.mcp.server.Tool;
import io.quarkiverse.mcp.server.ToolArg;
import jakarta.inject.Inject;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.logging.Logger;

public class PipelineMcpTools {
    private static final Logger LOG = Logger.getLogger(PipelineMcpTools.class.getName());
    private final ConcurrentHashMap<String, List<PipelineWatcher>> activeWatchers = new ConcurrentHashMap<>();
    private final ObjectMapper mapper = new ObjectMapper();

    @Inject PipelineSessionRegistry registry;
    @Inject PipelineOrchestrator orchestrator;
    @Inject WebSocketEventBus eventBus;
    @Inject DebateSessionRegistry debateRegistry;

    @Tool(name = "start_pipeline",
          description = "Create a review pipeline to visualize multi-dimension reviews. "
                  + "Links to a debate session for WebSocket routing.")
    public String startPipeline(
            @ToolArg(description = "Debate session ID to link pipeline events to") String debateSessionId,
            @ToolArg(description = "Dimensions as JSON array: [{name, workspacePath, degree}]") String dimensions,
            @ToolArg(description = "Sequential (true) or parallel (false)") boolean ordered,
            @ToolArg(description = "Path to the spec being reviewed") String specPath) {
        try {
            var dimsNode = mapper.readTree(dimensions);
            var dimList = new ArrayList<DimensionDescriptor>();
            for (var node : dimsNode) {
                dimList.add(new DimensionDescriptor(
                        node.path("name").asText(),
                        node.path("workspacePath").asText(),
                        node.path("degree").asText()));
            }

            String pipelineId = UUID.randomUUID().toString();
            var session = new PipelineSession(pipelineId, debateSessionId, dimList, ordered, specPath);
            registry.create(session);

            var watchers = new ArrayList<PipelineWatcher>();
            for (var dim : dimList) {
                Path wsPath = Path.of(dim.workspacePath());
                if (!Files.isDirectory(wsPath)) continue;
                var watcher = new PipelineWatcher(wsPath, event -> {
                    orchestrator.onEvent(session, event);
                    pushPipelineProgress(session);
                });
                try {
                    watcher.start();
                    watchers.add(watcher);
                } catch (Exception e) {
                    LOG.warning("Failed to start watcher for " + dim.name() + ": " + e.getMessage());
                }
            }
            activeWatchers.put(pipelineId, watchers);

            pushPipelineProgress(session);
            return mapper.writeValueAsString(session.toSnapshot());
        } catch (Exception e) {
            return "{\"error\": \"" + e.getMessage().replace("\"", "'") + "\"}";
        }
    }

    @Tool(name = "update_pipeline",
          description = "Update pipeline state — HIL checkpoint decisions, dimension status changes.")
    public String updatePipeline(
            @ToolArg(description = "Pipeline ID") String pipelineId,
            @ToolArg(description = "Action: checkpoint_reached, dimension_refused, dimension_accepted, crosscutting_started, pipeline_complete") String action,
            @ToolArg(description = "Dimension name (for dimension_refused, dimension_accepted)") String dimension) {
        var session = registry.get(pipelineId);
        if (session == null) return "{\"error\": \"Pipeline not found: " + pipelineId + "\"}";

        synchronized (session) {
            switch (action) {
                case "checkpoint_reached" -> {
                    if (session.checkpointStatus() == CheckpointStatus.PENDING) break;
                    session.setCheckpoint(CheckpointStatus.PENDING);
                }
                case "dimension_refused" -> {
                    if (dimension == null) return "{\"error\": \"dimension required for dimension_refused\"}";
                    session.advanceDimension(dimension, DimensionStatus.KILLED);
                    stopWatcherForDimension(pipelineId, dimension);
                    checkCheckpointResolution(session);
                }
                case "dimension_accepted" -> {
                    checkCheckpointResolution(session);
                }
                case "crosscutting_started" -> session.setPhase(PipelinePhase.CROSS_CUTTING);
                case "pipeline_complete" -> {
                    session.setPhase(PipelinePhase.COMPLETE);
                    stopAllWatchers(pipelineId);
                    registry.remove(pipelineId);
                }
                default -> { return "{\"error\": \"Unknown action: " + action + "\"}"; }
            }
        }

        pushPipelineProgress(session);
        try { return mapper.writeValueAsString(session.toSnapshot()); }
        catch (Exception e) { return "{\"error\": \"" + e.getMessage() + "\"}"; }
    }

    @Tool(name = "load_decisions",
          description = "Load brainstorming decisions from a decisions.md file into the pipeline.")
    public String loadDecisions(
            @ToolArg(description = "Pipeline ID") String pipelineId,
            @ToolArg(description = "Path to decisions.md") String decisionsPath) {
        var session = registry.get(pipelineId);
        if (session == null) return "{\"error\": \"Pipeline not found: " + pipelineId + "\"}";

        try {
            String content = Files.readString(Path.of(decisionsPath));
            var decisions = PipelineDecisionParser.parse(content);
            session.setDecisions(decisions);

            var debateSession = debateRegistry.activeSessions().stream()
                    .filter(s -> s.debateSessionId().equals(session.debateSessionId()))
                    .findFirst().orElse(null);
            if (debateSession != null) {
                var payload = new LinkedHashMap<String, Object>();
                payload.put("pipelineId", pipelineId);
                payload.put("decisions", decisions.stream().map(d -> {
                    var m = new LinkedHashMap<String, Object>();
                    m.put("id", d.id());
                    m.put("title", d.title());
                    m.put("choice", d.choice());
                    m.put("alternatives", d.alternatives());
                    m.put("rationale", d.rationale());
                    m.put("tradeoffs", d.tradeoffs());
                    m.put("status", d.status());
                    m.put("exploration", d.explorationDepth());
                    return m;
                }).toList());
                eventBus.pushMetadata(debateSession.channelId(), "pipeline-decisions", payload);
            }
            return mapper.writeValueAsString(Map.of(
                    "pipelineId", pipelineId,
                    "decisionsLoaded", decisions.size()));
        } catch (Exception e) {
            return "{\"error\": \"" + e.getMessage().replace("\"", "'") + "\"}";
        }
    }

    private void pushPipelineProgress(PipelineSession session) {
        debateRegistry.activeSessions().stream()
                .filter(s -> s.debateSessionId().equals(session.debateSessionId()))
                .findFirst()
                .ifPresent(ds -> eventBus.pushMetadata(
                        ds.channelId(), "pipeline-progress", session.toSnapshot()));
    }

    private void checkCheckpointResolution(PipelineSession session) {
        if (session.checkpointStatus() != CheckpointStatus.PENDING) return;
        session.setCheckpoint(CheckpointStatus.RESOLVED);
        if (session.currentPhase() == PipelinePhase.HIL_CHECKPOINT_1) {
            session.setPhase(PipelinePhase.REMAINING_ROUNDS);
        }
    }

    private void stopWatcherForDimension(String pipelineId, String dimension) {
        var session = registry.get(pipelineId);
        if (session == null) return;
        var watchers = activeWatchers.get(pipelineId);
        if (watchers == null) return;
        int idx = -1;
        var dims = session.dimensions();
        for (int i = 0; i < dims.size(); i++) {
            if (dims.get(i).name().equals(dimension)) { idx = i; break; }
        }
        if (idx >= 0 && idx < watchers.size()) watchers.get(idx).stop();
    }

    private void stopAllWatchers(String pipelineId) {
        var watchers = activeWatchers.remove(pipelineId);
        if (watchers != null) watchers.forEach(PipelineWatcher::stop);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=PipelineMcpToolsTest`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git add server/runtime/src/main/java/io/casehub/drafthouse/PipelineMcpTools.java server/runtime/src/test/java/io/casehub/drafthouse/PipelineMcpToolsTest.java
git commit -m "feat(#72): PipelineMcpTools — start_pipeline, update_pipeline, load_decisions

Refs #72"
```

---

### Task 5: review-pipeline browser panel

**Files:**
- Create: `blocks-ui repo: components/document-workbench/src/review-pipeline.ts`
- Modify: `blocks-ui repo: components/document-workbench/src/types.ts` (new interfaces)
- Modify: `blocks-ui repo: components/document-workbench/src/index.ts` (export)
- Modify: `server/runtime/src/main/webui/src/index.ts` (register panel, topbar, wiring)

**Interfaces:**
- Consumes: `pipeline-progress` and `pipeline-decisions` WebSocket events (Task 4)
- Produces: `<review-pipeline>` custom element with `configure(props)`. Dispatches `point-selected` events on finding click.

**Note:** This task modifies code in two repos — blocks-ui and drafthouse. The blocks-ui changes must be built and installed to `~/.m2` before the drafthouse webui can consume them. Run `yarn build && /opt/homebrew/bin/mvn install` in the blocks-ui repo after making changes there.

- [ ] **Step 1: Add TypeScript interfaces to types.ts**

In `blocks-ui/components/document-workbench/src/types.ts`, add:

```typescript
export interface PipelineProgressPayload {
  pipelineId: string;
  phase: string;
  checkpointStatus: string;
  ordered: boolean;
  dimensions: PipelineDimension[];
}

export interface PipelineDimension {
  name: string;
  status: string;
  currentRound: number;
  totalRounds: number;
  degree: string;
  issuesByPriority: Record<string, number>;
  cost: number;
  elapsed: number;
  findingsCount: number;
}

export interface PipelineDecisionPayload {
  pipelineId: string;
  decisions: PipelineDecisionData[];
}

export interface PipelineDecisionData {
  id: string;
  title: string;
  choice: string;
  alternatives: string[];
  rationale: string;
  tradeoffs: string;
  status: string;
  exploration: string;
}
```

- [ ] **Step 2: Create the review-pipeline panel**

Create `blocks-ui/components/document-workbench/src/review-pipeline.ts`:

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { customElement, state } from 'lit/decorators.js';
import { onPagesEvent } from '@casehubio/pages-component';
import type {
  PipelineProgressPayload,
  PipelineDimension,
  PipelineDecisionPayload,
  PipelineDecisionData,
} from './types.js';

@customElement('review-pipeline')
export class ReviewPipeline extends LitElement {
  @state() private _pipeline: PipelineProgressPayload | null = null;
  @state() private _decisions: PipelineDecisionData[] = [];
  @state() private _priorityFilter: string = 'all';

  private _cleanups: (() => void)[] = [];

  configure(_props: Record<string, unknown>): void {}

  override connectedCallback(): void {
    super.connectedCallback();
    this._cleanups.push(
      onPagesEvent<PipelineProgressPayload>(document, 'pipeline-progress', (p) => {
        this._pipeline = p;
      }),
      onPagesEvent<PipelineDecisionPayload>(document, 'pipeline-decisions', (p) => {
        this._decisions = p.decisions;
      }),
      onPagesEvent(document, 'reconnected', () => {
        // State will be re-pushed by server on reconnect
      }),
    );
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    this._cleanups.forEach(fn => fn());
    this._cleanups = [];
  }

  private _statusBadge(status: string) {
    const map: Record<string, { cls: string; label: string }> = {
      PENDING: { cls: 'pending', label: 'pending' },
      RUNNING: { cls: 'running', label: 'running' },
      DONE: { cls: 'done', label: 'done' },
      KILLED: { cls: 'killed', label: 'refused' },
      FAILED: { cls: 'failed', label: 'failed' },
    };
    const s = map[status] ?? { cls: 'pending', label: status.toLowerCase() };
    return html`<span class="badge ${s.cls}">${s.label}</span>`;
  }

  private _decisionBadge(status: string) {
    const cls = status === 'confirmed' ? 'confirmed'
      : status === 'challenged' ? 'challenged'
      : status === 'rejected' ? 'rejected'
      : status === 'revised' ? 'revised'
      : 'captured';
    return html`<span class="dbadge ${cls}">${status}</span>`;
  }

  private _phaseClass(phase: string, current: string) {
    const phases = ['ROUND_1', 'HIL_CHECKPOINT_1', 'REMAINING_ROUNDS', 'HIL_CHECKPOINT_2', 'CROSS_CUTTING', 'COMPLETE'];
    const ci = phases.indexOf(current);
    const pi = phases.indexOf(phase);
    if (pi < ci) return 'phase-done';
    if (pi === ci) return 'phase-active';
    return 'phase-pending';
  }

  private _phaseLabel(phase: string) {
    const labels: Record<string, string> = {
      ROUND_1: 'Round 1',
      HIL_CHECKPOINT_1: 'HIL',
      REMAINING_ROUNDS: 'Rounds 2+',
      HIL_CHECKPOINT_2: 'HIL',
      CROSS_CUTTING: 'Cross-cutting',
      COMPLETE: 'Complete',
    };
    return labels[phase] ?? phase;
  }

  private _formatElapsed(seconds: number): string {
    const m = Math.floor(seconds / 60);
    const s = seconds % 60;
    return m > 0 ? `${m}m ${s}s` : `${s}s`;
  }

  private _renderDecisions() {
    if (this._decisions.length === 0) return nothing;
    const allTerminal = this._decisions.every(d =>
      d.status === 'confirmed' || d.status === 'rejected');
    return html`
      <details ?open=${!allTerminal}>
        <summary class="section-header">Decisions (${this._decisions.length})</summary>
        <div class="decisions-list">
          ${this._decisions.map(d => html`
            <details class="decision-card">
              <summary>${d.id}: ${d.title} ${this._decisionBadge(d.status)}</summary>
              <div class="decision-detail">
                <div><strong>Choice:</strong> ${d.choice}</div>
                ${d.alternatives.length > 0 ? html`
                  <div><strong>Alternatives:</strong>
                    <ul>${d.alternatives.map(a => html`<li>${a}</li>`)}</ul>
                  </div>` : nothing}
                ${d.rationale ? html`<div><strong>Rationale:</strong> ${d.rationale}</div>` : nothing}
                ${d.tradeoffs ? html`<div><strong>Trade-offs:</strong> ${d.tradeoffs}</div>` : nothing}
              </div>
            </details>
          `)}
        </div>
      </details>
    `;
  }

  private _renderPhaseHeader() {
    if (!this._pipeline) return nothing;
    const phases = ['ROUND_1', 'HIL_CHECKPOINT_1', 'REMAINING_ROUNDS', 'HIL_CHECKPOINT_2', 'CROSS_CUTTING'];
    return html`
      <div class="phase-header">
        ${phases.map((p, i) => html`
          ${i > 0 ? html`<span class="phase-arrow">→</span>` : nothing}
          <span class="phase-segment ${this._phaseClass(p, this._pipeline!.phase)}">
            ${this._phaseLabel(p)}
            ${(p === 'HIL_CHECKPOINT_1' || p === 'HIL_CHECKPOINT_2')
              && this._pipeline!.checkpointStatus === 'PENDING'
              && this._pipeline!.phase === p
              ? html`<span class="checkpoint-badge">⚠</span>` : nothing}
          </span>
        `)}
      </div>
    `;
  }

  private _renderDimensionCards() {
    if (!this._pipeline) return nothing;
    return html`
      <div class="dimension-cards">
        ${this._pipeline.dimensions.map(dim => html`
          <div class="dim-card ${dim.status.toLowerCase()}">
            <div class="dim-header">
              ${this._statusBadge(dim.status)}
              <span class="dim-name">${dim.name}</span>
            </div>
            ${dim.status === 'RUNNING' || dim.status === 'DONE' ? html`
              <div class="dim-progress">
                <div class="progress-bar">
                  <div class="progress-fill" style="width:${dim.totalRounds > 0
                    ? (dim.currentRound / dim.totalRounds * 100) : 0}%"></div>
                </div>
                <span class="round-label">round ${dim.currentRound}${dim.totalRounds > 0
                  ? `/${dim.totalRounds}` : ''}</span>
              </div>
              <div class="dim-stats">
                ${this._renderIssuePills(dim)}
                <span class="cost">$${dim.cost.toFixed(2)}</span>
                ${dim.status === 'RUNNING' ? html`<span class="elapsed">${this._formatElapsed(dim.elapsed)}</span>` : nothing}
              </div>
            ` : nothing}
            ${dim.status === 'KILLED' ? html`<span class="dim-note">refused at checkpoint</span>` : nothing}
            ${dim.status === 'FAILED' ? html`<span class="dim-note">review failed</span>` : nothing}
          </div>
        `)}
      </div>
    `;
  }

  private _renderIssuePills(dim: PipelineDimension) {
    const priorities = ['HIGH', 'MEDIUM', 'LOW'];
    return html`
      <span class="issue-pills">
        ${priorities.map(p => {
          const count = dim.issuesByPriority[p] ?? 0;
          return count > 0 ? html`<span class="pill ${p.toLowerCase()}">${count} ${p[0]}</span>` : nothing;
        })}
      </span>
    `;
  }

  static override styles = css`
    :host { display: block; padding: 8px 12px; font-size: 12px; color: var(--pages-neutral-12, #e5e7eb); overflow-y: auto; }
    .empty { color: var(--pages-neutral-8, #9ca3af); font-style: italic; padding: 24px; text-align: center; }
    .section-header { cursor: pointer; font-weight: 600; font-size: 13px; margin-bottom: 6px; }
    .decisions-list { display: flex; flex-direction: column; gap: 4px; margin-bottom: 12px; }
    .decision-card { border: 1px solid var(--pages-neutral-6, #4b5563); border-radius: 4px; padding: 4px 8px; }
    .decision-card summary { cursor: pointer; display: flex; align-items: center; gap: 6px; }
    .decision-detail { padding: 6px 0 2px; font-size: 11px; line-height: 1.5; }
    .decision-detail ul { margin: 2px 0; padding-left: 16px; }
    .dbadge { font-size: 10px; padding: 1px 5px; border-radius: 3px; font-weight: 500; }
    .dbadge.captured { background: var(--pages-neutral-6); color: var(--pages-neutral-11); }
    .dbadge.confirmed { background: var(--pages-success-3, #064e3b); color: var(--pages-success-11, #6ee7b7); }
    .dbadge.challenged { background: var(--pages-warning-3, #78350f); color: var(--pages-warning-11, #fcd34d); }
    .dbadge.rejected { background: var(--pages-error-3, #7f1d1d); color: var(--pages-error-11, #fca5a5); }
    .dbadge.revised { background: var(--pages-accent-3, #312e81); color: var(--pages-accent-11, #a5b4fc); }
    .phase-header { display: flex; align-items: center; gap: 4px; margin: 8px 0; flex-wrap: wrap; }
    .phase-segment { padding: 2px 8px; border-radius: 3px; font-size: 11px; }
    .phase-arrow { color: var(--pages-neutral-7); font-size: 10px; }
    .phase-done { background: var(--pages-success-3); color: var(--pages-success-11); }
    .phase-active { background: var(--pages-accent-3); color: var(--pages-accent-11); font-weight: 600; }
    .phase-pending { background: var(--pages-neutral-3, #1f2937); color: var(--pages-neutral-8); }
    .checkpoint-badge { color: var(--pages-warning-9, #f59e0b); margin-left: 3px; }
    .dimension-cards { display: flex; flex-direction: column; gap: 6px; margin: 8px 0; }
    .dim-card { border: 1px solid var(--pages-neutral-6); border-radius: 4px; padding: 6px 8px; }
    .dim-card.killed { opacity: 0.6; }
    .dim-card.failed { border-color: var(--pages-error-7, #b91c1c); }
    .dim-header { display: flex; align-items: center; gap: 6px; }
    .dim-name { font-weight: 500; text-transform: capitalize; }
    .badge { font-size: 10px; padding: 1px 5px; border-radius: 3px; }
    .badge.pending { background: var(--pages-neutral-4); color: var(--pages-neutral-9); }
    .badge.running { background: var(--pages-accent-3); color: var(--pages-accent-11); }
    .badge.done { background: var(--pages-success-3); color: var(--pages-success-11); }
    .badge.killed { background: var(--pages-error-3); color: var(--pages-error-11); text-decoration: line-through; }
    .badge.failed { background: var(--pages-error-3); color: var(--pages-error-11); }
    .dim-progress { display: flex; align-items: center; gap: 6px; margin-top: 4px; }
    .progress-bar { flex: 1; height: 3px; background: var(--pages-neutral-4); border-radius: 2px; overflow: hidden; }
    .progress-fill { height: 100%; background: var(--pages-accent-9, #6366f1); transition: width 0.3s; }
    .round-label { font-size: 10px; color: var(--pages-neutral-8); white-space: nowrap; }
    .dim-stats { display: flex; align-items: center; gap: 8px; margin-top: 4px; font-size: 11px; }
    .issue-pills { display: flex; gap: 3px; }
    .pill { font-size: 10px; padding: 0 4px; border-radius: 2px; }
    .pill.high { background: var(--pages-error-3); color: var(--pages-error-11); }
    .pill.medium { background: var(--pages-warning-3); color: var(--pages-warning-11); }
    .pill.low { background: var(--pages-neutral-4); color: var(--pages-neutral-9); }
    .cost { color: var(--pages-neutral-8); }
    .elapsed { color: var(--pages-neutral-8); font-variant-numeric: tabular-nums; }
    .dim-note { font-size: 10px; color: var(--pages-neutral-8); font-style: italic; margin-top: 2px; }
  `;

  override render() {
    if (!this._pipeline && this._decisions.length === 0) {
      return html`<div class="empty">No active review pipeline</div>`;
    }
    return html`
      ${this._renderDecisions()}
      ${this._renderPhaseHeader()}
      ${this._renderDimensionCards()}
    `;
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'review-pipeline': ReviewPipeline;
  }
}
```

- [ ] **Step 3: Export from index.ts**

In `blocks-ui/components/document-workbench/src/index.ts`, add:
```typescript
export { ReviewPipeline } from './review-pipeline.js';
```

- [ ] **Step 4: Build and install blocks-ui**

Run in blocks-ui repo:
```bash
yarn build
/opt/homebrew/bin/mvn -f npm-packages/pom.xml install
```

- [ ] **Step 5: Wire into drafthouse workbench**

In `server/runtime/src/main/webui/src/index.ts`:

Add panel registration after existing registrations:
```typescript
registerPanel("review-pipeline", "review-pipeline");
```

Add to topbar HTML (after the review button):
```html
<button id="btn-pipeline" title="Toggle pipeline panel">⚙ Pipeline</button>
```

Add to the right-side vertical split (after review):
```typescript
withId("pipeline", hostPanel("review-pipeline", {})),
```

Update split ratios to include the new panel:
```typescript
], { ratio: [45, 25, 30] }),  // → becomes [35, 30, 15, 20]
```

Add toggle wiring:
```typescript
let pipelineVisible = false;
```

Add in `updatePanelVisibility()`:
```typescript
app.dispatchEvent(new CustomEvent("pages-dock-toggle", {
  bubbles: true, detail: { panelId: "pipeline", visible: pipelineVisible },
}));
document.getElementById("btn-pipeline")?.classList.toggle("active", pipelineVisible);
```

Add click handler:
```typescript
document.getElementById("btn-pipeline")?.addEventListener("click", () => {
  pipelineVisible = !pipelineVisible;
  updatePanelVisibility();
});
```

Add auto-show on pipeline event:
```typescript
// In the existing pages-event listener:
if (topic === "pipeline-progress" && !pipelineVisible) {
  pipelineVisible = true;
  updatePanelVisibility();
}
```

Initially hide the pipeline panel (start hidden):
```typescript
// After loadSite, before existing panel visibility calls:
updatePanelVisibility();  // hides pipeline since pipelineVisible = false
```

- [ ] **Step 6: Build drafthouse and verify**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml package -DskipTests`
Expected: build succeeds, webui bundles with the new panel

- [ ] **Step 7: Commit in both repos**

blocks-ui repo:
```bash
git add components/document-workbench/src/review-pipeline.ts components/document-workbench/src/types.ts components/document-workbench/src/index.ts
git commit -m "feat(casehubio/drafthouse#72): add review-pipeline panel

Refs casehubio/drafthouse#72"
```

drafthouse repo:
```bash
git add server/runtime/src/main/webui/src/index.ts
git commit -m "feat(#72): wire review-pipeline panel into workbench layout

Refs #72"
```

---

### Task 6: E2E tests and workspace-status enhancement

**Files:**
- Create: `server/runtime/src/test/java/io/casehub/drafthouse/e2e/ReviewPipelineE2ETest.java`
- Create: `server/runtime/src/test/resources/fixtures/pipeline-progress.json`
- Modify: `blocks-ui repo: components/document-workbench/src/workspace-status.ts`

**Interfaces:**
- Consumes: `pipeline-progress` and `pipeline-decisions` WebSocket events (Task 4), `<review-pipeline>` panel (Task 5)
- Produces: E2E test coverage, workspace-status pipeline summary

- [ ] **Step 1: Create pipeline-progress fixture**

Create `server/runtime/src/test/resources/fixtures/pipeline-progress.json`:
```json
{
  "pipelineId": "test-pipeline-1",
  "phase": "ROUND_1",
  "checkpointStatus": "NONE",
  "ordered": false,
  "dimensions": [
    {
      "name": "coherence",
      "status": "RUNNING",
      "currentRound": 1,
      "totalRounds": 3,
      "degree": "standard",
      "issuesByPriority": {"HIGH": 2, "MEDIUM": 1},
      "cost": 1.50,
      "elapsed": 90,
      "findingsCount": 3
    },
    {
      "name": "structure",
      "status": "PENDING",
      "currentRound": 0,
      "totalRounds": 0,
      "degree": "standard",
      "issuesByPriority": {},
      "cost": 0,
      "elapsed": 0,
      "findingsCount": 0
    }
  ]
}
```

- [ ] **Step 2: Write E2E test for panel visibility and rendering**

```java
package io.casehub.drafthouse.e2e;

import com.microsoft.playwright.*;
import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.*;

import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;

@QuarkusTest
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class ReviewPipelineE2ETest {

    static Playwright pw;
    static Browser browser;
    BrowserContext context;
    Page page;

    @BeforeAll
    static void launchBrowser() {
        pw = Playwright.create();
        browser = pw.chromium().launch(new BrowserType.LaunchOptions().setHeadless(true));
    }

    @AfterAll
    static void closeBrowser() {
        if (browser != null) browser.close();
        if (pw != null) pw.close();
    }

    @BeforeEach
    void setUp() {
        context = browser.newContext();
        page = context.newPage();
    }

    @AfterEach
    void tearDown() {
        if (context != null) context.close();
    }

    @Test
    @Order(1)
    void pipeline_panel_hidden_by_default() {
        page.navigate("http://localhost:9001/");
        page.waitForSelector("document-diff");
        // Pipeline panel should exist but be hidden
        var pipelineBtn = page.locator("#btn-pipeline");
        assertThat(pipelineBtn).isVisible();
        assertThat(pipelineBtn).not().hasClass("active");
    }

    @Test
    @Order(2)
    void pipeline_toggle_shows_panel() {
        page.navigate("http://localhost:9001/");
        page.waitForSelector("document-diff");
        page.click("#btn-pipeline");
        var pipelineBtn = page.locator("#btn-pipeline");
        assertThat(pipelineBtn).hasClass("active");
    }
}
```

- [ ] **Step 3: Update workspace-status with pipeline summary**

In `blocks-ui/components/document-workbench/src/workspace-status.ts`, add a handler for `pipeline-progress` in `connectedCallback`:

```typescript
onPagesEvent<PipelineProgressPayload>(document, 'pipeline-progress', (p) => {
  this._handlePipelineProgress(p);
}),
```

Add the handler method:
```typescript
private _handlePipelineProgress(p: PipelineProgressPayload): void {
  this._visible = true;
  this._terminal = p.phase === 'COMPLETE';
  const running = p.dimensions.filter(d => d.status === 'RUNNING').length;
  const total = p.dimensions.length;
  if (p.checkpointStatus === 'PENDING') {
    this._text = `Pipeline: checkpoint pending`;
  } else if (this._terminal) {
    this._text = `Pipeline: complete`;
  } else {
    this._text = `Pipeline: ${running}/${total} dimensions running`;
  }
  this._stopTimer();
}
```

Add the import for `PipelineProgressPayload` from `./types.js`.

- [ ] **Step 4: Build blocks-ui and install**

```bash
# In blocks-ui repo
yarn build
/opt/homebrew/bin/mvn -f npm-packages/pom.xml install
```

- [ ] **Step 5: Run E2E tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=ReviewPipelineE2ETest`
Expected: all tests PASS

- [ ] **Step 6: Commit in both repos**

blocks-ui:
```bash
git add components/document-workbench/src/workspace-status.ts
git commit -m "feat(casehubio/drafthouse#72): workspace-status pipeline summary

Refs casehubio/drafthouse#72"
```

drafthouse:
```bash
git add server/runtime/src/test/java/io/casehub/drafthouse/e2e/ReviewPipelineE2ETest.java server/runtime/src/test/resources/fixtures/pipeline-progress.json
git commit -m "feat(#72): E2E tests for review-pipeline panel

Refs #72"
```

---

### Task 7: Full integration test — build, run, verify

**Files:**
- No new files — verification task

**Interfaces:**
- Consumes: everything from Tasks 1-6

- [ ] **Step 1: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime`
Expected: all existing tests PASS, all new tests PASS

- [ ] **Step 2: Run the app and verify in browser**

```bash
java -jar server/runtime/target/drafthouse-server-runner.jar
```

Open `http://localhost:9001/` — verify:
- Pipeline button visible in topbar
- Panel hidden by default
- Click toggle shows empty state "No active review pipeline"
- No console errors

- [ ] **Step 3: Commit any fixes**

If any adjustments were needed, commit them with:
```bash
git commit -m "fix(#72): address integration test findings

Refs #72"
```
