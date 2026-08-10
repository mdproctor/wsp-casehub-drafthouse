# Autonomous Debate Triggering Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #71 — Claude-to-Claude continuous conversation protocol
**Issue group:** #71

**Goal:** Wire the triggering logic that starts `ConversationOrchestrator.converse()` when
the first message arrives on an autonomous debate session.

**Architecture:** `DebateChannelBackend.post()` gains an autonomous trigger path: on first
qualifying message for an autonomous session, build a `MessageView` from the `OutboundMessage`
and subscribe to `orchestrator.converse()` on a virtual thread. Completion and failure push
WebSocket metadata events. `ContestedEscalation` is added to the termination composition.
External interruption (FLAG_HUMAN, end_debate) calls `orchestrator.terminate()`.

**Tech Stack:** Java 21 (virtual threads), casehub-blocks 0.2-SNAPSHOT
(`ConversationOrchestrator`, `ContestedEscalation`, `TerminationDecision`),
casehub-qhorus 0.2-SNAPSHOT (`MessageView`, `OutboundMessage`), Mockito, AssertJ

## Global Constraints

- `DebateSession` is in `server/api/` (no runtime dependencies)
- `DebateChannelBackend` and `DebateMcpTools` are in `server/runtime/`
- All errors returned as `"error: ..."` strings per `mcp-tool-error-strings.md`
- `DebateProtocol.META_SENTINEL` prefix required for all encoded content
- `OutboundMessage.senderActorType()` maps to `MessageView.actorType()`
- `ConversationOrchestrator.converse()` returns `Uni<ConversationOutcome>` — the
  `Uni.createFrom().item()` wrapper runs the loop synchronously on the subscribing thread
- `ConversationOrchestrator.terminate()` sets a volatile boolean checked each iteration

---

### Task 1: DebateSession — add converseStarted guard

**Files:**
- Modify: `server/api/src/main/java/io/casehub/drafthouse/DebateSession.java`
- Test: `server/api/src/test/java/io/casehub/drafthouse/DebateSessionAutonomousTest.java`

**Interfaces:**
- Consumes: existing `DebateSession` class with `autonomous`, `orchestrator` fields
- Produces: `boolean markConverseStarted()` — returns true exactly once (CAS guard)

- [ ] **Step 1: Write the failing tests**

Add to `DebateSessionAutonomousTest.java`:

```java
@Test
void markConverseStarted_returnsTrueOnFirstCall() {
    var session = new DebateSession(UUID.randomUUID(), "s1", "ch-1", "agent-1");
    assertThat(session.markConverseStarted()).isTrue();
}

@Test
void markConverseStarted_returnsFalseOnSubsequentCalls() {
    var session = new DebateSession(UUID.randomUUID(), "s1", "ch-1", "agent-1");
    session.markConverseStarted();
    assertThat(session.markConverseStarted()).isFalse();
    assertThat(session.markConverseStarted()).isFalse();
}

@Test
void branchFrom_doesNotCopyConverseStarted() {
    var source = new DebateSession(UUID.randomUUID(), "s1", "ch-1", "agent-1");
    source.markConverseStarted();
    var branched = DebateSession.branchFrom(source, UUID.randomUUID(), "s2", "ch-2");
    assertThat(branched.markConverseStarted()).isTrue();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl api -Dtest=DebateSessionAutonomousTest`
Expected: FAIL — `markConverseStarted()` does not exist

- [ ] **Step 3: Implement markConverseStarted()**

Add to `DebateSession.java`:

```java
import java.util.concurrent.atomic.AtomicBoolean;
```

Add field (alongside other volatile fields):

```java
private final AtomicBoolean converseStarted = new AtomicBoolean(false);
```

Add method (after `setOrchestrator`):

```java
public boolean markConverseStarted() {
    return converseStarted.compareAndSet(false, true);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl api -Dtest=DebateSessionAutonomousTest`
Expected: PASS (all 8 tests)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/drafthouse add server/api/src/main/java/io/casehub/drafthouse/DebateSession.java server/api/src/test/java/io/casehub/drafthouse/DebateSessionAutonomousTest.java
git -C /Users/mdproctor/claude/casehub/drafthouse commit -m "feat(#71): add AtomicBoolean converseStarted guard to DebateSession

CAS-based idempotency: markConverseStarted() returns true exactly once.
Prevents concurrent messages from triggering multiple converse() calls.

Refs #71"
```

---

### Task 2: DebateChannelBackend — autonomous trigger in post()

**Files:**
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/DebateChannelBackend.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/DebateChannelBackendFactoryTest.java`

**Interfaces:**
- Consumes: `DebateSession.isAutonomous()`, `DebateSession.orchestrator()`,
  `DebateSession.markConverseStarted()` (from Task 1),
  `ConversationOrchestrator.converse(MessageView)` → `Uni<ConversationOutcome>`,
  `ConversationOrchestrator.terminate()`,
  `WebSocketEventBus.pushMetadata(UUID, String, Map)`,
  `OutboundMessage` record fields: `sender()`, `type()`, `content()`, `correlationId()`,
  `inReplyTo()`, `target()`, `topic()`, `artefactRefs()`, `senderActorType()`
- Produces: autonomous trigger path in `post()`, `handleCompletion()`, `handleFailure()`,
  FLAG_HUMAN → `terminate()` path, WebSocket metadata events
  `"autonomous-completed"` and `"autonomous-failed"`

- [ ] **Step 1: Write the failing test — trigger converse on first autonomous message**

Add to `DebateChannelBackendFactoryTest.java`:

```java
@Test
void autonomousSession_firstMessage_triggersConverseOnVirtualThread() throws Exception {
    UUID channelId = UUID.randomUUID();
    ChannelRef channelRef = new ChannelRef(channelId, "drafthouse/debate/d-" + channelId);

    DebateSession session = new DebateSession(
            channelId, channelId.toString(), channelRef.name(), null);
    session.setAutonomous(true);

    var orchestrator = mock(io.casehub.blocks.conversation.orchestration.ConversationOrchestrator.class);
    var outcome = new io.casehub.blocks.conversation.orchestration.ConversationOutcome(
            null, new io.casehub.blocks.agentic.termination.TerminationDecision.Complete("All agreed"),
            java.util.List.of(), 4, java.time.Duration.ofSeconds(10));
    when(orchestrator.converse(any())).thenReturn(
            io.smallrye.mutiny.Uni.createFrom().item(outcome));
    session.setOrchestrator(orchestrator);

    when(debateRegistry.find(channelId)).thenReturn(Optional.of(session));

    OutboundMessage message = new OutboundMessage(
            UUID.randomUUID(), "rev-agent", MessageType.QUERY,
            DebateProtocol.META_SENTINEL + "entryType=RAISE|role=REV|round=1|priority=P1\n\nTest point",
            UUID.randomUUID().toString(), null,
            io.casehub.platform.api.identity.ActorType.AGENT, java.util.List.of(), null);

    debateBackend.post(channelRef, message);

    // Virtual thread is async — wait briefly for it to complete
    Thread.sleep(500);

    verify(orchestrator).converse(any(io.casehub.qhorus.api.message.MessageView.class));
}
```

- [ ] **Step 2: Write the failing test — second message does NOT re-trigger**

```java
@Test
void autonomousSession_secondMessage_doesNotRetrigger() throws Exception {
    UUID channelId = UUID.randomUUID();
    ChannelRef channelRef = new ChannelRef(channelId, "drafthouse/debate/d-" + channelId);

    DebateSession session = new DebateSession(
            channelId, channelId.toString(), channelRef.name(), null);
    session.setAutonomous(true);

    var orchestrator = mock(io.casehub.blocks.conversation.orchestration.ConversationOrchestrator.class);
    java.util.concurrent.CountDownLatch latch = new java.util.concurrent.CountDownLatch(1);
    when(orchestrator.converse(any())).thenReturn(
            io.smallrye.mutiny.Uni.createFrom().item(() -> {
                try { latch.await(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                return new io.casehub.blocks.conversation.orchestration.ConversationOutcome(
                        null, new io.casehub.blocks.agentic.termination.TerminationDecision.Complete("done"),
                        java.util.List.of(), 2, java.time.Duration.ofSeconds(5));
            }));
    session.setOrchestrator(orchestrator);

    when(debateRegistry.find(channelId)).thenReturn(Optional.of(session));

    OutboundMessage msg1 = new OutboundMessage(
            UUID.randomUUID(), "rev-agent", MessageType.QUERY,
            DebateProtocol.META_SENTINEL + "entryType=RAISE|role=REV|round=1|priority=P1\n\nFirst",
            UUID.randomUUID().toString(), null,
            io.casehub.platform.api.identity.ActorType.AGENT, java.util.List.of(), null);
    OutboundMessage msg2 = new OutboundMessage(
            UUID.randomUUID(), "imp-agent", MessageType.RESPONSE,
            DebateProtocol.META_SENTINEL + "entryType=COUNTER|role=IMP|round=1\n\nSecond",
            UUID.randomUUID().toString(), null,
            io.casehub.platform.api.identity.ActorType.AGENT, java.util.List.of(), null);

    debateBackend.post(channelRef, msg1);
    debateBackend.post(channelRef, msg2);

    latch.countDown();
    Thread.sleep(500);

    verify(orchestrator, times(1)).converse(any());
}
```

- [ ] **Step 3: Write the failing test — non-autonomous session does not trigger**

```java
@Test
void nonAutonomousSession_doesNotTriggerConverse() {
    UUID channelId = UUID.randomUUID();
    ChannelRef channelRef = new ChannelRef(channelId, "drafthouse/debate/d-" + channelId);

    DebateSession session = new DebateSession(
            channelId, channelId.toString(), channelRef.name(), null);
    // autonomous defaults to false

    when(debateRegistry.find(channelId)).thenReturn(Optional.of(session));

    OutboundMessage message = new OutboundMessage(
            UUID.randomUUID(), "rev-agent", MessageType.QUERY,
            DebateProtocol.META_SENTINEL + "entryType=RAISE|role=REV|round=1|priority=P1\n\nTest",
            UUID.randomUUID().toString(), null,
            io.casehub.platform.api.identity.ActorType.AGENT, java.util.List.of(), null);

    debateBackend.post(channelRef, message);

    // No orchestrator interaction — session isn't autonomous
    assertThat(session.orchestrator()).isNull();
}
```

- [ ] **Step 4: Write the failing test — FLAG_HUMAN terminates running orchestrator**

```java
@Test
void autonomousSession_flagHuman_terminatesOrchestrator() {
    UUID channelId = UUID.randomUUID();
    ChannelRef channelRef = new ChannelRef(channelId, "drafthouse/debate/d-" + channelId);

    DebateSession session = new DebateSession(
            channelId, channelId.toString(), channelRef.name(), null);
    session.setAutonomous(true);
    session.markConverseStarted();  // simulate already running

    var orchestrator = mock(io.casehub.blocks.conversation.orchestration.ConversationOrchestrator.class);
    session.setOrchestrator(orchestrator);

    when(debateRegistry.find(channelId)).thenReturn(Optional.of(session));

    OutboundMessage flagMsg = new OutboundMessage(
            UUID.randomUUID(), "human-user", MessageType.HANDOFF,
            DebateProtocol.META_SENTINEL + "entryType=FLAG_HUMAN|role=REV|round=2\n\nNeeds human review",
            UUID.randomUUID().toString(), null,
            io.casehub.platform.api.identity.ActorType.HUMAN, java.util.List.of(), null);

    debateBackend.post(channelRef, flagMsg);

    verify(orchestrator).terminate();
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DebateChannelBackendFactoryTest`
Expected: FAIL — autonomous trigger logic does not exist

- [ ] **Step 6: Implement the autonomous trigger in DebateChannelBackend**

Modify `DebateChannelBackend.java`. Add import:

```java
import io.casehub.qhorus.api.message.MessageView;
import java.time.Instant;
import java.util.Map;
```

Add the trigger logic at the end of `post()`, after the existing SUB_TASK_REQUEST
dispatch and before the closing brace. The autonomous check must run for ALL message
types (not just SUB_TASK_REQUEST), so it goes after the thread-message early return
but applies to the full message flow. Restructure `post()`:

```java
@Override
public void post(ChannelRef channel, OutboundMessage message) {
    Map<String, String> meta = DebateProtocol.parseMeta(message.content());

    // Thread messages → thread event path
    if (meta.containsKey("threadId")) {
        // ... existing thread handling unchanged ...
        return;
    }

    // Existing debate entry path
    io.casehub.drafthouse.debate.DebateStreamEntry entry =
            io.casehub.drafthouse.debate.DebateStreamEntry.from(message);
    if (entry != null) {
        eventBus.pushDebateEntries(channel.id(), java.util.List.of(entry));
    }

    // SUB_TASK_REQUEST dispatch (existing)
    if ("SUB_TASK_REQUEST".equals(meta.get("entryType"))) {
        DebateSession session = registry.find(channel.id()).orElse(null);
        if (session != null) {
            String correlationId = message.correlationId() != null
                                   ? message.correlationId() : UUID.randomUUID().toString();
            channelAgentEvent.fireAsync(new ChannelAgentRequest(
                    channel.id(), correlationId, message, null));
        } else {
            LOG.warning("DebateChannelBackend: SUB_TASK_REQUEST on " + channel.id()
                        + " — no active session, dropped");
        }
        return;
    }

    // Autonomous trigger — check on every non-SUB_TASK, non-thread message
    DebateSession session = registry.find(channel.id()).orElse(null);
    if (session == null || !session.isAutonomous()) { return; }

    // FLAG_HUMAN from external source → terminate running orchestrator
    if ("FLAG_HUMAN".equals(meta.get("entryType"))
            && session.orchestrator() != null) {
        session.orchestrator().terminate();
        return;
    }

    // First qualifying message → start converse() on virtual thread
    if (session.orchestrator() != null && session.markConverseStarted()) {
        MessageView triggeringMessage = new MessageView(
                null, channel.id(), message.sender(), message.type(),
                message.content(), message.correlationId(), message.inReplyTo(),
                message.target(), message.topic(), message.artefactRefs(),
                message.senderActorType(), Instant.now(), null, 0);

        Thread.startVirtualThread(() -> {
            try {
                var outcome = session.orchestrator()
                        .converse(triggeringMessage)
                        .await().indefinitely();
                handleCompletion(channel.id(), session, outcome);
            } catch (Exception e) {
                LOG.warning("Autonomous debate failed on " + channel.id() + ": " + e.getMessage());
                handleFailure(channel.id(), session, e);
            }
        });
    }
}
```

Add completion and failure handlers:

```java
private void handleCompletion(UUID channelId, DebateSession session,
        io.casehub.blocks.conversation.orchestration.ConversationOutcome outcome) {
    String reason = switch (outcome.terminationDecision()) {
        case io.casehub.blocks.agentic.termination.TerminationDecision.Complete c ->
                c.result() != null ? c.result().toString() : "completed";
        case io.casehub.blocks.agentic.termination.TerminationDecision.Escalate e -> "escalated: " + e.reason();
        case io.casehub.blocks.agentic.termination.TerminationDecision.Failed f -> "failed: " + f.reason();
        default -> "unknown";
    };
    eventBus.pushMetadata(channelId, "autonomous-completed",
            Map.of("reason", reason,
                   "dispatchCount", outcome.dispatchCount(),
                   "durationMs", outcome.elapsed().toMillis()));
    session.setOrchestrator(null);
}

private void handleFailure(UUID channelId, DebateSession session, Exception e) {
    eventBus.pushMetadata(channelId, "autonomous-failed",
            Map.of("error", e.getMessage() != null ? e.getMessage() : e.getClass().getName()));
    session.setOrchestrator(null);
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DebateChannelBackendFactoryTest`
Expected: PASS (all tests including new autonomous tests)

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/drafthouse add server/runtime/src/main/java/io/casehub/drafthouse/DebateChannelBackend.java server/runtime/src/test/java/io/casehub/drafthouse/DebateChannelBackendFactoryTest.java
git -C /Users/mdproctor/claude/casehub/drafthouse commit -m "feat(#71): trigger orchestrator.converse() from DebateChannelBackend.post()

On first qualifying message for an autonomous session, build MessageView
from OutboundMessage and subscribe to converse() on a virtual thread.
CAS guard via markConverseStarted() ensures exactly-once trigger.
FLAG_HUMAN from external sources calls orchestrator.terminate().
Completion/failure push WebSocket metadata events.

Refs #71"
```

---

### Task 3: DebateMcpTools — ContestedEscalation + endDebate termination

**Files:**
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/DebateMcpTools.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/DebateSessionLifecycleTest.java`

**Interfaces:**
- Consumes: `ContestedEscalation(int maxDisputeRounds)` from blocks,
  `ConversationOrchestrator.terminate()`,
  `DebateSession.orchestrator()`, `DebateSession.setOrchestrator(null)`
- Produces: updated `wireAutonomousOrchestrator()` with `ContestedEscalation(3)`,
  updated `endDebate()` with orchestrator termination

- [ ] **Step 1: Write the failing test — endDebate terminates running orchestrator**

Add to `DebateSessionLifecycleTest.java`:

```java
@Test
void endDebate_terminatesRunningOrchestrator() {
    String result = tools.startDebate(specFile.toAbsolutePath().toString(), null, true);
    assertThat(result).contains("autonomous\":true");

    String sessionId = extractSessionId(result);
    DebateSession session = registry.find(UUID.fromString(sessionId)).orElseThrow();
    assertThat(session.orchestrator()).isNotNull();

    String endResult = tools.endDebate(sessionId, false);
    assertThat(endResult).contains("\"status\":\"ended\"");
    assertThat(session.orchestrator()).isNull();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DebateSessionLifecycleTest#endDebate_terminatesRunningOrchestrator`
Expected: FAIL — endDebate does not call terminate() or clear orchestrator

- [ ] **Step 3: Add orchestrator termination to endDebate()**

In `DebateMcpTools.java`, in the `endDebate()` method, add after
`var watcher = activeWatchers.remove(debateSessionId);` block and before
`registry.remove(channelId);`:

```java
if (session.orchestrator() != null) {
    session.orchestrator().terminate();
    session.setOrchestrator(null);
}
```

- [ ] **Step 4: Add ContestedEscalation to wireAutonomousOrchestrator()**

In `wireAutonomousOrchestrator()`, add import:

```java
import io.casehub.blocks.conversation.orchestration.ContestedEscalation;
```

Change the termination composition from:

```java
var termination = new CompositeTermination(List.of(
        new AllAgreedTermination(Set.of("AGREED", "VERIFIED")),
        new MaxIterationsTermination<>(20)));
```

To:

```java
var termination = new CompositeTermination(List.of(
        new AllAgreedTermination(Set.of("AGREED", "VERIFIED")),
        new ContestedEscalation(3),
        new MaxIterationsTermination<>(20)));
```

- [ ] **Step 5: Run all lifecycle tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DebateSessionLifecycleTest`
Expected: PASS (all tests)

- [ ] **Step 6: Run the full test suite**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime`
Expected: PASS (all tests, no regressions)

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/drafthouse add server/runtime/src/main/java/io/casehub/drafthouse/DebateMcpTools.java server/runtime/src/test/java/io/casehub/drafthouse/DebateSessionLifecycleTest.java
git -C /Users/mdproctor/claude/casehub/drafthouse commit -m "feat(#71): add ContestedEscalation + orchestrator termination in endDebate

ContestedEscalation(3) terminates autonomous debates when a point is
disputed more than 3 times. endDebate() calls orchestrator.terminate()
and clears the reference for sessions with a running orchestrator.

Refs #71"
```
