# Channel-based HIL Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #100 — Channel-based HIL — concurrent human participation in adversarial debates
**Issue group:** #100

**Goal:** Enable concurrent human participation in adversarial debates via REST endpoints,
with actions flowing into the Qhorus channel and written to workspace `decisions/` files.

**Architecture:** New `HumanActionResource` provides 5 REST endpoints for human actions
(comment, raise, override, prioritise, batch). Each dispatches to the Qhorus channel with
`ActorType.HUMAN` and writes to `decisions/human-round-{n}.md`. The projection folds human
entries like agent entries. UI panels gain interactive affordances and human-specific styling.

**Tech Stack:** Quarkus 3.34.3, casehub-qhorus 0.2-SNAPSHOT, casehub-blocks 0.2-SNAPSHOT,
LitElement (TypeScript), REST-assured (@QuarkusTest)

## Global Constraints

- All channel messages use `DebateProtocol.META_SENTINEL` encoding (PP-20260608-d94c7d)
- `ChannelProjection.apply()` must never throw (PP-20260610-a47ef5)
- MCP `@Tool` methods return error strings, never exceptions (PP-20260604-6e8d5d)
- Use `message.actorType()` for actor classification, not sender strings (PP-20260607-508f7b)
- Session lifecycle methods must deregister all Qhorus instances (PP-20260608-21c69f)
- Cross-module change: `ConversationFold.reprioritisePoint()` added to casehub-blocks SNAPSHOT
- IntelliJ MCP required for all Java/TypeScript editing — `project_path=/Users/mdproctor/claude/casehub/drafthouse`

---

### Task 1: ConversationFold.reprioritisePoint() (casehub-blocks)

**Cross-repo:** `/Users/mdproctor/claude/casehub/blocks/`

**Files:**
- Modify: `src/main/java/io/casehub/blocks/conversation/ConversationFold.java`
- Test: `src/test/java/io/casehub/blocks/conversation/ConversationFoldTest.java`

**Interfaces:**
- Consumes: `ConversationState`, `ConversationPoint`, `PointClassification`, `Priority`, `ThreadEntry`
- Produces: `ConversationFold.reprioritisePoint(state, pointId, messageId, messageType, sender, createdAt, role, round, newPriority, content)` → `ConversationState`

- [ ] **Step 1: Write the failing test**

Add a `@Nested class ReprioritisePoint` in `ConversationFoldTest.java`:

```java
@Nested
class ReprioritisePoint {

    @Test
    void updatesPointPriority() {
        var classification = new PointClassification(Priority.LOW, "ISOLATED", "§3.2");
        var state = ConversationFold.createPoint(empty,
                "point-1", "general", 1L, null, "rev-sender", null,
                classification, "REV", 1, "RAISE", "Minor issue");

        var result = ConversationFold.reprioritisePoint(state,
                "point-1", 2L, null, "human-sender", null,
                "HUMAN", 2, Priority.HIGH, "Actually critical");

        ConversationPoint point = result.points().get("point-1");
        assertThat(point.classification().priority()).isEqualTo(Priority.HIGH);
        assertThat(point.classification().scope()).isEqualTo("ISOLATED");
        assertThat(point.classification().location()).isEqualTo("§3.2");
    }

    @Test
    void addsThreadEntry() {
        var classification = new PointClassification(Priority.LOW, null, null);
        var state = ConversationFold.createPoint(empty,
                "point-1", "general", 1L, null, "rev-sender", null,
                classification, "REV", 1, "RAISE", "Issue");

        var result = ConversationFold.reprioritisePoint(state,
                "point-1", 2L, null, "human-sender", null,
                "HUMAN", 2, Priority.HIGH, "Escalating");

        ConversationPoint point = result.points().get("point-1");
        assertThat(point.thread()).hasSize(2);
        ThreadEntry entry = point.thread().get(1);
        assertThat(entry.role()).isEqualTo("HUMAN");
        assertThat(entry.round()).isEqualTo(2);
        assertThat(entry.entryType()).isEqualTo("REPRIORITISE");
        assertThat(entry.content()).isEqualTo("Escalating");
    }

    @Test
    void preservesStatusUnchanged() {
        var classification = new PointClassification(Priority.LOW, null, null);
        var s0 = ConversationFold.createPoint(empty,
                "point-1", "general", 1L, null, "rev-sender", null,
                classification, "REV", 1, "RAISE", "Issue");
        var s1 = ConversationFold.respondToPoint(s0,
                "point-1", 2L, null, "imp-sender", null,
                "IMP", 1, "AGREE", "Agreed", "AGREED");

        var result = ConversationFold.reprioritisePoint(s1,
                "point-1", 3L, null, "human-sender", null,
                "HUMAN", 2, Priority.HIGH, "Escalating");

        assertThat(result.points().get("point-1").status()).isEqualTo("AGREED");
    }

    @Test
    void unknownPointId_returnsStateUnchanged() {
        var result = ConversationFold.reprioritisePoint(empty,
                "nonexistent", 1L, null, "human-sender", null,
                "HUMAN", 1, Priority.HIGH, "content");

        assertThat(result).isSameAs(empty);
    }

    @Test
    void preservesOtherPointsAndState() {
        var classification = new PointClassification(Priority.LOW, null, null);
        var s0 = ConversationFold.createPoint(empty,
                "point-1", "general", 1L, null, "rev-sender", null,
                classification, "REV", 1, "RAISE", "first");
        var s1 = ConversationFold.createPoint(s0,
                "point-2", "general", 2L, null, "rev-sender", null,
                classification, "REV", 1, "RAISE", "second");
        var s2 = ConversationFold.addMemo(s1, "REV", 1, "a memo");

        var result = ConversationFold.reprioritisePoint(s2,
                "point-1", 3L, null, "human-sender", null,
                "HUMAN", 2, Priority.HIGH, "escalating");

        assertThat(result.points()).hasSize(2);
        assertThat(result.points().get("point-2").classification().priority()).isEqualTo(Priority.LOW);
        assertThat(result.memos()).hasSize(1);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/blocks/pom.xml test -Dtest=ConversationFoldTest -pl .`
Expected: compilation error — `reprioritisePoint` does not exist

- [ ] **Step 3: Implement reprioritisePoint()**

Add to `ConversationFold.java` after `respondToPoint()`:

```java
public static ConversationState reprioritisePoint(ConversationState state,
                                                   String targetId,
                                                   Long messageId,
                                                   io.casehub.qhorus.api.message.MessageType messageType,
                                                   String sender,
                                                   java.time.Instant createdAt,
                                                   String role, int round,
                                                   Priority newPriority,
                                                   String content) {
    if (!state.points().containsKey(targetId)) {return state;}

    ConversationPoint existing = state.points().get(targetId);
    var thread = new ArrayList<>(existing.thread());
    thread.add(new ThreadEntry(null, messageId, messageType, sender, createdAt,
                               role, round, "REPRIORITISE", content));

    PointClassification oldClass = existing.classification();
    PointClassification newClass = new PointClassification(newPriority,
                                                           oldClass.scope(), oldClass.location());
    var updated = new ConversationPoint(existing.id(), existing.topic(), newClass,
                                        thread, existing.status());

    var points = new LinkedHashMap<>(state.points());
    points.put(targetId, updated);

    return new ConversationState(points, new ArrayList<>(state.humanFlags()),
                                 new ArrayList<>(state.memos()), new LinkedHashMap<>(state.subTaskFindings()));
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/blocks/pom.xml test -Dtest=ConversationFoldTest`
Expected: all tests PASS

- [ ] **Step 5: Install SNAPSHOT to local Maven repo**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/blocks/pom.xml install -DskipTests`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks add src/main/java/io/casehub/blocks/conversation/ConversationFold.java src/test/java/io/casehub/blocks/conversation/ConversationFoldTest.java
git -C /Users/mdproctor/claude/casehub/blocks commit -m "feat: add ConversationFold.reprioritisePoint() for HIL priority changes

Refs casehubio/drafthouse#100"
```

---

### Task 2: Domain Model Changes (api module)

**Files:**
- Modify: `server/api/src/main/java/io/casehub/drafthouse/debate/AgentType.java`
- Modify: `server/api/src/main/java/io/casehub/drafthouse/debate/EntryType.java`
- Modify: `server/api/src/main/java/io/casehub/drafthouse/DebateSession.java`
- Modify: `server/api/src/main/java/io/casehub/drafthouse/DebateSessionSnapshot.java`
- Test: `server/api/src/test/java/io/casehub/drafthouse/DebateSessionTest.java`

**Interfaces:**
- Produces: `AgentType.HUMAN`, `EntryType.COMMENT`, `EntryType.HUMAN_OVERRIDE`,
  `EntryType.REPRIORITISE`, `DebateSession.workspacePath()`,
  `DebateSession.setWorkspacePath(String)`, `DebateSessionSnapshot` with workspacePath

- [ ] **Step 1: Add HUMAN to AgentType**

Use `ide_edit_member` on `AgentType.java`:
```java
public enum AgentType {
    REV, IMP, SUPERVISOR, MODERATOR, SELECTOR, HUMAN
}
```

- [ ] **Step 2: Add new entry types to EntryType**

Use `ide_edit_member` on `EntryType.java`:
```java
public enum EntryType {
    RAISE, AGREE, COUNTER, DISPUTE, QUALIFY, FLAG_HUMAN, DECLINED,
    VERIFIED, DEFERRED,
    MEMO,
    SUB_TASK_REQUEST,
    SUB_TASK_FINDING,
    SUB_TASK_ERROR,
    RESTART_CONTEXT,
    ROUND_SNAPSHOT,
    COMMENT,
    HUMAN_OVERRIDE,
    REPRIORITISE
}
```

- [ ] **Step 3: Add workspacePath to DebateSession**

Add field and accessors to `DebateSession.java` using `ide_insert_member`:

```java
private volatile String workspacePath;

public String workspacePath() {
    return workspacePath;
}

public void setWorkspacePath(String workspacePath) {
    this.workspacePath = workspacePath;
}
```

Update `fromSnapshot()` to restore workspacePath:
```java
// After creating session, before return:
session.setWorkspacePath(snapshot.workspacePath());
```

- [ ] **Step 4: Add workspacePath to DebateSessionSnapshot**

Use `ide_edit_member` to replace the record:
```java
public record DebateSessionSnapshot(
        UUID channelId,
        String debateSessionId,
        String channelName,
        List<DocumentEntry> documents,
        ComparisonPair comparison,
        Map<AgentType, String> participants,
        String agentId,
        String workspacePath) {}
```

Update `DebateSession.snapshot()` to include workspacePath:
```java
return new DebateSessionSnapshot(
        channelId, debateSessionId, channelName,
        docs, comp, Map.copyOf(participants), agentId, workspacePath);
```

- [ ] **Step 5: Write test for snapshot round-trip with workspacePath**

Add test to `DebateSessionTest.java`:
```java
@Test
void snapshot_preservesWorkspacePath() {
    DebateSession session = new DebateSession(
            UUID.randomUUID(), "test-session", "test-channel", "agent-1");
    session.setWorkspacePath("/tmp/workspace");

    DebateSessionSnapshot snapshot = session.snapshot();
    assertThat(snapshot.workspacePath()).isEqualTo("/tmp/workspace");

    DebateSession restored = DebateSession.fromSnapshot(snapshot);
    assertThat(restored.workspacePath()).isEqualTo("/tmp/workspace");
}

@Test
void snapshot_nullWorkspacePath_roundTrips() {
    DebateSession session = new DebateSession(
            UUID.randomUUID(), "test-session", "test-channel", "agent-1");

    DebateSessionSnapshot snapshot = session.snapshot();
    assertThat(snapshot.workspacePath()).isNull();

    DebateSession restored = DebateSession.fromSnapshot(snapshot);
    assertThat(restored.workspacePath()).isNull();
}
```

- [ ] **Step 6: Fix any compilation issues from DebateSessionSnapshot change**

The record gained a new field. All callers constructing `DebateSessionSnapshot` directly
need updating. Check with `ide_find_references` on `DebateSessionSnapshot` constructor.
Known callers: `DebateSession.snapshot()` (updated in Step 4),
`DebateSessionStoreContractTest.InMemoryDebateSessionStore` (add null for workspacePath),
any test fixtures creating snapshots.

- [ ] **Step 7: Run tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl api`
Expected: all tests PASS

- [ ] **Step 8: Commit**

```bash
git add server/api/
git commit -m "feat(#100): add HUMAN agent type, HIL entry types, and workspacePath to session

Adds AgentType.HUMAN, EntryType.COMMENT/HUMAN_OVERRIDE/REPRIORITISE.
DebateSession gains workspacePath field, persisted in snapshot for
restart resilience.

Refs #100"
```

---

### Task 3: DebateChannelProjection — COMMENT, HUMAN_OVERRIDE, REPRIORITISE

**Files:**
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/debate/DebateChannelProjection.java`
- Modify: `server/runtime/src/test/java/io/casehub/drafthouse/debate/DebateChannelProjectionTest.java`

**Interfaces:**
- Consumes: `ConversationFold.reprioritisePoint()` (Task 1), `EntryType.COMMENT`,
  `EntryType.HUMAN_OVERRIDE`, `EntryType.REPRIORITISE` (Task 2)
- Produces: Projection correctly folds COMMENT (no status change), HUMAN_OVERRIDE
  (terminal), REPRIORITISE (priority change with ThreadEntry)

- [ ] **Step 1: Write failing tests**

Add to `DebateChannelProjectionTest.java`:

```java
@Test
void comment_doesNotChangeStatus() {
    ConversationState s0 = proj.apply(proj.identity(),
            msg(MessageType.QUERY, "pt-1",
                    ratefacts("RAISE", "REV", 1, "HIGH", "ISOLATED"), "Issue."));
    ConversationState s1 = proj.apply(s0,
            msg(MessageType.RESPONSE, "pt-1", ratefacts("COMMENT", "HUMAN", 1), "Human comment."));
    assertThat(s1.points().get("pt-1").status()).isEqualTo("OPEN");
    assertThat(s1.points().get("pt-1").thread()).hasSize(2);
}

@Test
void humanOverride_transitionsToHumanOverride() {
    ConversationState s0 = proj.apply(proj.identity(),
            msg(MessageType.QUERY, "pt-1",
                    ratefacts("RAISE", "REV", 1, "HIGH", "ISOLATED"), "Issue."));
    ConversationState s1 = proj.apply(s0,
            msg(MessageType.DONE, "pt-1", ratefacts("HUMAN_OVERRIDE", "HUMAN", 1), "Overridden."));
    assertThat(s1.points().get("pt-1").status()).isEqualTo("HUMAN_OVERRIDE");
}

@Test
void reprioritise_updatesPriority() {
    ConversationState s0 = proj.apply(proj.identity(),
            msg(MessageType.QUERY, "pt-1",
                    ratefacts("RAISE", "REV", 1, "LOW", "ISOLATED"), "Minor issue."));
    String reprioritiseMeta = "entryType=REPRIORITISE|role=HUMAN|round=2|priority=HIGH";
    ConversationState s1 = proj.apply(s0,
            msg(MessageType.RESPONSE, "pt-1", reprioritiseMeta, "Actually critical."));
    assertThat(s1.points().get("pt-1").classification().priority())
            .isEqualTo(io.casehub.blocks.conversation.Priority.HIGH);
    assertThat(s1.points().get("pt-1").thread()).hasSize(2);
    assertThat(s1.points().get("pt-1").status()).isEqualTo("OPEN");
}

@Test
void reprioritise_unknownPoint_returnsStateUnchanged() {
    ConversationState s0 = proj.identity();
    String reprioritiseMeta = "entryType=REPRIORITISE|role=HUMAN|round=1|priority=HIGH";
    ConversationState s1 = proj.apply(s0,
            msg(MessageType.RESPONSE, "nonexistent", reprioritiseMeta, "Escalating."));
    assertThat(s1.points()).isEmpty();
}

@Test
void reprioritise_malformedPriority_returnsStateUnchanged() {
    ConversationState s0 = proj.apply(proj.identity(),
            msg(MessageType.QUERY, "pt-1",
                    ratefacts("RAISE", "REV", 1, "LOW", "ISOLATED"), "Issue."));
    String reprioritiseMeta = "entryType=REPRIORITISE|role=HUMAN|round=2|priority=INVALID";
    ConversationState s1 = proj.apply(s0,
            msg(MessageType.RESPONSE, "pt-1", reprioritiseMeta, "Bad priority."));
    assertThat(s1.points().get("pt-1").classification().priority())
            .isEqualTo(io.casehub.blocks.conversation.Priority.LOW);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DebateChannelProjectionTest`
Expected: COMMENT and HUMAN_OVERRIDE tests may pass (statusAfter returns null by default),
REPRIORITISE tests fail

- [ ] **Step 3: Implement statusAfter() changes**

Use `ide_replace_member` on `DebateChannelProjection.statusAfter`:
```java
@Override
protected String statusAfter(String entryType) {
    return switch (entryType) {
        case "AGREE" -> "AGREED";
        case "COUNTER", "QUALIFY" -> "ACTIVE";
        case "DISPUTE" -> "DISPUTED";
        case "DECLINED" -> "DECLINED";
        case "VERIFIED" -> "VERIFIED";
        case "DEFERRED" -> "DEFERRED";
        case "HUMAN_OVERRIDE" -> "HUMAN_OVERRIDE";
        case "COMMENT" -> null;
        default -> null;
    };
}
```

- [ ] **Step 4: Implement apply() override for REPRIORITISE**

Use `ide_replace_member` on `DebateChannelProjection.apply`:
```java
@Override
public ConversationState apply(ConversationState state, MessageView message) {
    try {
        Map<String, String> meta = ChannelMessageMeta.parseMeta(sentinel(), message.content());
        String entryType = meta.get(ConversationProtocol.ENTRY_TYPE);
        if ("ROUND_SNAPSHOT".equals(entryType)) {
            return state;
        }
        if ("REPRIORITISE".equals(entryType)) {
            String priorityStr = meta.get(ConversationProtocol.PRIORITY);
            if (priorityStr == null) {
                LOG.log(Level.WARNING, "REPRIORITISE missing priority — discarded");
                return state;
            }
            Priority newPriority;
            try {
                newPriority = Priority.valueOf(priorityStr.toUpperCase());
            } catch (IllegalArgumentException e) {
                LOG.log(Level.WARNING, "REPRIORITISE invalid priority: " + priorityStr + " — discarded");
                return state;
            }
            String role = meta.get(ConversationProtocol.ROLE);
            int round = ChannelMessageMeta.parseInt(meta, ConversationProtocol.ROUND);
            String body = ChannelMessageMeta.bodyContent(sentinel(), message.content());
            return ConversationFold.reprioritisePoint(state,
                    message.correlationId(), message.id(), message.type(),
                    message.sender(), message.createdAt(),
                    role, round, newPriority, body);
        }
    } catch (Exception e) {
        LOG.log(Level.WARNING, "Pre-base-class check failed — delegating to base", e);
    }
    return super.apply(state, message);
}
```

Add imports: `import io.casehub.blocks.conversation.ConversationFold;` and
`import io.casehub.blocks.conversation.Priority;`

- [ ] **Step 5: Update DEBATE_CONFIG**

Use `ide_replace_member` on the `DEBATE_CONFIG` field. Add to each map:

In `statusEmoji`: `Map.entry("HUMAN_OVERRIDE", "👤")`
In `resolvedStatuses`: add `"HUMAN_OVERRIDE"`
In `entryTypeLabel`: `Map.entry("COMMENT", "commented")`, `Map.entry("HUMAN_OVERRIDE", "overrode")`, `Map.entry("REPRIORITISE", "reprioritised")`
In `roleLabel`: `Map.of("REV", "REV", "IMP", "IMP", "HUMAN", "HUM")`

- [ ] **Step 6: Run tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DebateChannelProjectionTest`
Expected: all tests PASS

- [ ] **Step 7: Commit**

```bash
git add server/runtime/src/main/java/io/casehub/drafthouse/debate/DebateChannelProjection.java server/runtime/src/test/java/io/casehub/drafthouse/debate/DebateChannelProjectionTest.java
git commit -m "feat(#100): projection handles COMMENT, HUMAN_OVERRIDE, REPRIORITISE

COMMENT adds to thread without status change. HUMAN_OVERRIDE is terminal.
REPRIORITISE intercepted in apply() override, delegates to
ConversationFold.reprioritisePoint() with try-catch per PP-20260610-a47ef5.

Refs #100"
```

---

### Task 4: DebateParticipants Extraction + loadWorkspace Update

**Files:**
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/DebateParticipants.java`
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/DebateMcpTools.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/DebateParticipantsTest.java`

**Interfaces:**
- Consumes: `DebateSession.registerIfAbsent()`, `DebateSession.instanceId()`,
  `InstanceService.register()`, `DebateSessionRegistry.persist()`
- Produces: `DebateParticipants.ensureSender(session, role, instanceService, registry)` → `String` (instance ID)

- [ ] **Step 1: Write failing test**

Create `DebateParticipantsTest.java`:
```java
package io.casehub.drafthouse;

import io.casehub.drafthouse.debate.AgentType;
import io.casehub.qhorus.api.spi.InstanceService;
import org.junit.jupiter.api.Test;

import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyList;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.*;

class DebateParticipantsTest {

    @Test
    void ensureSender_registersOnFirstCall() {
        InstanceService instanceService = mock(InstanceService.class);
        DebateSessionRegistry registry = mock(DebateSessionRegistry.class);
        DebateSession session = new DebateSession(
                UUID.randomUUID(), "test-id", "test-channel", "agent-1");

        String result = DebateParticipants.ensureSender(
                session, AgentType.HUMAN, instanceService, registry);

        assertThat(result).startsWith("drafthouse-human-");
        verify(instanceService).register(anyString(), anyString(), anyList());
        verify(registry).persist(session);
    }

    @Test
    void ensureSender_returnsExistingOnSecondCall() {
        InstanceService instanceService = mock(InstanceService.class);
        DebateSessionRegistry registry = mock(DebateSessionRegistry.class);
        DebateSession session = new DebateSession(
                UUID.randomUUID(), "test-id", "test-channel", "agent-1");

        String first = DebateParticipants.ensureSender(
                session, AgentType.HUMAN, instanceService, registry);
        String second = DebateParticipants.ensureSender(
                session, AgentType.HUMAN, instanceService, registry);

        assertThat(second).isEqualTo(first);
        verify(instanceService, times(1)).register(anyString(), anyString(), anyList());
        verify(registry, times(1)).persist(session);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DebateParticipantsTest`
Expected: compilation error — `DebateParticipants` does not exist

- [ ] **Step 3: Create DebateParticipants**

Create `server/runtime/src/main/java/io/casehub/drafthouse/DebateParticipants.java`:
```java
package io.casehub.drafthouse;

import io.casehub.drafthouse.debate.AgentType;
import io.casehub.qhorus.api.spi.InstanceService;

import java.util.List;

final class DebateParticipants {

    private DebateParticipants() {}

    static String ensureSender(DebateSession session, AgentType role,
                               InstanceService instanceService,
                               DebateSessionRegistry registry) {
        String existing = session.instanceIdFor(role);
        String instanceId = session.registerIfAbsent(role, () -> {
            String id = DebateSession.instanceId(role, session.debateSessionId());
            instanceService.register(id,
                    "DraftHouse " + role.name().toLowerCase() + " " + session.debateSessionId(),
                    List.of("document-debate-" + role.name().toLowerCase()));
            return id;
        });
        if (existing == null) {
            registry.persist(session);
        }
        return instanceId;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DebateParticipantsTest`
Expected: PASS

- [ ] **Step 5: Update DebateMcpTools.sender() to use DebateParticipants**

Use `ide_replace_member` on `DebateMcpTools.sender`:
```java
private String sender(final DebateSession session, final AgentType role) {
    return DebateParticipants.ensureSender(session, role, instanceService, registry);
}
```

- [ ] **Step 6: Update DebateMcpTools.loadWorkspace() to set workspacePath**

In the `loadWorkspace` method, after workspace parsing succeeds and a debate session is
resolved, add: `session.setWorkspacePath(workspacePath);`

Find the exact location using `ide_find_definition` on loadWorkspace, then insert
`session.setWorkspacePath(workspacePath);` after the session is resolved. The workspace
path is the `workspacePath` parameter already available in the method.

- [ ] **Step 7: Run all existing tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime`
Expected: all tests PASS (refactoring — no behaviour change)

- [ ] **Step 8: Commit**

```bash
git add server/runtime/src/main/java/io/casehub/drafthouse/DebateParticipants.java server/runtime/src/test/java/io/casehub/drafthouse/DebateParticipantsTest.java server/runtime/src/main/java/io/casehub/drafthouse/DebateMcpTools.java
git commit -m "refactor(#100): extract DebateParticipants.ensureSender() and set workspacePath

Shared sender registration used by both DebateMcpTools and
HumanActionResource. loadWorkspace now stores workspacePath on
DebateSession for DecisionFileWriter.

Refs #100"
```

---

### Task 5: DecisionFileWriter

**Files:**
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/DecisionFileWriter.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/DecisionFileWriterTest.java`

**Interfaces:**
- Consumes: workspace path from `DebateSession.workspacePath()`
- Produces: `DecisionFileWriter.append(workspacePath, round, section, pointId, content)`

- [ ] **Step 1: Write failing tests**

Create `DecisionFileWriterTest.java`:
```java
package io.casehub.drafthouse;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

import static org.assertj.core.api.Assertions.assertThat;

class DecisionFileWriterTest {

    @Test
    void append_createsDirectoryAndFile(@TempDir Path tempDir) throws IOException {
        DecisionFileWriter writer = new DecisionFileWriter();
        writer.append(tempDir.toString(), 1, "Comments", "R1-05", "Human comment");

        Path file = tempDir.resolve("decisions/human-round-1.md");
        assertThat(file).exists();
        String content = Files.readString(file);
        assertThat(content).contains("# Human Decisions — Round 1");
        assertThat(content).contains("## Comments");
        assertThat(content).contains("### R1-05");
        assertThat(content).contains("Human comment");
    }

    @Test
    void append_multipleEntries_sameSection(@TempDir Path tempDir) throws IOException {
        DecisionFileWriter writer = new DecisionFileWriter();
        writer.append(tempDir.toString(), 1, "Comments", "R1-05", "First comment");
        writer.append(tempDir.toString(), 1, "Comments", "R1-07", "Second comment");

        String content = Files.readString(tempDir.resolve("decisions/human-round-1.md"));
        assertThat(content).contains("### R1-05");
        assertThat(content).contains("### R1-07");
        long sectionCount = content.lines().filter(l -> l.equals("## Comments")).count();
        assertThat(sectionCount).isEqualTo(1);
    }

    @Test
    void append_differentSections(@TempDir Path tempDir) throws IOException {
        DecisionFileWriter writer = new DecisionFileWriter();
        writer.append(tempDir.toString(), 2, "Comments", "R1-05", "A comment");
        writer.append(tempDir.toString(), 2, "Overrides", "R1-03", "Override reason");

        String content = Files.readString(tempDir.resolve("decisions/human-round-2.md"));
        assertThat(content).contains("## Comments");
        assertThat(content).contains("## Overrides");
    }

    @Test
    void append_differentRounds_separateFiles(@TempDir Path tempDir) throws IOException {
        DecisionFileWriter writer = new DecisionFileWriter();
        writer.append(tempDir.toString(), 1, "Comments", "R1-05", "Round 1");
        writer.append(tempDir.toString(), 2, "Comments", "R2-01", "Round 2");

        assertThat(tempDir.resolve("decisions/human-round-1.md")).exists();
        assertThat(tempDir.resolve("decisions/human-round-2.md")).exists();
    }

    @Test
    void append_nullWorkspacePath_silentlySkips() {
        DecisionFileWriter writer = new DecisionFileWriter();
        writer.append(null, 1, "Comments", "R1-05", "content");
        // No exception, no file created
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DecisionFileWriterTest`
Expected: compilation error — `DecisionFileWriter` does not exist

- [ ] **Step 3: Implement DecisionFileWriter**

Create `server/runtime/src/main/java/io/casehub/drafthouse/DecisionFileWriter.java`:
```java
package io.casehub.drafthouse;

import jakarta.enterprise.context.ApplicationScoped;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardOpenOption;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.logging.Level;
import java.util.logging.Logger;

@ApplicationScoped
public class DecisionFileWriter {

    private static final Logger LOG = Logger.getLogger(DecisionFileWriter.class.getName());

    private final Map<String, Object> fileLocks = new ConcurrentHashMap<>();

    public void append(String workspacePath, int round, String section,
                       String pointId, String content) {
        if (workspacePath == null) return;

        Path decisionsDir = Path.of(workspacePath, "decisions");
        Path file = decisionsDir.resolve("human-round-" + round + ".md");
        String lockKey = file.toString();
        Object lock = fileLocks.computeIfAbsent(lockKey, k -> new Object());

        synchronized (lock) {
            try {
                Files.createDirectories(decisionsDir);

                boolean isNew = !Files.exists(file);
                StringBuilder sb = new StringBuilder();

                if (isNew) {
                    sb.append("# Human Decisions — Round ").append(round).append("\n");
                }

                String existing = isNew ? "" : Files.readString(file);
                if (!existing.contains("## " + section)) {
                    sb.append("\n## ").append(section).append("\n");
                }

                sb.append("\n### ").append(pointId).append("\n");
                sb.append(content).append("\n");

                Files.writeString(file, sb.toString(),
                        StandardOpenOption.CREATE, StandardOpenOption.APPEND);
            } catch (IOException e) {
                LOG.log(Level.WARNING, "Failed to write decision file: " + file, e);
            }
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DecisionFileWriterTest`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git add server/runtime/src/main/java/io/casehub/drafthouse/DecisionFileWriter.java server/runtime/src/test/java/io/casehub/drafthouse/DecisionFileWriterTest.java
git commit -m "feat(#100): add DecisionFileWriter for workspace decisions/ files

Writes structured markdown to decisions/human-round-{n}.md.
Synchronized on file path for concurrent batch operations.
Silently skips when workspacePath is null.

Refs #100"
```

---

### Task 6: HumanActionResource — All Endpoints

**Files:**
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/HumanActionResource.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/HumanActionResourceTest.java`

**Interfaces:**
- Consumes: `DebateParticipants.ensureSender()` (Task 4), `DecisionFileWriter` (Task 5),
  `DebateChannelProjection` (Task 3), `DebateSessionRegistry`, `MessageService`,
  `InstanceService`, `ProjectionService`
- Produces: REST endpoints at `/api/debate/{id}/human/*`

- [ ] **Step 1: Write failing tests for comment and raise**

Create `HumanActionResourceTest.java` as a `@QuarkusTest`:
```java
package io.casehub.drafthouse;

import io.casehub.drafthouse.debate.AgentType;
import io.casehub.qhorus.runtime.message.MessageService;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.ws.rs.core.MediaType;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.UUID;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class HumanActionResourceTest {

    private static final Pattern DEBATE_ID_PATTERN =
            Pattern.compile("\"debateSessionId\":\"([^\"]+)\"");
    private static final Pattern POINT_ID_PATTERN =
            Pattern.compile("\"pointId\":\"([^\"]+)\"");

    @Inject DebateMcpTools tools;
    @Inject DebateSessionRegistry registry;
    @Inject MessageService messageService;

    private String activeDebateSessionId;

    @BeforeEach
    void setUp() {
        activeDebateSessionId = null;
    }

    @AfterEach
    void tearDown() {
        if (activeDebateSessionId != null) {
            tools.endDebate(activeDebateSessionId, false);
        }
    }

    private String startDebateAndRaisePoint() {
        String startResult = tools.startDebate("test-spec.md", null);
        activeDebateSessionId = extractGroup(DEBATE_ID_PATTERN, startResult);
        String raiseResult = tools.raisePoint(activeDebateSessionId, "REV", 1,
                "Test issue", "HIGH", "ISOLATED", null);
        return extractGroup(POINT_ID_PATTERN, raiseResult);
    }

    @Test
    void comment_addsEntryToChannel() {
        String pointId = startDebateAndRaisePoint();

        given()
                .contentType(MediaType.APPLICATION_JSON)
                .body("{\"pointId\":\"" + pointId + "\",\"content\":\"Human comment\"}")
                .when()
                .post("/api/debate/" + activeDebateSessionId + "/human/comment")
                .then()
                .statusCode(200);

        DebateSession session = registry.find(UUID.fromString(activeDebateSessionId)).orElseThrow();
        assertThat(session.participants()).containsKey(AgentType.HUMAN);
    }

    @Test
    void comment_invalidSession_returns404() {
        given()
                .contentType(MediaType.APPLICATION_JSON)
                .body("{\"pointId\":\"fake\",\"content\":\"comment\"}")
                .when()
                .post("/api/debate/" + UUID.randomUUID() + "/human/comment")
                .then()
                .statusCode(404);
    }

    @Test
    void raise_createsNewPoint() {
        String startResult = tools.startDebate("test-spec.md", null);
        activeDebateSessionId = extractGroup(DEBATE_ID_PATTERN, startResult);

        String body = given()
                .contentType(MediaType.APPLICATION_JSON)
                .body("{\"content\":\"Human-raised point\",\"priority\":\"P1\"}")
                .when()
                .post("/api/debate/" + activeDebateSessionId + "/human/raise")
                .then()
                .statusCode(200)
                .extract().body().asString();

        assertThat(body).contains("pointId");
    }

    @Test
    void override_setsHumanOverrideStatus() {
        String pointId = startDebateAndRaisePoint();

        given()
                .contentType(MediaType.APPLICATION_JSON)
                .body("{\"pointId\":\"" + pointId + "\",\"reason\":\"Override reason\"}")
                .when()
                .post("/api/debate/" + activeDebateSessionId + "/human/override")
                .then()
                .statusCode(200);
    }

    @Test
    void prioritise_changesPriority() {
        String pointId = startDebateAndRaisePoint();

        given()
                .contentType(MediaType.APPLICATION_JSON)
                .body("{\"pointId\":\"" + pointId + "\",\"newPriority\":\"P3\"}")
                .when()
                .post("/api/debate/" + activeDebateSessionId + "/human/prioritise")
                .then()
                .statusCode(200);
    }

    @Test
    void batch_acceptsMultiplePoints() {
        String startResult = tools.startDebate("test-spec.md", null);
        activeDebateSessionId = extractGroup(DEBATE_ID_PATTERN, startResult);

        String r1 = tools.raisePoint(activeDebateSessionId, "REV", 1,
                "Issue 1", "LOW", "ISOLATED", null);
        String r2 = tools.raisePoint(activeDebateSessionId, "REV", 1,
                "Issue 2", "LOW", "ISOLATED", null);
        String pid1 = extractGroup(POINT_ID_PATTERN, r1);
        String pid2 = extractGroup(POINT_ID_PATTERN, r2);

        given()
                .contentType(MediaType.APPLICATION_JSON)
                .body("{\"pointIds\":[\"" + pid1 + "\",\"" + pid2 + "\"],\"verdict\":\"VERIFIED\"}")
                .when()
                .post("/api/debate/" + activeDebateSessionId + "/human/batch")
                .then()
                .statusCode(200);
    }

    @Test
    void batch_emptyPointIds_returns400() {
        String startResult = tools.startDebate("test-spec.md", null);
        activeDebateSessionId = extractGroup(DEBATE_ID_PATTERN, startResult);

        given()
                .contentType(MediaType.APPLICATION_JSON)
                .body("{\"pointIds\":[],\"verdict\":\"VERIFIED\"}")
                .when()
                .post("/api/debate/" + activeDebateSessionId + "/human/batch")
                .then()
                .statusCode(400);
    }

    private static String extractGroup(Pattern pattern, String input) {
        Matcher m = pattern.matcher(input);
        assertThat(m.find()).as("Pattern %s not found in: %s", pattern, input).isTrue();
        return m.group(1);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=HumanActionResourceTest`
Expected: compilation error — `HumanActionResource` does not exist, endpoints return 404

- [ ] **Step 3: Implement HumanActionResource**

Create `server/runtime/src/main/java/io/casehub/drafthouse/HumanActionResource.java`:
```java
package io.casehub.drafthouse;

import io.casehub.blocks.channel.ChannelMessageMeta;
import io.casehub.blocks.conversation.ConversationProtocol;
import io.casehub.blocks.conversation.ConversationState;
import io.casehub.blocks.conversation.Priority;
import io.casehub.drafthouse.debate.AgentType;
import io.casehub.drafthouse.debate.DebateChannelProjection;
import io.casehub.drafthouse.debate.DebateProtocol;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.spi.InstanceService;
import io.casehub.qhorus.api.spi.ProjectionService;
import io.casehub.qhorus.runtime.message.MessageService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.util.*;
import java.util.logging.Logger;

@ApplicationScoped
@Path("/api/debate/{debateSessionId}/human")
public class HumanActionResource {

    private static final Logger LOG = Logger.getLogger(HumanActionResource.class.getName());

    @Inject DebateSessionRegistry registry;
    @Inject MessageService messageService;
    @Inject InstanceService instanceService;
    @Inject ProjectionService projectionService;
    @Inject DebateChannelProjection debateProjection;
    @Inject DecisionFileWriter decisionFileWriter;

    record CommentRequest(String pointId, String content) {}
    record RaiseRequest(String content, String priority, String location,
                        String side, Integer startLine, Integer endLine, String selectedText) {}
    record OverrideRequest(String pointId, String reason) {}
    record PrioritiseRequest(String pointId, String newPriority) {}
    record BatchRequest(List<String> pointIds, String verdict) {}

    @POST @Path("/comment")
    @Consumes(MediaType.APPLICATION_JSON) @Produces(MediaType.APPLICATION_JSON)
    public Response comment(@PathParam("debateSessionId") String debateSessionId,
                            CommentRequest request) {
        DebateSession session = resolveSession(debateSessionId);
        if (session == null) return notFound(debateSessionId);
        if (request.pointId() == null || request.content() == null || request.content().isBlank())
            return badRequest("pointId and content are required");

        Long inReplyTo = messageService.findByCorrelationId(request.pointId()).map(m -> m.id()).orElse(null);
        if (inReplyTo == null) return badRequest("point not found: " + request.pointId());

        String sender = DebateParticipants.ensureSender(session, AgentType.HUMAN, instanceService, registry);
        int round = currentRound(session);

        Map<String, String> meta = new LinkedHashMap<>();
        meta.put(ConversationProtocol.ENTRY_TYPE, "COMMENT");
        meta.put(ConversationProtocol.ROLE, "HUMAN");
        meta.put(ConversationProtocol.ROUND, String.valueOf(round));
        String encoded = ChannelMessageMeta.encode(DebateProtocol.META_SENTINEL, meta, request.content());

        messageService.dispatch(MessageDispatch.builder()
                .channelId(session.channelId())
                .sender(sender)
                .type(MessageType.RESPONSE)
                .content(encoded)
                .correlationId(request.pointId())
                .inReplyTo(inReplyTo)
                .actorType(ActorType.HUMAN)
                .build());

        decisionFileWriter.append(session.workspacePath(), round, "Comments",
                request.pointId(), request.content());

        return Response.ok("{\"status\":\"ok\"}").build();
    }

    @POST @Path("/raise")
    @Consumes(MediaType.APPLICATION_JSON) @Produces(MediaType.APPLICATION_JSON)
    public Response raise(@PathParam("debateSessionId") String debateSessionId,
                          RaiseRequest request) {
        DebateSession session = resolveSession(debateSessionId);
        if (session == null) return notFound(debateSessionId);
        if (request.content() == null || request.content().isBlank())
            return badRequest("content is required");

        String priorityStr = parsePriorityLabel(request.priority());
        if (priorityStr == null) return badRequest("priority must be P1, P2, or P3");

        String sender = DebateParticipants.ensureSender(session, AgentType.HUMAN, instanceService, registry);
        int round = currentRound(session);
        String pointId = UUID.randomUUID().toString();

        Map<String, String> meta = new LinkedHashMap<>();
        meta.put(ConversationProtocol.ENTRY_TYPE, "RAISE");
        meta.put(ConversationProtocol.ROLE, "HUMAN");
        meta.put(ConversationProtocol.ROUND, String.valueOf(round));
        meta.put(ConversationProtocol.PRIORITY, priorityStr);
        if (request.location() != null && !request.location().isBlank()) {
            meta.put(ConversationProtocol.LOCATION, request.location());
        }
        if (request.side() != null) meta.put("side", request.side());
        if (request.startLine() != null) meta.put("startLine", String.valueOf(request.startLine()));
        if (request.endLine() != null) meta.put("endLine", String.valueOf(request.endLine()));

        String encoded = ChannelMessageMeta.encode(DebateProtocol.META_SENTINEL, meta, request.content());

        messageService.dispatch(MessageDispatch.builder()
                .channelId(session.channelId())
                .sender(sender)
                .type(MessageType.QUERY)
                .content(encoded)
                .correlationId(pointId)
                .actorType(ActorType.HUMAN)
                .build());

        String location = request.location() != null ? request.location() : "";
        if (request.startLine() != null && request.endLine() != null) {
            location = (location.isEmpty() ? "" : location + ", ") +
                    "lines " + request.startLine() + "-" + request.endLine();
        }
        String shortId = pointId.length() > 8 ? pointId.substring(0, 8) : pointId;
        String header = "H-" + shortId + (location.isEmpty() ? "" : " — " + location);
        String body = request.priority() != null ? "**Priority:** " + request.priority() + "\n" + request.content() : request.content();
        decisionFileWriter.append(session.workspacePath(), round, "New Points", header, body);

        return Response.ok("{\"status\":\"ok\",\"pointId\":\"" + pointId + "\"}").build();
    }

    @POST @Path("/override")
    @Consumes(MediaType.APPLICATION_JSON) @Produces(MediaType.APPLICATION_JSON)
    public Response override(@PathParam("debateSessionId") String debateSessionId,
                             OverrideRequest request) {
        DebateSession session = resolveSession(debateSessionId);
        if (session == null) return notFound(debateSessionId);
        if (request.pointId() == null || request.reason() == null || request.reason().isBlank())
            return badRequest("pointId and reason are required");

        Long inReplyTo = messageService.findByCorrelationId(request.pointId()).map(m -> m.id()).orElse(null);
        if (inReplyTo == null) return badRequest("point not found: " + request.pointId());

        ConversationState state = projectState(session);
        var point = state.points().get(request.pointId());
        if (point != null && isResolved(point.status())) {
            return Response.status(409).entity("{\"error\":\"point already resolved: " + point.status() + "\"}").build();
        }

        String sender = DebateParticipants.ensureSender(session, AgentType.HUMAN, instanceService, registry);
        int round = currentRound(session);

        Map<String, String> meta = new LinkedHashMap<>();
        meta.put(ConversationProtocol.ENTRY_TYPE, "HUMAN_OVERRIDE");
        meta.put(ConversationProtocol.ROLE, "HUMAN");
        meta.put(ConversationProtocol.ROUND, String.valueOf(round));
        String encoded = ChannelMessageMeta.encode(DebateProtocol.META_SENTINEL, meta, request.reason());

        messageService.dispatch(MessageDispatch.builder()
                .channelId(session.channelId())
                .sender(sender)
                .type(MessageType.DONE)
                .content(encoded)
                .correlationId(request.pointId())
                .inReplyTo(inReplyTo)
                .actorType(ActorType.HUMAN)
                .build());

        decisionFileWriter.append(session.workspacePath(), round, "Overrides",
                request.pointId() + " → HUMAN_OVERRIDE", request.reason());

        return Response.ok("{\"status\":\"ok\"}").build();
    }

    @POST @Path("/prioritise")
    @Consumes(MediaType.APPLICATION_JSON) @Produces(MediaType.APPLICATION_JSON)
    public Response prioritise(@PathParam("debateSessionId") String debateSessionId,
                               PrioritiseRequest request) {
        DebateSession session = resolveSession(debateSessionId);
        if (session == null) return notFound(debateSessionId);
        if (request.pointId() == null || request.newPriority() == null)
            return badRequest("pointId and newPriority are required");

        String priorityStr = parsePriorityLabel(request.newPriority());
        if (priorityStr == null) return badRequest("newPriority must be P1, P2, or P3");

        Long inReplyTo = messageService.findByCorrelationId(request.pointId()).map(m -> m.id()).orElse(null);
        if (inReplyTo == null) return badRequest("point not found: " + request.pointId());

        ConversationState state = projectState(session);
        var point = state.points().get(request.pointId());
        if (point != null && point.classification().priority().name().equals(priorityStr)) {
            return badRequest("point already has priority " + request.newPriority());
        }

        String sender = DebateParticipants.ensureSender(session, AgentType.HUMAN, instanceService, registry);
        int round = currentRound(session);

        Map<String, String> meta = new LinkedHashMap<>();
        meta.put(ConversationProtocol.ENTRY_TYPE, "REPRIORITISE");
        meta.put(ConversationProtocol.ROLE, "HUMAN");
        meta.put(ConversationProtocol.ROUND, String.valueOf(round));
        meta.put(ConversationProtocol.PRIORITY, priorityStr);
        String content = "Priority changed to " + request.newPriority();
        String encoded = ChannelMessageMeta.encode(DebateProtocol.META_SENTINEL, meta, content);

        messageService.dispatch(MessageDispatch.builder()
                .channelId(session.channelId())
                .sender(sender)
                .type(MessageType.RESPONSE)
                .content(encoded)
                .correlationId(request.pointId())
                .inReplyTo(inReplyTo)
                .actorType(ActorType.HUMAN)
                .build());

        String oldPriority = point != null ? labelForPriority(point.classification().priority()) : "?";
        decisionFileWriter.append(session.workspacePath(), round, "Priority Changes",
                request.pointId() + " → " + request.newPriority() + " (was " + oldPriority + ")", content);

        return Response.ok("{\"status\":\"ok\"}").build();
    }

    @POST @Path("/batch")
    @Consumes(MediaType.APPLICATION_JSON) @Produces(MediaType.APPLICATION_JSON)
    public Response batch(@PathParam("debateSessionId") String debateSessionId,
                          BatchRequest request) {
        DebateSession session = resolveSession(debateSessionId);
        if (session == null) return notFound(debateSessionId);
        if (request.pointIds() == null || request.pointIds().isEmpty())
            return badRequest("pointIds must be non-empty");
        if (request.verdict() == null || (!request.verdict().equals("VERIFIED") && !request.verdict().equals("DEFERRED")))
            return badRequest("verdict must be VERIFIED or DEFERRED");

        String sender = DebateParticipants.ensureSender(session, AgentType.HUMAN, instanceService, registry);
        int round = currentRound(session);
        MessageType msgType = "VERIFIED".equals(request.verdict()) ? MessageType.DONE : MessageType.DECLINE;

        for (String pointId : request.pointIds()) {
            Long inReplyTo = messageService.findByCorrelationId(pointId).map(m -> m.id()).orElse(null);
            if (inReplyTo == null) continue;

            Map<String, String> meta = new LinkedHashMap<>();
            meta.put(ConversationProtocol.ENTRY_TYPE, request.verdict());
            meta.put(ConversationProtocol.ROLE, "HUMAN");
            meta.put(ConversationProtocol.ROUND, String.valueOf(round));
            String encoded = ChannelMessageMeta.encode(DebateProtocol.META_SENTINEL, meta, "Batch " + request.verdict().toLowerCase());

            messageService.dispatch(MessageDispatch.builder()
                    .channelId(session.channelId())
                    .sender(sender)
                    .type(msgType)
                    .content(encoded)
                    .correlationId(pointId)
                    .inReplyTo(inReplyTo)
                    .actorType(ActorType.HUMAN)
                    .build());
        }

        String label = "VERIFIED".equals(request.verdict()) ? "Approved" : "Deferred";
        decisionFileWriter.append(session.workspacePath(), round, "Batch Decisions",
                label, label + ": " + String.join(", ", request.pointIds()));

        return Response.ok("{\"status\":\"ok\"}").build();
    }

    // ── helpers ───────────────────────────────────────────────────────────

    private DebateSession resolveSession(String debateSessionId) {
        try {
            UUID channelId = UUID.fromString(debateSessionId);
            return registry.find(channelId).orElse(null);
        } catch (IllegalArgumentException e) {
            return null;
        }
    }

    private int currentRound(DebateSession session) {
        try {
            ConversationState state = projectState(session);
            return state.points().values().stream()
                    .flatMap(p -> p.thread().stream())
                    .mapToInt(e -> e.round())
                    .max()
                    .orElse(1);
        } catch (Exception e) {
            return 1;
        }
    }

    private ConversationState projectState(DebateSession session) {
        return projectionService.project(session.channelId(), debateProjection).state();
    }

    private static boolean isResolved(String status) {
        return Set.of("AGREED", "DECLINED", "VERIFIED", "DEFERRED", "HUMAN_OVERRIDE").contains(status);
    }

    private static String parsePriorityLabel(String label) {
        if (label == null) return null;
        return switch (label.toUpperCase()) {
            case "P1", "HIGH" -> "HIGH";
            case "P2", "MEDIUM" -> "MEDIUM";
            case "P3", "LOW" -> "LOW";
            default -> null;
        };
    }

    private static String labelForPriority(Priority p) {
        return switch (p) {
            case HIGH -> "P1";
            case MEDIUM -> "P2";
            case LOW -> "P3";
        };
    }

    private static Response notFound(String id) {
        return Response.status(404).entity("{\"error\":\"session not found: " + id + "\"}").build();
    }

    private static Response badRequest(String msg) {
        return Response.status(400).entity("{\"error\":\"" + msg + "\"}").build();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=HumanActionResourceTest`
Expected: all tests PASS

- [ ] **Step 5: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime`
Expected: all tests PASS — no regressions

- [ ] **Step 6: Commit**

```bash
git add server/runtime/src/main/java/io/casehub/drafthouse/HumanActionResource.java server/runtime/src/test/java/io/casehub/drafthouse/HumanActionResourceTest.java
git commit -m "feat(#100): add HumanActionResource — 5 REST endpoints for HIL

comment, raise, override, prioritise, batch endpoints at
/api/debate/{id}/human/*. Each dispatches to Qhorus channel with
ActorType.HUMAN and writes to decisions/ via DecisionFileWriter.

Refs #100"
```

---

### Task 7: UI Changes — Channel Feed + Review Tracker

**Files:**
- Modify: `server/runtime/src/main/webui/src/panels/channel-feed.ts`
- Modify: `server/runtime/src/main/webui/src/panels/review-tracker.ts`

**Interfaces:**
- Consumes: `DebateStreamEntry` with `agentRole: "HUMAN"`, entry types `COMMENT`,
  `HUMAN_OVERRIDE`, `REPRIORITISE`. REST endpoints from Task 6.
- Produces: Human badge styling, interactive action buttons, batch accept bar

- [ ] **Step 1: Update channel-feed.ts label maps**

Update `AGENT_LABELS`:
```typescript
const AGENT_LABELS: Record<string, string> = {
  REV: 'Reviewer',
  IMP: 'Implementer',
  HUMAN: 'Human',
  SUPERVISOR: 'Supervisor',
  MODERATOR: 'Moderator',
  SELECTOR: 'Selector',
};
```

Update `ENTRY_TYPE_LABELS` — add:
```typescript
  COMMENT: 'commented',
  HUMAN_OVERRIDE: 'overrode',
  REPRIORITISE: 'reprioritised',
  VERIFIED: 'verified',
  DEFERRED: 'deferred',
```

- [ ] **Step 2: Add human badge styling in channel-feed.ts**

In the `_renderEntry` method, detect human entries and apply distinct styling.
Add a CSS class `.human` to the role badge when `entry.agentRole === 'HUMAN'`.

In `static styles`, add:
```css
.role-badge.human {
  background: var(--human-badge-bg, #e67e22);
  color: var(--human-badge-fg, #fff);
}
.role-badge.human::before {
  content: '👤 ';
}
```

- [ ] **Step 3: Update review-tracker.ts status/label maps**

Update `ENTRY_TO_STATUS` — add:
```typescript
  HUMAN_OVERRIDE: 'HUMAN_OVERRIDE',
```
(COMMENT not added — no status change, same behaviour as MEMO)

Update `STATUS_ORDER` — add: `HUMAN_OVERRIDE: 8`
Update `STATUS_ICON` — add: `HUMAN_OVERRIDE: '👤'`
Update `RESOLVED_STATUSES` — add: `'HUMAN_OVERRIDE'`
Update `AGENT_SHORT` — add: `HUMAN: 'HUM', SUPERVISOR: 'SUP', MODERATOR: 'MOD', SELECTOR: 'SEL'`
  Remove stale `FAC` entry.
Update `ACTION_SHORT` — add: `COMMENT: 'commented', HUMAN_OVERRIDE: 'overrode', REPRIORITISE: 'reprioritised'`

- [ ] **Step 4: Add action buttons to review-tracker.ts**

Add a `_debateSessionId` property (set from `configure(props)`).

In `_renderPoint()`, after the point summary, add action buttons for unresolved points:

```typescript
${!RESOLVED_STATUSES.has(point.status) ? html`
  <div class="point-actions">
    <button class="action-btn comment-btn" title="Comment"
      @click=${(e: Event) => { e.stopPropagation(); this._showCommentInput(point); }}>💬</button>
    <button class="action-btn override-btn" title="Override"
      @click=${(e: Event) => { e.stopPropagation(); this._showOverrideInput(point); }}>👤</button>
    <select class="action-btn priority-select" title="Priority"
      @change=${(e: Event) => this._onPriorityChange(point, e)}>
      <option value="">↕</option>
      <option value="P1">P1</option>
      <option value="P2">P2</option>
      <option value="P3">P3</option>
    </select>
  </div>
` : nothing}
```

Add state for inline inputs:
```typescript
@state() private _commentingPointId: string | null = null;
@state() private _overridingPointId: string | null = null;
```

Add methods:
- `_showCommentInput(point)` — sets `_commentingPointId`, renders inline text input
- `_submitComment(pointId, content)` — POSTs to `/api/debate/{id}/human/comment`
- `_showOverrideInput(point)` — sets `_overridingPointId`, renders inline reason input
- `_submitOverride(pointId, reason)` — POSTs to `/api/debate/{id}/human/override`
- `_onPriorityChange(point, event)` — POSTs to `/api/debate/{id}/human/prioritise`

Each method uses `fetch()` to POST JSON to the REST endpoint.

- [ ] **Step 5: Add batch accept bar**

Add state: `@state() private _batchEligiblePointIds: string[] = [];`

In the render method, above the points list, add:
```typescript
${this._batchEligiblePointIds.length >= 2 ? html`
  <div class="batch-bar">
    <span>${this._batchEligiblePointIds.length} low-priority points open</span>
    <button @click=${() => this._submitBatch('VERIFIED')}>Accept all</button>
    <button @click=${() => this._submitBatch('DEFERRED')}>Defer all</button>
  </div>
` : nothing}
```

Compute `_batchEligiblePointIds` in `_derivePoints()` — collect pointIds where
status is unresolved and priority is LOW (P3).

Add `_submitBatch(verdict)` method — POSTs to `/api/debate/{id}/human/batch`.

- [ ] **Step 6: Add CSS for action buttons and batch bar**

```css
.point-actions {
  display: flex; gap: 4px; margin-top: 4px;
}
.action-btn {
  background: var(--surface-2, #333); border: 1px solid var(--border, #555);
  color: var(--text-secondary, #aaa); cursor: pointer;
  font-size: 12px; padding: 2px 6px; border-radius: 3px;
}
.action-btn:hover { background: var(--surface-3, #444); }
.batch-bar {
  display: flex; align-items: center; gap: 8px;
  padding: 6px 10px; background: var(--surface-1, #252525);
  border-bottom: 1px solid var(--border, #333); font-size: 12px;
}
.batch-bar button {
  background: var(--accent, #4a9eff); color: #fff; border: none;
  padding: 3px 8px; border-radius: 3px; cursor: pointer; font-size: 11px;
}
.comment-input, .override-input {
  width: 100%; margin-top: 4px; padding: 4px;
  background: var(--surface-2, #333); border: 1px solid var(--border, #555);
  color: var(--text-primary, #e0e0e0); font-size: 12px; border-radius: 3px;
}
```

- [ ] **Step 7: Build and verify**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml package -DskipTests`
Expected: build succeeds (Quinoa bundles TypeScript)

Verify with `ide_diagnostics` on both TypeScript files.

- [ ] **Step 8: Commit**

```bash
git add server/runtime/src/main/webui/src/panels/channel-feed.ts server/runtime/src/main/webui/src/panels/review-tracker.ts
git commit -m "feat(#100): UI — human badge, action buttons, batch accept bar

Channel feed: HUMAN agent label, human badge colour with 👤 prefix.
Clean up stale FAC label, add missing agent types.
Review tracker: action buttons (comment, override, priority) on
unresolved points, batch accept/defer bar for 2+ LOW-priority points.

Refs #100"
```

---

## Self-Review

**Spec coverage:**
- ✅ AgentType.HUMAN — Task 2
- ✅ EntryType.COMMENT/HUMAN_OVERRIDE/REPRIORITISE — Task 2
- ✅ DebateChannelProjection hooks — Task 3
- ✅ DebateSession.workspacePath — Task 2
- ✅ DebateSessionSnapshot.workspacePath — Task 2
- ✅ DebateParticipants.ensureSender() extraction — Task 4
- ✅ DecisionFileWriter — Task 5
- ✅ HumanActionResource (all 5 endpoints) — Task 6
- ✅ Input validation — Task 6
- ✅ Round derivation — Task 6
- ✅ Channel feed labels and styling — Task 7
- ✅ Review tracker status maps and action buttons — Task 7
- ✅ Batch accept bar — Task 7
- ✅ Document diff raise-point button — Task 7 (raise endpoint ready; diff panel button deferred to integration)
- ⚠️ Document diff "Raise Point" button — the spec mentions a button appearing on text selection in the diff panel. This requires changes to `document-diff.ts` which is not included in Task 7. However, the REST endpoint exists (Task 6). The diff panel integration is a visual concern that can be added as a follow-on once the core flow works.

**Placeholder scan:** No TBDs, TODOs, or "similar to" references found.

**Type consistency:** Checked signatures across tasks — `ensureSender`, `reprioritisePoint`, `DecisionFileWriter.append` are consistent.

**Tooling safety scan:** No bash file operations on source files. All code edits use `ide_*` tools or `ide_create_file`.
