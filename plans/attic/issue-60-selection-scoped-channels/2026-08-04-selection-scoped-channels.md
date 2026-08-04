# Selection-Scoped Conversation Channels Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #60 — Selection-scoped conversation channels
**Issue group:** #60

**Goal:** Add persistent conversation threads anchored to document
selections, with a unified model for human and agent-initiated threads.

**Architecture:** Threads are a metadata-partitioned grouping within the
existing Qhorus channel. `DebateSession` holds thread metadata (ID,
anchor, status). `ThreadProjection` owns conversation content.
`ThreadMcpTools` provides the agent surface. `ThreadStreamEntry` carries
thread events to the browser via WebSocket. A new `<selection-threads>`
panel renders threads with bidirectional linking to the diff panel.

**Tech Stack:** Java 21, Quarkus 3.34.3, Qhorus channels, LitElement,
casehub-pages, TypeScript

## Global Constraints

- Thread messages share the debate channel — no separate channels
- `DHMETA:` sentinel for all thread message encoding
- `threadId` metadata key partitions thread messages from debate messages
- `ThreadProjection` is sole authority for thread conversation content
- `DebateSession.threads` holds metadata only (ID, anchor, status)
- Thread lifecycle: OPEN → RESOLVED (no REOPEN)
- `channel-projection-apply-must-not-throw` protocol applies to ThreadProjection
- `channel-projection-actor-type` protocol applies to thread message dispatch
- `debate-message-sentinel-encoding` protocol applies to thread encoding

---

### Task 1: Domain Model — SelectionThread, ThreadStatus, ThreadEntry

**Files:**
- Create: `server/api/src/main/java/io/casehub/drafthouse/SelectionThread.java`
- Create: `server/api/src/main/java/io/casehub/drafthouse/ThreadStatus.java`
- Create: `server/api/src/main/java/io/casehub/drafthouse/ThreadEntry.java`
- Modify: `server/api/src/main/java/io/casehub/drafthouse/DebateSession.java`
- Modify: `server/api/src/main/java/io/casehub/drafthouse/DebateSessionSnapshot.java`
- Test: `server/api/src/test/java/io/casehub/drafthouse/SelectionThreadTest.java`

**Interfaces:**
- Produces: `SelectionThread(String threadId, SelectionScope anchor, ThreadStatus status)`
- Produces: `ThreadStatus { OPEN, RESOLVED }`
- Produces: `ThreadEntry(String threadId, String sender, String content, String agentRole, Instant timestamp)`
- Produces: `DebateSession.startThread(SelectionScope anchor) → String`
- Produces: `DebateSession.resolveThread(String threadId)`
- Produces: `DebateSession.findThreadsNear(SelectionScope scope) → List<SelectionThread>`
- Produces: `DebateSession.threads() → Map<String, SelectionThread>`

- [ ] **Step 1: Write tests for SelectionThread and thread operations on DebateSession**

```java
// SelectionThreadTest.java
package io.casehub.drafthouse;

import io.casehub.drafthouse.debate.AgentType;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

class SelectionThreadTest {

    @Test
    void startThread_createsOpenThread() {
        DebateSession session = new DebateSession(
                UUID.randomUUID(), "test-session", "test/channel", "agent-1");
        SelectionScope anchor = new SelectionScope(DocumentSide.A, 10, 15, "selected text");

        String threadId = session.startThread(anchor);

        assertNotNull(threadId);
        SelectionThread thread = session.threads().get(threadId);
        assertNotNull(thread);
        assertEquals(ThreadStatus.OPEN, thread.status());
        assertEquals(anchor, thread.anchor());
    }

    @Test
    void resolveThread_setsStatusToResolved() {
        DebateSession session = new DebateSession(
                UUID.randomUUID(), "test-session", "test/channel", "agent-1");
        SelectionScope anchor = new SelectionScope(DocumentSide.A, 10, 15, "selected text");
        String threadId = session.startThread(anchor);

        session.resolveThread(threadId);

        assertEquals(ThreadStatus.RESOLVED, session.threads().get(threadId).status());
    }

    @Test
    void resolveThread_unknownId_throws() {
        DebateSession session = new DebateSession(
                UUID.randomUUID(), "test-session", "test/channel", "agent-1");
        assertThrows(IllegalArgumentException.class, () -> session.resolveThread("bogus"));
    }

    @Test
    void findThreadsNear_overlapping_returnsMatch() {
        DebateSession session = new DebateSession(
                UUID.randomUUID(), "test-session", "test/channel", "agent-1");
        SelectionScope anchor = new SelectionScope(DocumentSide.A, 10, 15, "first");
        session.startThread(anchor);

        SelectionScope query = new SelectionScope(DocumentSide.A, 12, 18, "overlap");
        List<SelectionThread> nearby = session.findThreadsNear(query);

        assertEquals(1, nearby.size());
    }

    @Test
    void findThreadsNear_differentSide_returnsEmpty() {
        DebateSession session = new DebateSession(
                UUID.randomUUID(), "test-session", "test/channel", "agent-1");
        session.startThread(new SelectionScope(DocumentSide.A, 10, 15, "first"));

        SelectionScope query = new SelectionScope(DocumentSide.B, 10, 15, "same lines, different side");
        List<SelectionThread> nearby = session.findThreadsNear(query);

        assertTrue(nearby.isEmpty());
    }

    @Test
    void findThreadsNear_noOverlap_returnsEmpty() {
        DebateSession session = new DebateSession(
                UUID.randomUUID(), "test-session", "test/channel", "agent-1");
        session.startThread(new SelectionScope(DocumentSide.A, 10, 15, "first"));

        SelectionScope query = new SelectionScope(DocumentSide.A, 20, 25, "no overlap");
        List<SelectionThread> nearby = session.findThreadsNear(query);

        assertTrue(nearby.isEmpty());
    }

    @Test
    void threads_includedInSnapshot() {
        DebateSession session = new DebateSession(
                UUID.randomUUID(), "test-session", "test/channel", "agent-1");
        session.addDocument("/path/to/spec.md", "spec");
        String threadId = session.startThread(new SelectionScope(DocumentSide.A, 1, 5, "text"));

        DebateSessionSnapshot snapshot = session.snapshot();

        assertNotNull(snapshot.threads());
        assertEquals(1, snapshot.threads().size());
        assertTrue(snapshot.threads().containsKey(threadId));
    }

    @Test
    void fromSnapshot_restoresThreads() {
        DebateSession original = new DebateSession(
                UUID.randomUUID(), "test-session", "test/channel", "agent-1");
        original.addDocument("/path/to/spec.md", "spec");
        String threadId = original.startThread(new SelectionScope(DocumentSide.A, 1, 5, "text"));

        DebateSessionSnapshot snapshot = original.snapshot();
        DebateSession restored = DebateSession.fromSnapshot(snapshot);

        assertEquals(1, restored.threads().size());
        SelectionThread thread = restored.threads().get(threadId);
        assertNotNull(thread);
        assertEquals(ThreadStatus.OPEN, thread.status());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl api -Dtest=SelectionThreadTest`
Expected: Compilation failure — `SelectionThread`, `ThreadStatus`, `ThreadEntry` not found

- [ ] **Step 3: Create ThreadStatus enum**

```java
package io.casehub.drafthouse;

public enum ThreadStatus { OPEN, RESOLVED }
```

- [ ] **Step 4: Create SelectionThread record**

```java
package io.casehub.drafthouse;

import java.util.Objects;

public record SelectionThread(String threadId, SelectionScope anchor, ThreadStatus status) {
    public SelectionThread {
        Objects.requireNonNull(threadId, "threadId");
        Objects.requireNonNull(anchor, "anchor");
        Objects.requireNonNull(status, "status");
    }

    public SelectionThread withStatus(ThreadStatus newStatus) {
        return new SelectionThread(threadId, anchor, newStatus);
    }
}
```

- [ ] **Step 5: Create ThreadEntry record**

```java
package io.casehub.drafthouse;

import java.time.Instant;
import java.util.Objects;

public record ThreadEntry(String threadId, String sender, String content,
                           String agentRole, Instant timestamp) {
    public ThreadEntry {
        Objects.requireNonNull(threadId, "threadId");
        Objects.requireNonNull(content, "content");
    }
}
```

- [ ] **Step 6: Add thread operations to DebateSession**

Add to `DebateSession`:
- Field: `private final ConcurrentHashMap<String, SelectionThread> threads = new ConcurrentHashMap<>();`
- Method: `startThread(SelectionScope anchor)` — creates UUID, puts OPEN thread, returns ID
- Method: `resolveThread(String threadId)` — validates exists, replaces with RESOLVED status
- Method: `findThreadsNear(SelectionScope scope)` — filters by same side + line overlap
- Method: `threads()` — returns `Collections.unmodifiableMap(threads)`
- Update `fromSnapshot()` to restore threads from snapshot
- Update `snapshot()` to include threads

- [ ] **Step 7: Update DebateSessionSnapshot to include threads**

Add `Map<String, SelectionThread> threads` as a new record component. Update the constructor call in `DebateSession.snapshot()` to pass `Map.copyOf(threads)`. Update `DebateSession.fromSnapshot()` to restore threads.

Note: this is a record component addition — all existing callers that construct `DebateSessionSnapshot` must be updated. Use `ide_find_references` on the constructor to find all call sites (including `JpaDebateSessionStore`, `NoOpDebateSessionStore`, test code).

- [ ] **Step 8: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl api -Dtest=SelectionThreadTest`
Expected: All 7 tests PASS

- [ ] **Step 9: Fix any callers broken by DebateSessionSnapshot change**

Use `ide_find_references` on `DebateSessionSnapshot` constructor. Update each call site to pass `Map.of()` for threads where no thread state exists (e.g., `JpaDebateSessionStore`, `NoOpDebateSessionStore`, `DebateSessionStoreContractTest`).

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl api`
Expected: All existing tests still pass

- [ ] **Step 10: Commit**

```
feat(#60): add SelectionThread domain model and DebateSession thread operations

Refs #60
```

---

### Task 2: ThreadProjection — Channel Projection for Thread State

**Files:**
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/debate/ThreadProjection.java`
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/debate/ThreadState.java`
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/debate/ThreadView.java`
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/debate/DebateChannelProjection.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/debate/ThreadProjectionTest.java`

**Interfaces:**
- Consumes: `SelectionThread`, `ThreadStatus`, `ThreadEntry`, `SelectionScope`, `DocumentSide` from Task 1
- Consumes: `DebateProtocol.META_SENTINEL`, `DebateProtocol.parseMeta()`, `DebateProtocol.bodyContent()` (existing)
- Produces: `ThreadProjection implements ChannelProjection<ThreadState>`
- Produces: `ThreadState` — holds `Map<String, ThreadView>`
- Produces: `ThreadView` — holds threadId, anchor, status, entries list, createdBy

- [ ] **Step 1: Write tests for ThreadProjection**

```java
package io.casehub.drafthouse.debate;

import io.casehub.blocks.channel.ChannelMessageMeta;
import io.casehub.blocks.conversation.ConversationProtocol;
import io.casehub.drafthouse.DocumentSide;
import io.casehub.drafthouse.ThreadStatus;
import io.casehub.qhorus.api.message.MessageType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class ThreadProjectionTest {

    private ThreadProjection projection;

    @BeforeEach
    void setUp() {
        projection = new ThreadProjection();
    }

    @Test
    void start_createsThread() {
        ThreadState state = projection.initialState();
        state = projection.apply(state, threadMessage("START", "t1", "REV",
                "Initial comment", Map.of(
                        "side", "A", "startLine", "10", "endLine", "15",
                        "selectedText", "hello world")));

        assertEquals(1, state.threads().size());
        ThreadView view = state.threads().get("t1");
        assertNotNull(view);
        assertEquals(ThreadStatus.OPEN, view.status());
        assertEquals(DocumentSide.A, view.anchor().side());
        assertEquals(10, view.anchor().startLine());
        assertEquals(1, view.entries().size());
    }

    @Test
    void reply_addsEntry() {
        ThreadState state = projection.initialState();
        state = projection.apply(state, threadMessage("START", "t1", "REV",
                "Initial", Map.of("side", "A", "startLine", "10", "endLine", "15",
                        "selectedText", "text")));
        state = projection.apply(state, threadMessage("REPLY", "t1", "HUMAN",
                "Good point", Map.of()));

        assertEquals(2, state.threads().get("t1").entries().size());
    }

    @Test
    void resolve_updatesStatus() {
        ThreadState state = projection.initialState();
        state = projection.apply(state, threadMessage("START", "t1", "REV",
                "comment", Map.of("side", "A", "startLine", "1", "endLine", "5",
                        "selectedText", "x")));
        state = projection.apply(state, threadMessage("RESOLVE", "t1", "HUMAN",
                "", Map.of()));

        assertEquals(ThreadStatus.RESOLVED, state.threads().get("t1").status());
    }

    @Test
    void nonThreadMessage_skipped() {
        ThreadState state = projection.initialState();
        // A regular debate message (no threadId) should be ignored
        state = projection.apply(state, debateMessage("RAISE", "REV", "some point"));

        assertTrue(state.threads().isEmpty());
    }

    @Test
    void malformedStart_noSide_skipped() {
        ThreadState state = projection.initialState();
        // START without anchor fields — should be skipped, not throw
        state = projection.apply(state, threadMessage("START", "t1", "REV",
                "bad", Map.of()));

        assertTrue(state.threads().isEmpty());
    }

    private MessageView threadMessage(String action, String threadId,
                                       String role, String content,
                                       Map<String, String> extraMeta) {
        Map<String, String> meta = new LinkedHashMap<>();
        meta.put("threadId", threadId);
        meta.put("threadAction", action);
        meta.put(ConversationProtocol.ROLE, role);
        meta.putAll(extraMeta);
        String encoded = ChannelMessageMeta.encode(DebateProtocol.META_SENTINEL, meta, content);
        return new TestMessageView(encoded, "sender-1", UUID.randomUUID().toString(),
                MessageType.QUERY, 1L, Instant.now());
    }

    private MessageView debateMessage(String entryType, String role, String content) {
        Map<String, String> meta = new LinkedHashMap<>();
        meta.put(ConversationProtocol.ENTRY_TYPE, entryType);
        meta.put(ConversationProtocol.ROLE, role);
        meta.put(ConversationProtocol.ROUND, "1");
        meta.put(ConversationProtocol.PRIORITY, "HIGH");
        meta.put(ConversationProtocol.SCOPE, "ISOLATED");
        String encoded = ChannelMessageMeta.encode(DebateProtocol.META_SENTINEL, meta, content);
        return new TestMessageView(encoded, "sender-1", UUID.randomUUID().toString(),
                MessageType.QUERY, 1L, Instant.now());
    }

    // TestMessageView — minimal MessageView implementation for unit tests
    // Check if one already exists in the test tree; if so, reuse it
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=ThreadProjectionTest`
Expected: Compilation failure — `ThreadProjection`, `ThreadState`, `ThreadView` not found

- [ ] **Step 3: Create ThreadState and ThreadView records**

```java
// ThreadState.java
package io.casehub.drafthouse.debate;

import java.util.Map;

public record ThreadState(Map<String, ThreadView> threads) {
    public static ThreadState empty() {
        return new ThreadState(Map.of());
    }
}
```

```java
// ThreadView.java
package io.casehub.drafthouse.debate;

import io.casehub.drafthouse.SelectionScope;
import io.casehub.drafthouse.ThreadEntry;
import io.casehub.drafthouse.ThreadStatus;
import java.util.List;

public record ThreadView(
        String threadId,
        SelectionScope anchor,
        ThreadStatus status,
        List<ThreadEntry> entries,
        String createdBy) {}
```

- [ ] **Step 4: Implement ThreadProjection**

```java
package io.casehub.drafthouse.debate;

import io.casehub.blocks.channel.ChannelMessageMeta;
import io.casehub.blocks.conversation.ConversationProtocol;
import io.casehub.drafthouse.DocumentSide;
import io.casehub.drafthouse.SelectionScope;
import io.casehub.drafthouse.ThreadEntry;
import io.casehub.drafthouse.ThreadStatus;
import io.casehub.qhorus.api.projection.ChannelProjection;
import io.casehub.qhorus.api.projection.MessageView;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.logging.Level;
import java.util.logging.Logger;

@ApplicationScoped
public class ThreadProjection implements ChannelProjection<ThreadState> {

    private static final Logger LOG = Logger.getLogger(ThreadProjection.class.getName());

    @Override
    public String projectionName() { return "thread"; }

    @Override
    public ThreadState initialState() { return ThreadState.empty(); }

    @Override
    public ThreadState apply(ThreadState state, MessageView message) {
        try {
            Map<String, String> meta = ChannelMessageMeta.parseMeta(
                    DebateProtocol.META_SENTINEL, message.content());
            String threadId = meta.get("threadId");
            if (threadId == null) return state; // not a thread message

            String action = meta.get("threadAction");
            if (action == null) {
                LOG.warning("Thread message missing threadAction — discarded");
                return state;
            }

            String role = meta.get(ConversationProtocol.ROLE);
            String body = ChannelMessageMeta.bodyContent(
                    DebateProtocol.META_SENTINEL, message.content());

            return switch (action) {
                case "START" -> applyStart(state, threadId, role, body, meta, message);
                case "REPLY" -> applyReply(state, threadId, role, body, message);
                case "RESOLVE" -> applyResolve(state, threadId);
                default -> {
                    LOG.warning("Unknown threadAction: " + action + " — discarded");
                    yield state;
                }
            };
        } catch (Exception e) {
            LOG.log(Level.WARNING, "ThreadProjection.apply() failed — discarded", e);
            return state;
        }
    }

    private ThreadState applyStart(ThreadState state, String threadId,
                                    String role, String body,
                                    Map<String, String> meta, MessageView msg) {
        String sideStr = meta.get("side");
        String startStr = meta.get("startLine");
        String endStr = meta.get("endLine");
        String selectedText = meta.get("selectedText");

        if (sideStr == null || startStr == null || endStr == null || selectedText == null) {
            LOG.warning("Thread START missing anchor fields — discarded");
            return state;
        }

        DocumentSide side;
        try { side = DocumentSide.valueOf(sideStr); }
        catch (IllegalArgumentException e) {
            LOG.warning("Thread START invalid side: " + sideStr + " — discarded");
            return state;
        }

        int startLine, endLine;
        try {
            startLine = Integer.parseInt(startStr);
            endLine = Integer.parseInt(endStr);
        } catch (NumberFormatException e) {
            LOG.warning("Thread START invalid line numbers — discarded");
            return state;
        }

        SelectionScope anchor = new SelectionScope(side, startLine, endLine, selectedText);
        ThreadEntry entry = new ThreadEntry(threadId, msg.sender(), body, role, msg.createdAt());
        ThreadView view = new ThreadView(threadId, anchor, ThreadStatus.OPEN,
                List.of(entry), role);

        Map<String, ThreadView> updated = new LinkedHashMap<>(state.threads());
        updated.put(threadId, view);
        return new ThreadState(Map.copyOf(updated));
    }

    private ThreadState applyReply(ThreadState state, String threadId,
                                    String role, String body, MessageView msg) {
        ThreadView existing = state.threads().get(threadId);
        if (existing == null) {
            LOG.warning("Thread REPLY for unknown thread " + threadId + " — discarded");
            return state;
        }

        ThreadEntry entry = new ThreadEntry(threadId, msg.sender(), body, role, msg.createdAt());
        List<ThreadEntry> entries = new ArrayList<>(existing.entries());
        entries.add(entry);
        ThreadView updated = new ThreadView(existing.threadId(), existing.anchor(),
                existing.status(), List.copyOf(entries), existing.createdBy());

        Map<String, ThreadView> threads = new LinkedHashMap<>(state.threads());
        threads.put(threadId, updated);
        return new ThreadState(Map.copyOf(threads));
    }

    private ThreadState applyResolve(ThreadState state, String threadId) {
        ThreadView existing = state.threads().get(threadId);
        if (existing == null) {
            LOG.warning("Thread RESOLVE for unknown thread " + threadId + " — discarded");
            return state;
        }

        ThreadView resolved = new ThreadView(existing.threadId(), existing.anchor(),
                ThreadStatus.RESOLVED, existing.entries(), existing.createdBy());

        Map<String, ThreadView> threads = new LinkedHashMap<>(state.threads());
        threads.put(threadId, resolved);
        return new ThreadState(Map.copyOf(threads));
    }
}
```

- [ ] **Step 5: Add threadId skip filter to DebateChannelProjection.apply()**

At the top of `DebateChannelProjection.apply()`, before the existing `ROUND_SNAPSHOT` check:

```java
Map<String, String> meta = ChannelMessageMeta.parseMeta(sentinel(), message.content());
if (meta.containsKey("threadId")) {
    return state; // thread message — handled by ThreadProjection
}
```

Note: the existing code already parses meta in the try block. Move the `meta` parse before the try block and add the threadId check, or add the check inside the existing try block before the `ROUND_SNAPSHOT` check.

- [ ] **Step 6: Run tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=ThreadProjectionTest`
Expected: All tests PASS

Also run existing debate tests to verify no regression:
Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DebateChannelProjectionTest`

- [ ] **Step 7: Commit**

```
feat(#60): add ThreadProjection and thread-aware skip filter in DebateChannelProjection

Refs #60
```

---

### Task 3: ThreadStreamEntry + WebSocket Event Routing

**Files:**
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/debate/ThreadStreamEntry.java`
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/DebateChannelBackend.java`
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/WebSocketEventBus.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/debate/ThreadStreamEntryTest.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/DebateChannelBackendTest.java` (modify existing)

**Interfaces:**
- Consumes: `DebateProtocol.parseMeta()`, `DebateProtocol.bodyContent()` (existing)
- Produces: `ThreadStreamEntry` record with `from(OutboundMessage)` factory
- Produces: `WebSocketEventBus.pushThreadEntries(UUID channelId, List<ThreadStreamEntry> entries)`

- [ ] **Step 1: Write test for ThreadStreamEntry.from()**

```java
package io.casehub.drafthouse.debate;

import io.casehub.blocks.channel.ChannelMessageMeta;
import io.casehub.blocks.conversation.ConversationProtocol;
import org.junit.jupiter.api.Test;

import java.util.LinkedHashMap;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class ThreadStreamEntryTest {

    @Test
    void from_startMessage_parsesAllFields() {
        Map<String, String> meta = new LinkedHashMap<>();
        meta.put("threadId", "t-123");
        meta.put("threadAction", "START");
        meta.put(ConversationProtocol.ROLE, "REV");
        meta.put("side", "A");
        meta.put("startLine", "10");
        meta.put("endLine", "15");
        meta.put("selectedText", "hello world");
        String encoded = ChannelMessageMeta.encode(DebateProtocol.META_SENTINEL, meta, "My comment");

        ThreadStreamEntry entry = ThreadStreamEntry.from(encoded, "sender-1");

        assertNotNull(entry);
        assertEquals("t-123", entry.threadId());
        assertEquals("START", entry.threadAction());
        assertEquals("REV", entry.agentRole());
        assertEquals("My comment", entry.content());
        assertEquals("A", entry.anchor().side());
        assertEquals(10, entry.anchor().startLine());
    }

    @Test
    void from_replyMessage_noAnchor() {
        Map<String, String> meta = new LinkedHashMap<>();
        meta.put("threadId", "t-123");
        meta.put("threadAction", "REPLY");
        meta.put(ConversationProtocol.ROLE, "HUMAN");
        String encoded = ChannelMessageMeta.encode(DebateProtocol.META_SENTINEL, meta, "Great point");

        ThreadStreamEntry entry = ThreadStreamEntry.from(encoded, "sender-2");

        assertNotNull(entry);
        assertEquals("REPLY", entry.threadAction());
        assertNull(entry.anchor());
    }

    @Test
    void from_nonThreadMessage_returnsNull() {
        Map<String, String> meta = new LinkedHashMap<>();
        meta.put(ConversationProtocol.ENTRY_TYPE, "RAISE");
        meta.put(ConversationProtocol.ROLE, "REV");
        meta.put(ConversationProtocol.ROUND, "1");
        String encoded = ChannelMessageMeta.encode(DebateProtocol.META_SENTINEL, meta, "debate point");

        ThreadStreamEntry entry = ThreadStreamEntry.from(encoded, "sender-1");

        assertNull(entry);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=ThreadStreamEntryTest`
Expected: Compilation failure

- [ ] **Step 3: Create ThreadStreamEntry record**

```java
package io.casehub.drafthouse.debate;

import io.casehub.blocks.channel.ChannelMessageMeta;
import io.casehub.blocks.conversation.ConversationProtocol;

import java.time.Instant;
import java.util.Map;

public record ThreadStreamEntry(
        String threadId,
        String threadAction,
        String content,
        String agentRole,
        String sender,
        Instant timestamp,
        Anchor anchor) {

    public record Anchor(String side, int startLine, int endLine, String selectedText) {}

    public static ThreadStreamEntry from(String encodedContent, String sender) {
        Map<String, String> meta = DebateProtocol.parseMeta(encodedContent);
        String threadId = meta.get("threadId");
        if (threadId == null) return null;

        String action = meta.get("threadAction");
        if (action == null) return null;

        String role = meta.get(ConversationProtocol.ROLE);
        String body = DebateProtocol.bodyContent(encodedContent);

        Anchor anchor = null;
        if ("START".equals(action)) {
            String side = meta.get("side");
            String startLine = meta.get("startLine");
            String endLine = meta.get("endLine");
            String selectedText = meta.get("selectedText");
            if (side != null && startLine != null && endLine != null && selectedText != null) {
                try {
                    anchor = new Anchor(side, Integer.parseInt(startLine),
                            Integer.parseInt(endLine), selectedText);
                } catch (NumberFormatException ignored) {}
            }
        }

        return new ThreadStreamEntry(threadId, action, body, role, sender, Instant.now(), anchor);
    }

    public static ThreadStreamEntry from(io.casehub.qhorus.api.gateway.OutboundMessage msg) {
        if (msg.content() == null) return null;
        return from(msg.content(), msg.sender());
    }
}
```

- [ ] **Step 4: Add pushThreadEntries to WebSocketEventBus**

Add method to `WebSocketEventBus`:

```java
public void pushThreadEntries(UUID channelId, List<ThreadStreamEntry> entries) {
    String json = formatEvent("thread-entries", entries);
    topicRegistry.forSession(channelId, conn -> sendSafe(conn, json));
}
```

- [ ] **Step 5: Update DebateChannelBackend.post() for thread-aware routing**

In `DebateChannelBackend.post()`, before the existing `DebateStreamEntry.from()` call, check for thread messages:

```java
@Override
public void post(ChannelRef channel, OutboundMessage message) {
    Map<String, String> meta = DebateProtocol.parseMeta(message.content());

    // Thread messages → thread event path
    if (meta.containsKey("threadId")) {
        ThreadStreamEntry threadEntry = ThreadStreamEntry.from(message);
        if (threadEntry != null) {
            eventBus.pushThreadEntries(channel.id(), java.util.List.of(threadEntry));
            // Push metadata events for lifecycle changes
            String action = meta.get("threadAction");
            if ("START".equals(action)) {
                eventBus.pushMetadata(channel.id(), "thread-created",
                        java.util.Map.of("threadId", meta.get("threadId"),
                                "anchor", threadEntry.anchor() != null ? threadEntry.anchor() : "",
                                "createdBy", threadEntry.agentRole()));
            } else if ("RESOLVE".equals(action)) {
                eventBus.pushMetadata(channel.id(), "thread-resolved",
                        java.util.Map.of("threadId", meta.get("threadId")));
            }
        }
        return; // don't fall through to debate entry path
    }

    // Existing debate entry path unchanged
    DebateStreamEntry entry = DebateStreamEntry.from(message);
    // ... rest unchanged
}
```

- [ ] **Step 6: Run tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=ThreadStreamEntryTest`
Expected: PASS

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DebateChannelBackendTest`
Expected: Existing tests still PASS

- [ ] **Step 7: Commit**

```
feat(#60): add ThreadStreamEntry and thread-aware WebSocket routing in DebateChannelBackend

Refs #60
```

---

### Task 4: ThreadMcpTools — MCP Tool Surface

**Files:**
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/ThreadMcpTools.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/ThreadMcpToolsTest.java`

**Interfaces:**
- Consumes: `DebateSession.startThread()`, `DebateSession.resolveThread()`, `DebateSession.findThreadsNear()`, `DebateSession.threads()` from Task 1
- Consumes: `DebateProtocol.META_SENTINEL`, `ChannelMessageMeta.encode()` (existing)
- Consumes: `ThreadProjection` from Task 2
- Consumes: `DebateSessionRegistry`, `MessageService`, `ProjectionService`, `InstanceService`, `DebateParticipants` (existing)
- Produces: MCP tools `start_thread`, `reply_to_thread`, `resolve_thread`, `get_thread_summary`

- [ ] **Step 1: Write test for start_thread**

Test that `start_thread` creates a thread on the session, dispatches a message to the channel, and returns JSON with threadId and nearbyThreads. Use `@QuarkusTest` with the existing test infrastructure.

Check the existing `DebateMcpToolsTest` pattern (if one exists) for the test setup approach. If no integration test exists, write a unit test with mocks for the injected services.

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement ThreadMcpTools**

```java
package io.casehub.drafthouse;

import io.casehub.blocks.channel.ChannelMessageMeta;
import io.casehub.blocks.conversation.ConversationProtocol;
import io.casehub.drafthouse.debate.AgentType;
import io.casehub.drafthouse.debate.DebateProtocol;
import io.casehub.drafthouse.debate.ThreadProjection;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.instance.InstanceService;
import io.casehub.qhorus.runtime.message.MessageService;
import io.casehub.qhorus.runtime.message.ProjectionService;
import io.quarkiverse.mcp.server.Tool;
import io.quarkiverse.mcp.server.ToolArg;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.LinkedHashMap;
import java.util.Map;
import java.util.UUID;
import java.util.logging.Logger;
import java.util.stream.Collectors;

@ApplicationScoped
public class ThreadMcpTools {

    private static final Logger LOG = Logger.getLogger(ThreadMcpTools.class.getName());

    @Inject DebateSessionRegistry registry;
    @Inject MessageService messageService;
    @Inject InstanceService instanceService;
    @Inject ProjectionService projectionService;
    @Inject ThreadProjection threadProjection;

    @Tool(name = "start_thread",
          description = "Start a selection-scoped conversation thread on a debate session. "
                        + "Returns threadId and any nearby existing threads.")
    public String startThread(
            @ToolArg(description = "debateSessionId returned by start_debate") String debateSessionId,
            @ToolArg(description = "Your agent role: REV | IMP | SUPERVISOR | MODERATOR | SELECTOR | HUMAN") String agentRole,
            @ToolArg(description = "Document side: A or B") String side,
            @ToolArg(description = "Start line of the selection") int startLine,
            @ToolArg(description = "End line of the selection") int endLine,
            @ToolArg(description = "The selected text") String selectedText,
            @ToolArg(description = "Initial thread comment") String content) {

        DebateSession session = resolveSession(debateSessionId);
        if (session == null) return sessionError(debateSessionId);

        AgentType role = parseRole(agentRole);
        if (role == null) return roleError(agentRole);

        DocumentSide docSide;
        try { docSide = DocumentSide.valueOf(side); }
        catch (IllegalArgumentException e) { return "error: invalid side: " + side; }

        SelectionScope anchor;
        try { anchor = new SelectionScope(docSide, startLine, endLine, selectedText); }
        catch (IllegalArgumentException e) { return "error: " + e.getMessage(); }

        String threadId = session.startThread(anchor);
        registry.persist(session);

        // Encode and dispatch
        Map<String, String> meta = new LinkedHashMap<>();
        meta.put("threadId", threadId);
        meta.put("threadAction", "START");
        meta.put(ConversationProtocol.ROLE, agentRole);
        meta.put("side", side);
        meta.put("startLine", String.valueOf(startLine));
        meta.put("endLine", String.valueOf(endLine));
        meta.put("selectedText", selectedText);
        String encoded = ChannelMessageMeta.encode(DebateProtocol.META_SENTINEL, meta, content);

        String sender = DebateParticipants.ensureSender(session, role, instanceService, registry);
        messageService.dispatch(MessageDispatch.builder()
                .channelId(session.channelId())
                .sender(sender)
                .type(MessageType.QUERY)
                .content(encoded)
                .correlationId(threadId)
                .actorType(role == AgentType.HUMAN ? ActorType.HUMAN : ActorType.AGENT)
                .build());

        // Find nearby threads for the response
        var nearby = session.findThreadsNear(anchor).stream()
                .filter(t -> !t.threadId().equals(threadId))
                .map(t -> "{\"threadId\":\"" + t.threadId()
                           + "\",\"status\":\"" + t.status()
                           + "\",\"startLine\":" + t.anchor().startLine()
                           + ",\"endLine\":" + t.anchor().endLine() + "}")
                .collect(Collectors.joining(","));

        return "{\"threadId\":\"" + threadId + "\",\"status\":\"created\""
               + ",\"nearbyThreads\":[" + nearby + "]}";
    }

    @Tool(name = "reply_to_thread",
          description = "Reply to an existing selection-scoped thread.")
    public String replyToThread(
            @ToolArg(description = "debateSessionId") String debateSessionId,
            @ToolArg(description = "Your agent role") String agentRole,
            @ToolArg(description = "threadId from start_thread") String threadId,
            @ToolArg(description = "Reply content") String content) {

        DebateSession session = resolveSession(debateSessionId);
        if (session == null) return sessionError(debateSessionId);

        AgentType role = parseRole(agentRole);
        if (role == null) return roleError(agentRole);

        SelectionThread thread = session.threads().get(threadId);
        if (thread == null) return "error: thread not found: " + threadId;
        if (thread.status() == ThreadStatus.RESOLVED) return "error: thread is resolved: " + threadId;

        Map<String, String> meta = new LinkedHashMap<>();
        meta.put("threadId", threadId);
        meta.put("threadAction", "REPLY");
        meta.put(ConversationProtocol.ROLE, agentRole);
        String encoded = ChannelMessageMeta.encode(DebateProtocol.META_SENTINEL, meta, content);

        String sender = DebateParticipants.ensureSender(session, role, instanceService, registry);
        messageService.dispatch(MessageDispatch.builder()
                .channelId(session.channelId())
                .sender(sender)
                .type(MessageType.RESPONSE)
                .content(encoded)
                .correlationId(threadId)
                .actorType(role == AgentType.HUMAN ? ActorType.HUMAN : ActorType.AGENT)
                .build());

        return "{\"status\":\"dispatched\"}";
    }

    @Tool(name = "resolve_thread",
          description = "Resolve (close) a selection-scoped thread.")
    public String resolveThread(
            @ToolArg(description = "debateSessionId") String debateSessionId,
            @ToolArg(description = "Your agent role") String agentRole,
            @ToolArg(description = "threadId to resolve") String threadId) {

        DebateSession session = resolveSession(debateSessionId);
        if (session == null) return sessionError(debateSessionId);

        AgentType role = parseRole(agentRole);
        if (role == null) return roleError(agentRole);

        try { session.resolveThread(threadId); }
        catch (IllegalArgumentException e) { return "error: " + e.getMessage(); }
        registry.persist(session);

        Map<String, String> meta = new LinkedHashMap<>();
        meta.put("threadId", threadId);
        meta.put("threadAction", "RESOLVE");
        meta.put(ConversationProtocol.ROLE, agentRole);
        String encoded = ChannelMessageMeta.encode(DebateProtocol.META_SENTINEL, meta, "");

        String sender = DebateParticipants.ensureSender(session, role, instanceService, registry);
        messageService.dispatch(MessageDispatch.builder()
                .channelId(session.channelId())
                .sender(sender)
                .type(MessageType.DONE)
                .content(encoded)
                .correlationId(threadId)
                .actorType(role == AgentType.HUMAN ? ActorType.HUMAN : ActorType.AGENT)
                .build());

        return "{\"status\":\"resolved\"}";
    }

    @Tool(name = "get_thread_summary",
          description = "Get thread summary for a debate session. Pass threadId for a single thread, or omit for all threads.")
    public String getThreadSummary(
            @ToolArg(description = "debateSessionId") String debateSessionId,
            @ToolArg(description = "Optional threadId. Omit for all threads.") String threadId) {

        DebateSession session = resolveSession(debateSessionId);
        if (session == null) return sessionError(debateSessionId);

        var result = projectionService.project(session.channelId(), threadProjection);
        var threads = result.state().threads();

        if (threadId != null && !threadId.isBlank()) {
            var view = threads.get(threadId);
            if (view == null) return "error: thread not found: " + threadId;
            return renderThreadView(view);
        }

        if (threads.isEmpty()) return "{\"threads\":[],\"count\":0}";

        String threadsJson = threads.values().stream()
                .map(this::renderThreadCompact)
                .collect(Collectors.joining(","));
        return "{\"threads\":[" + threadsJson + "],\"count\":" + threads.size() + "}";
    }

    private String renderThreadView(io.casehub.drafthouse.debate.ThreadView view) {
        String entriesJson = view.entries().stream()
                .map(e -> "{\"agentRole\":\"" + e.agentRole()
                           + "\",\"content\":" + DraftHouseMcpTools.jsonString(e.content())
                           + ",\"timestamp\":\"" + e.timestamp() + "\"}")
                .collect(Collectors.joining(","));
        return "{\"threadId\":\"" + view.threadId()
               + "\",\"status\":\"" + view.status()
               + "\",\"anchor\":{\"side\":\"" + view.anchor().side()
               + "\",\"startLine\":" + view.anchor().startLine()
               + ",\"endLine\":" + view.anchor().endLine() + "}"
               + ",\"entries\":[" + entriesJson + "]}";
    }

    private String renderThreadCompact(io.casehub.drafthouse.debate.ThreadView view) {
        return "{\"threadId\":\"" + view.threadId()
               + "\",\"status\":\"" + view.status()
               + "\",\"entryCount\":" + view.entries().size()
               + ",\"anchor\":{\"side\":\"" + view.anchor().side()
               + "\",\"startLine\":" + view.anchor().startLine()
               + ",\"endLine\":" + view.anchor().endLine() + "}}";
    }

    private DebateSession resolveSession(String debateSessionId) {
        try {
            UUID channelId = UUID.fromString(debateSessionId);
            return registry.find(channelId).orElse(null);
        } catch (IllegalArgumentException e) { return null; }
    }

    private String sessionError(String id) { return "error: no active debate session for: " + id; }

    private String roleError(String role) { return "error: invalid agentRole: " + role; }

    private static AgentType parseRole(String agentRole) {
        try { return AgentType.valueOf(agentRole); }
        catch (IllegalArgumentException e) { return null; }
    }
}
```

- [ ] **Step 4: Run tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=ThreadMcpToolsTest`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(#60): add ThreadMcpTools — start_thread, reply_to_thread, resolve_thread, get_thread_summary

Refs #60
```

---

### Task 5: REST Endpoints for Browser-Initiated Thread Actions

**Files:**
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/HumanActionResource.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/HumanActionResourceThreadTest.java`

**Interfaces:**
- Consumes: `DebateSession.startThread()`, `DebateSession.resolveThread()`, `DebateSession.threads()` from Task 1
- Consumes: `DebateParticipants.ensureSender()`, `MessageService`, `InstanceService` (existing)
- Produces: `POST /api/debate/{id}/human/thread` — start thread
- Produces: `POST /api/debate/{id}/human/thread/{threadId}/reply` — reply
- Produces: `POST /api/debate/{id}/human/thread/{threadId}/resolve` — resolve

- [ ] **Step 1: Write test for thread REST endpoints**

Write integration test verifying:
- POST /thread with valid selection → 200 with threadId
- POST /thread/{id}/reply → 200
- POST /thread/{id}/resolve → 200
- POST /thread/{bogus}/reply → 400 "thread not found"
- POST /thread/{resolved}/reply → 400 "thread is resolved"

Use the same test setup pattern as existing `HumanActionResource` tests (check if integration tests exist, otherwise use `@QuarkusTest`).

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Add request records and endpoints to HumanActionResource**

Add to `HumanActionResource`:

```java
record ThreadStartRequest(String content, String priority, String side,
                           int startLine, int endLine, String selectedText) {}
record ThreadReplyRequest(String content) {}

@POST @Path("/thread")
@Consumes(MediaType.APPLICATION_JSON) @Produces(MediaType.APPLICATION_JSON)
public Response startThread(@PathParam("debateSessionId") String debateSessionId,
                             ThreadStartRequest request) { ... }

@POST @Path("/thread/{threadId}/reply")
@Consumes(MediaType.APPLICATION_JSON) @Produces(MediaType.APPLICATION_JSON)
public Response replyToThread(@PathParam("debateSessionId") String debateSessionId,
                               @PathParam("threadId") String threadId,
                               ThreadReplyRequest request) { ... }

@POST @Path("/thread/{threadId}/resolve")
@Produces(MediaType.APPLICATION_JSON)
public Response resolveThread(@PathParam("debateSessionId") String debateSessionId,
                               @PathParam("threadId") String threadId) { ... }
```

Each endpoint follows the same pattern as `comment()` and `raise()`:
1. Resolve session
2. Validate inputs
3. Create sender via `DebateParticipants.ensureSender(HUMAN)`
4. Encode with `DHMETA:` sentinel
5. Dispatch via `messageService.dispatch()`
6. Return JSON response

- [ ] **Step 4: Run tests**

- [ ] **Step 5: Commit**

```
feat(#60): add thread REST endpoints on HumanActionResource

Refs #60
```

---

### Task 6: UI — ThreadStreamEntry Type + selection-threads Panel

**Files:**
- Create: `@casehubio/blocks-ui-document-workbench/src/selection-threads.ts`
- Modify: `@casehubio/blocks-ui-document-workbench/src/types.ts`
- Modify: `@casehubio/blocks-ui-document-workbench/src/index.ts`
- Modify: `server/runtime/src/main/webui/src/index.ts` (workbench layout)
- Test: `@casehubio/blocks-ui-document-workbench/src/selection-threads.test.ts`

**Interfaces:**
- Consumes: `thread-entries`, `thread-created`, `thread-resolved` WebSocket events from Task 3
- Consumes: `POST /api/debate/{id}/human/thread`, `/thread/{id}/reply`, `/thread/{id}/resolve` from Task 5
- Produces: `<selection-threads>` LitElement panel
- Produces: `ThreadStreamEntry` TypeScript interface
- Produces: `thread-selected` custom event (dispatched on thread click)

- [ ] **Step 1: Add ThreadStreamEntry to types.ts**

```typescript
export interface ThreadStreamEntry {
  threadId: string;
  threadAction: string;
  content: string;
  agentRole: string;
  timestamp?: string;
  anchor?: {
    side: string;
    startLine: number;
    endLine: number;
    selectedText: string;
  };
}

export interface ThreadInfo {
  threadId: string;
  anchor: { side: string; startLine: number; endLine: number; selectedText: string };
  status: string;
  entries: ThreadStreamEntry[];
  createdBy: string;
}
```

- [ ] **Step 2: Create selection-threads panel**

Create `selection-threads.ts` as a LitElement with Shadow DOM:
- Thread list view showing compact cards
- Thread detail view on click with conversation entries + reply input
- Subscribes to `thread-entries`, `thread-created`, `thread-resolved`, `reconnected`
- Dispatches `thread-selected` custom event on thread click
- `configure(props)` accepts `debateSessionId`
- Reply input POSTs to `/api/debate/{id}/human/thread/{threadId}/reply`
- Resolve button POSTs to `/api/debate/{id}/human/thread/{threadId}/resolve`
- Start thread button POSTs to `/api/debate/{id}/human/thread`

- [ ] **Step 3: Register panel in blocks-ui index.ts**

Export `SelectionThreads` from the package index.

- [ ] **Step 4: Wire into workbench layout**

In `server/runtime/src/main/webui/src/index.ts`:
- Register `<selection-threads>` panel via `registerPanel()`
- Add to the right-column layout alongside debate-feed (tabbed: Debate / Threads)
- Wire `configure({ debateSessionId })` when debate session connects

- [ ] **Step 5: Write tests**

Write test in `selection-threads.test.ts`:
- Renders placeholder when no session configured
- Renders thread list when thread-created events arrive
- Renders thread detail on click
- Auto-scrolls on new entries

- [ ] **Step 6: Build and verify**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml package -DskipTests`
Verify the webui bundles without errors.

- [ ] **Step 7: Commit**

```
feat(#60): add <selection-threads> panel with thread list/detail views

Refs #60
```

---

### Task 7: Diff Panel Integration — Gutter Markers + Bidirectional Navigation

**Files:**
- Modify: `@casehubio/blocks-ui-document-workbench/src/document-diff.ts`

**Interfaces:**
- Consumes: `thread-created`, `thread-resolved` WebSocket events
- Consumes: `thread-focused` custom event from selection-threads panel (Task 6)
- Produces: `thread-selected` custom event when gutter marker clicked
- Produces: Gutter markers at thread anchor line ranges

- [ ] **Step 1: Add thread state tracking to document-diff**

Add `@state()` property for thread anchors:
```typescript
@state() private _threadAnchors: Array<{
  threadId: string;
  side: string;
  startLine: number;
  endLine: number;
  status: string;
}> = [];
```

Subscribe to `thread-created` and `thread-resolved` events in `connectedCallback()`.

- [ ] **Step 2: Render gutter markers**

In the diff panel's render method, add colored markers in the gutter at thread anchor line ranges. Accent color for OPEN threads, neutral for RESOLVED.

- [ ] **Step 3: Add click-to-navigate**

When a gutter marker is clicked, dispatch `thread-selected` custom event with `{threadId}`.

- [ ] **Step 4: Add reverse navigation**

Listen for `thread-focused` custom event. When received, scroll to the anchor's line range and briefly highlight it (CSS animation).

- [ ] **Step 5: Add "Start Thread" button to selection UI**

When user selects text (existing mouseup flow), show a "Start Thread" button alongside any existing selection UI. Clicking it opens a small input area. On submit, POST to `/api/debate/{id}/human/thread`.

- [ ] **Step 6: Build and verify**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml package -DskipTests`

- [ ] **Step 7: Manual E2E verification**

Start the server, open the browser, create a debate, select text, start a thread, reply, resolve. Verify gutter markers appear and bidirectional navigation works.

- [ ] **Step 8: Commit**

```
feat(#60): add diff panel gutter markers and bidirectional thread navigation

Closes #60
```
