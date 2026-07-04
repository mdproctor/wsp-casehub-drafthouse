# WebSocket Push Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace all SSE endpoints and polling with a single persistent WebSocket connection using the pages push protocol, delivering zero-latency events end-to-end.

**Architecture:** Single Quarkus WebSocket endpoint (`/api/ws`) using `quarkus-websockets-next`. A `WebSocketEventBus` CDI singleton routes events from three producer types (ChannelBackend, metadata push methods, file watcher) to the correct WebSocket connections. Panels are unchanged — they already subscribe to `pages-event` CustomEvents; only the transport layer beneath them changes.

**Tech Stack:** Quarkus 3.34.3, quarkus-websockets-next, casehub-pages (pages-data WebSocket source), TypeScript/esbuild

**Spec:** `docs/superpowers/specs/2026-07-03-websocket-push-design.md`

## Global Constraints

- Java 21, Quarkus 3.34.3
- `quarkus-websockets-next` for server-side WebSocket (same extension Claudony uses)
- Pages wire format for all server→client messages: `{ "op": "event", "topic": "...", "payload": ... }`
- Pages `subscribe`/`unsubscribe` for client→server signaling (pending pages#98)
- `CopyOnWriteArraySet` for connection sets in `WebSocketEventBus` (low-write, high-read)
- `connection.sendText(json).subscribe().with(v -> {}, err -> unregister(connection))` for self-healing dead connection cleanup
- `@RunOnVirtualThread` on WebSocket message handlers (file watch setup is blocking I/O)
- Build: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime`
- Single test class: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=<ClassName>`

---

### Task 1: WebSocketEventBus — event routing hub

The foundation. All other tasks depend on this. Pure unit-testable CDI bean with no WebSocket endpoint dependency.

**Files:**
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/WebSocketEventBus.java`
- Create: `server/runtime/src/test/java/io/casehub/drafthouse/WebSocketEventBusTest.java`

**Interfaces:**
- Consumes: `io.quarkus.websockets.next.WebSocketConnection` (from quarkus-websockets-next — add dependency first)
- Produces: `WebSocketEventBus` — injected by `DebateWebSocket` (Task 2), `DebateChannelBackend` (Task 3), `DebateEventResource` (Task 3), `DebateMcpTools` (Task 3)

- [ ] **Step 1: Add quarkus-websockets-next dependency**

Add to `server/runtime/pom.xml` inside `<dependencies>`:
```xml
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-websockets-next</artifactId>
</dependency>
```

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests`
Expected: BUILD SUCCESS

- [ ] **Step 2: Write WebSocketEventBus**

Create `server/runtime/src/main/java/io/casehub/drafthouse/WebSocketEventBus.java`:

```java
package io.casehub.drafthouse;

import java.util.Set;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArraySet;
import java.util.logging.Logger;

import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.drafthouse.debate.DebateStreamEntry;
import io.quarkus.websockets.next.WebSocketConnection;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class WebSocketEventBus {

    private static final Logger LOG = Logger.getLogger(WebSocketEventBus.class.getName());

    private final Set<WebSocketConnection> allConnections = new CopyOnWriteArraySet<>();
    private final ConcurrentHashMap<UUID, CopyOnWriteArraySet<WebSocketConnection>> sessionWatchers = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, CopyOnWriteArraySet<WebSocketConnection>> fileWatchers = new ConcurrentHashMap<>();

    @Inject ObjectMapper mapper;

    public void register(WebSocketConnection conn) {
        allConnections.add(conn);
    }

    public void unregister(WebSocketConnection conn) {
        allConnections.remove(conn);
        sessionWatchers.values().forEach(set -> set.remove(conn));
        fileWatchers.values().forEach(set -> set.remove(conn));
    }

    public void watchSession(WebSocketConnection conn, UUID channelId) {
        sessionWatchers.computeIfAbsent(channelId, k -> new CopyOnWriteArraySet<>()).add(conn);
    }

    public void unwatchSession(WebSocketConnection conn, UUID channelId) {
        CopyOnWriteArraySet<WebSocketConnection> watchers = sessionWatchers.get(channelId);
        if (watchers != null) {
            watchers.remove(conn);
            if (watchers.isEmpty()) sessionWatchers.remove(channelId);
        }
    }

    public void watchFile(WebSocketConnection conn, String path) {
        fileWatchers.computeIfAbsent(path, k -> new CopyOnWriteArraySet<>()).add(conn);
    }

    public void unwatchFile(WebSocketConnection conn, String path) {
        CopyOnWriteArraySet<WebSocketConnection> watchers = fileWatchers.get(path);
        if (watchers != null) {
            watchers.remove(conn);
            if (watchers.isEmpty()) fileWatchers.remove(path);
        }
    }

    public boolean hasFileWatchers(String path) {
        CopyOnWriteArraySet<WebSocketConnection> watchers = fileWatchers.get(path);
        return watchers != null && !watchers.isEmpty();
    }

    public void broadcast(String topic, Object payload) {
        String json = formatEvent(topic, payload);
        if (json == null) return;
        for (WebSocketConnection conn : allConnections) {
            sendSafe(conn, json);
        }
    }

    public void pushDebateEntries(UUID channelId, java.util.List<DebateStreamEntry> entries) {
        CopyOnWriteArraySet<WebSocketConnection> watchers = sessionWatchers.get(channelId);
        if (watchers == null || watchers.isEmpty()) return;
        String json = formatEvent("debate-entries", entries);
        if (json == null) return;
        for (WebSocketConnection conn : watchers) {
            sendSafe(conn, json);
        }
    }

    public void pushMetadata(UUID channelId, String topic, Object payload) {
        CopyOnWriteArraySet<WebSocketConnection> watchers = sessionWatchers.get(channelId);
        if (watchers == null || watchers.isEmpty()) return;
        String json = formatEvent(topic, payload);
        if (json == null) return;
        for (WebSocketConnection conn : watchers) {
            sendSafe(conn, json);
        }
    }

    public void pushFileChanged(String path) {
        CopyOnWriteArraySet<WebSocketConnection> watchers = fileWatchers.get(path);
        if (watchers == null || watchers.isEmpty()) return;
        String json = formatEvent("file-changed", java.util.Map.of("path", path));
        if (json == null) return;
        for (WebSocketConnection conn : watchers) {
            sendSafe(conn, json);
        }
    }

    private String formatEvent(String topic, Object payload) {
        try {
            String payloadJson = mapper.writeValueAsString(payload);
            return "{\"op\":\"event\",\"topic\":\"" + topic + "\",\"payload\":" + payloadJson + "}";
        } catch (Exception e) {
            LOG.warning("Failed to format event '" + topic + "': " + e.getMessage());
            return null;
        }
    }

    private void sendSafe(WebSocketConnection conn, String json) {
        conn.sendText(json).subscribe().with(v -> {}, err -> {
            LOG.fine("sendText failed — removing dead connection: " + err.getMessage());
            unregister(conn);
        });
    }

    // Test visibility
    int sessionWatcherCount(UUID channelId) {
        CopyOnWriteArraySet<WebSocketConnection> w = sessionWatchers.get(channelId);
        return w == null ? 0 : w.size();
    }

    int fileWatcherCount(String path) {
        CopyOnWriteArraySet<WebSocketConnection> w = fileWatchers.get(path);
        return w == null ? 0 : w.size();
    }

    int connectionCount() {
        return allConnections.size();
    }
}
```

- [ ] **Step 3: Write WebSocketEventBus tests**

Create `server/runtime/src/test/java/io/casehub/drafthouse/WebSocketEventBusTest.java`:

```java
package io.casehub.drafthouse;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.*;

import java.util.List;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.drafthouse.debate.AgentType;
import io.casehub.drafthouse.debate.DebateStreamEntry;
import io.casehub.drafthouse.debate.EntryType;
import io.quarkus.websockets.next.WebSocketConnection;
import io.smallrye.mutiny.Uni;

class WebSocketEventBusTest {

    private WebSocketEventBus bus;
    private ObjectMapper mapper;

    @BeforeEach
    void setUp() {
        bus = new WebSocketEventBus();
        mapper = new ObjectMapper();
        mapper.findAndRegisterModules();
        // Inject mapper via reflection (package-private field)
        try {
            var f = WebSocketEventBus.class.getDeclaredField("mapper");
            f.setAccessible(true);
            f.set(bus, mapper);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    @Test
    void register_and_unregister_tracks_connections() {
        WebSocketConnection conn = mockConnection();
        bus.register(conn);
        assertThat(bus.connectionCount()).isEqualTo(1);
        bus.unregister(conn);
        assertThat(bus.connectionCount()).isEqualTo(0);
    }

    @Test
    void watchSession_and_unwatchSession() {
        WebSocketConnection conn = mockConnection();
        UUID channelId = UUID.randomUUID();
        bus.register(conn);
        bus.watchSession(conn, channelId);
        assertThat(bus.sessionWatcherCount(channelId)).isEqualTo(1);
        bus.unwatchSession(conn, channelId);
        assertThat(bus.sessionWatcherCount(channelId)).isEqualTo(0);
    }

    @Test
    void watchFile_and_unwatchFile_with_reference_counting() {
        WebSocketConnection conn1 = mockConnection();
        WebSocketConnection conn2 = mockConnection();
        String path = "/tmp/test.md";
        bus.register(conn1);
        bus.register(conn2);
        bus.watchFile(conn1, path);
        bus.watchFile(conn2, path);
        assertThat(bus.hasFileWatchers(path)).isTrue();
        assertThat(bus.fileWatcherCount(path)).isEqualTo(2);
        bus.unwatchFile(conn1, path);
        assertThat(bus.hasFileWatchers(path)).isTrue();
        bus.unwatchFile(conn2, path);
        assertThat(bus.hasFileWatchers(path)).isFalse();
    }

    @Test
    void unregister_removes_from_all_watcher_sets() {
        WebSocketConnection conn = mockConnection();
        UUID channelId = UUID.randomUUID();
        String filePath = "/tmp/test.md";
        bus.register(conn);
        bus.watchSession(conn, channelId);
        bus.watchFile(conn, filePath);
        bus.unregister(conn);
        assertThat(bus.connectionCount()).isEqualTo(0);
        assertThat(bus.sessionWatcherCount(channelId)).isEqualTo(0);
        assertThat(bus.fileWatcherCount(filePath)).isEqualTo(0);
    }

    @Test
    void broadcast_sends_to_all_connections() {
        WebSocketConnection conn1 = mockConnection();
        WebSocketConnection conn2 = mockConnection();
        bus.register(conn1);
        bus.register(conn2);
        bus.broadcast("sessions", List.of());
        verify(conn1).sendText(anyString());
        verify(conn2).sendText(anyString());
    }

    @Test
    void pushDebateEntries_sends_only_to_session_watchers() {
        WebSocketConnection watcher = mockConnection();
        WebSocketConnection other = mockConnection();
        UUID channelId = UUID.randomUUID();
        bus.register(watcher);
        bus.register(other);
        bus.watchSession(watcher, channelId);

        DebateStreamEntry entry = new DebateStreamEntry(
                EntryType.POINT, AgentType.REV, 1, "test content",
                "p1", null, null, null, null, "rev-agent",
                java.time.Instant.now());
        bus.pushDebateEntries(channelId, List.of(entry));
        verify(watcher).sendText(anyString());
        verify(other, never()).sendText(anyString());
    }

    @Test
    void pushMetadata_sends_only_to_session_watchers() {
        WebSocketConnection watcher = mockConnection();
        WebSocketConnection other = mockConnection();
        UUID channelId = UUID.randomUUID();
        bus.register(watcher);
        bus.register(other);
        bus.watchSession(watcher, channelId);
        bus.pushMetadata(channelId, "context-usage", java.util.Map.of("effectivePercent", 42.0));
        verify(watcher).sendText(anyString());
        verify(other, never()).sendText(anyString());
    }

    @Test
    void pushFileChanged_sends_only_to_file_watchers() {
        WebSocketConnection watcher = mockConnection();
        WebSocketConnection other = mockConnection();
        bus.register(watcher);
        bus.register(other);
        bus.watchFile(watcher, "/tmp/test.md");
        bus.pushFileChanged("/tmp/test.md");
        verify(watcher).sendText(anyString());
        verify(other, never()).sendText(anyString());
    }

    @Test
    void sendText_failure_triggers_unregister() {
        WebSocketConnection deadConn = mock(WebSocketConnection.class);
        when(deadConn.sendText(anyString())).thenReturn(Uni.createFrom().failure(new RuntimeException("closed")));
        bus.register(deadConn);
        bus.broadcast("test", "payload");
        // Give the Uni subscriber time to fire
        assertThat(bus.connectionCount()).isEqualTo(0);
    }

    private WebSocketConnection mockConnection() {
        WebSocketConnection conn = mock(WebSocketConnection.class);
        when(conn.sendText(anyString())).thenReturn(Uni.createFrom().voidItem());
        return conn;
    }
}
```

- [ ] **Step 4: Run tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=WebSocketEventBusTest`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```
feat: add WebSocketEventBus — event routing hub for WebSocket push

Refs #87
```

---

### Task 2: DebateWebSocket endpoint + catch-up + integration tests

The WebSocket endpoint that connects browsers to the event bus. Handles connect, subscribe/unsubscribe parsing, catch-up delivery, and cleanup.

**Files:**
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/DebateWebSocket.java`
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/DraftHouseConfig.java` — add `debate().catchUpLimit()`
- Create: `server/runtime/src/test/java/io/casehub/drafthouse/DebateWebSocketTest.java`

**Interfaces:**
- Consumes: `WebSocketEventBus` (Task 1), `DebateSessionRegistry`, `MessageService`, `DraftHouseConfig`
- Produces: `DebateWebSocket` — the `/api/ws` WebSocket endpoint

- [ ] **Step 1: Add catch-up limit config**

Add to `DraftHouseConfig.java` after the `Context` interface:

```java
Debate debate();

interface Debate {
    @WithDefault("500")
    int catchUpLimit();
}
```

- [ ] **Step 2: Write DebateWebSocket**

Create `server/runtime/src/main/java/io/casehub/drafthouse/DebateWebSocket.java`:

```java
package io.casehub.drafthouse;

import java.io.IOException;
import java.nio.file.*;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;
import java.util.logging.Logger;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.blocks.channel.ContextSnapshot;
import io.casehub.drafthouse.debate.DebateStreamEntry;
import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.runtime.message.MessageService;
import io.quarkus.websockets.next.*;
import io.smallrye.common.annotation.RunOnVirtualThread;
import jakarta.inject.Inject;

@WebSocket(path = "/api/ws")
public class DebateWebSocket {

    private static final Logger LOG = Logger.getLogger(DebateWebSocket.class.getName());

    @Inject WebSocketEventBus eventBus;
    @Inject DebateSessionRegistry registry;
    @Inject MessageService messageService;
    @Inject DraftHouseConfig config;
    @Inject ObjectMapper mapper;

    private final ConcurrentHashMap<String, FileWatchHandle> activeFileWatches = new ConcurrentHashMap<>();

    record FileWatchHandle(WatchService watchService, Thread watchThread) {}

    @OnOpen
    void onOpen(WebSocketConnection connection) {
        eventBus.register(connection);
        sendEvent(connection, "reconnected", Map.of());
        Collection<DebateEventResource.SessionInfo> sessions = registry.activeSessions().stream()
                .map(s -> new DebateEventResource.SessionInfo(
                        s.debateSessionId(), s.channelName(), s.primaryPath(), s.agentId()))
                .toList();
        sendEvent(connection, "sessions", sessions);
    }

    @OnTextMessage
    @RunOnVirtualThread
    void onMessage(WebSocketConnection connection, String message) {
        JsonNode node;
        try {
            node = mapper.readTree(message);
        } catch (Exception e) {
            LOG.warning("Malformed JSON from WebSocket client: " + e.getMessage());
            return;
        }

        String op = node.has("op") ? node.get("op").asText() : null;
        if (op == null) {
            LOG.warning("WebSocket message missing 'op' field");
            return;
        }

        String dataset = node.has("dataset") ? node.get("dataset").asText() : null;

        switch (op) {
            case "subscribe" -> handleSubscribe(connection, dataset);
            case "unsubscribe" -> handleUnsubscribe(connection, dataset);
            default -> LOG.warning("Unknown WebSocket op: " + op);
        }
    }

    @OnClose
    void onClose(WebSocketConnection connection) {
        cleanup(connection);
    }

    @OnError
    void onError(WebSocketConnection connection, Throwable error) {
        LOG.fine("WebSocket error: " + error.getMessage());
        cleanup(connection);
    }

    private void handleSubscribe(WebSocketConnection connection, String dataset) {
        if (dataset == null) return;

        if (dataset.startsWith("debate:")) {
            String sessionIdStr = dataset.substring("debate:".length());
            UUID channelId;
            try {
                channelId = UUID.fromString(sessionIdStr);
            } catch (IllegalArgumentException e) {
                LOG.warning("Invalid debate session UUID: " + sessionIdStr);
                return;
            }
            DebateSession session = registry.find(channelId).orElse(null);
            if (session == null) return;

            eventBus.watchSession(connection, channelId);
            sendCatchUp(connection, session, channelId);

        } else if (dataset.startsWith("file:")) {
            String path = dataset.substring("file:".length());
            if (!isPathAllowed(connection, path)) {
                LOG.warning("File watch rejected — path not in any watched session's document set: " + path);
                return;
            }
            eventBus.watchFile(connection, path);
            startFileWatch(path);
        }
        // Unrecognized dataset patterns are silently ignored (e.g. "_events" dummy subscription)
    }

    private void handleUnsubscribe(WebSocketConnection connection, String dataset) {
        if (dataset == null) return;

        if (dataset.startsWith("debate:")) {
            String sessionIdStr = dataset.substring("debate:".length());
            try {
                UUID channelId = UUID.fromString(sessionIdStr);
                eventBus.unwatchSession(connection, channelId);
            } catch (IllegalArgumentException e) {
                // ignore
            }
        } else if (dataset.startsWith("file:")) {
            String path = dataset.substring("file:".length());
            eventBus.unwatchFile(connection, path);
            stopFileWatchIfUnused(path);
        }
    }

    private void sendCatchUp(WebSocketConnection connection, DebateSession session, UUID channelId) {
        List<Message> messages = messageService.pollAfter(channelId, 0L, config.debate().catchUpLimit());
        List<DebateStreamEntry> entries = messages.stream()
                .map(DebateStreamEntry::from)
                .filter(Objects::nonNull)
                .toList();
        if (!entries.isEmpty()) {
            sendEvent(connection, "debate-entries", entries);
        }

        ContextSnapshot ctxSnapshot = session.contextTracker().snapshot(
                config.context().windowSizeChars(), config.context().thresholdPercent());
        sendEvent(connection, "context-usage", Map.of(
                "serverContributionChars", ctxSnapshot.serverContributionChars(),
                "windowSizeChars", ctxSnapshot.windowSizeChars(),
                "agentReportedPercent", ctxSnapshot.agentReportedPercent() != null ? ctxSnapshot.agentReportedPercent() : "null",
                "effectivePercent", ctxSnapshot.effectivePercent(),
                "messageCount", ctxSnapshot.messageCount(),
                "thresholdExceeded", ctxSnapshot.thresholdExceeded()
        ));

        String docsJson = DocumentSetJson.documentsToJson(session.documents());
        sendEventRaw(connection, "documents-changed", "{\"documents\":" + docsJson + "}");

        ComparisonPair cp = session.currentComparison();
        if (cp != null) {
            sendEvent(connection, "comparison-changed", Map.of("pathA", cp.pathA(), "pathB", cp.pathB()));
        }
    }

    private boolean isPathAllowed(WebSocketConnection connection, String path) {
        // Allow if path is in any watched session's document set for this connection
        for (var entry : registry.activeSessions()) {
            if (entry.documents() != null) {
                for (DocumentEntry doc : entry.documents().entries()) {
                    if (path.equals(doc.path())) return true;
                }
            }
        }
        return false;
    }

    private void startFileWatch(String path) {
        if (activeFileWatches.containsKey(path)) return;
        try {
            Path target = Path.of(path);
            Path dir = target.getParent();
            String name = target.getFileName().toString();
            WatchService ws = FileSystems.getDefault().newWatchService();
            dir.register(ws, StandardWatchEventKinds.ENTRY_MODIFY);

            Thread watchThread = Thread.ofVirtual().start(() -> {
                try {
                    while (!Thread.currentThread().isInterrupted()) {
                        WatchKey key = ws.poll(200, TimeUnit.MILLISECONDS);
                        if (key == null) continue;
                        for (WatchEvent<?> event : key.pollEvents()) {
                            if (name.equals(event.context().toString())) {
                                eventBus.pushFileChanged(path);
                            }
                        }
                        if (!key.reset()) break;
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } catch (ClosedWatchServiceException ignored) {
                } finally {
                    try { ws.close(); } catch (IOException ignored) {}
                }
            });
            activeFileWatches.put(path, new FileWatchHandle(ws, watchThread));
        } catch (IOException e) {
            LOG.warning("Failed to start file watch for " + path + ": " + e.getMessage());
        }
    }

    private void stopFileWatchIfUnused(String path) {
        if (eventBus.hasFileWatchers(path)) return;
        FileWatchHandle handle = activeFileWatches.remove(path);
        if (handle != null) {
            handle.watchThread().interrupt();
            try { handle.watchService().close(); } catch (IOException ignored) {}
        }
    }

    private void cleanup(WebSocketConnection connection) {
        eventBus.unregister(connection);
        // Check all file watches for cleanup
        for (String path : new ArrayList<>(activeFileWatches.keySet())) {
            stopFileWatchIfUnused(path);
        }
    }

    private void sendEvent(WebSocketConnection connection, String topic, Object payload) {
        try {
            String payloadJson = mapper.writeValueAsString(payload);
            String json = "{\"op\":\"event\",\"topic\":\"" + topic + "\",\"payload\":" + payloadJson + "}";
            connection.sendText(json).subscribe().with(v -> {}, err -> {
                LOG.fine("sendEvent failed: " + err.getMessage());
                eventBus.unregister(connection);
            });
        } catch (Exception e) {
            LOG.warning("Failed to serialize event '" + topic + "': " + e.getMessage());
        }
    }

    private void sendEventRaw(WebSocketConnection connection, String topic, String payloadJson) {
        String json = "{\"op\":\"event\",\"topic\":\"" + topic + "\",\"payload\":" + payloadJson + "}";
        connection.sendText(json).subscribe().with(v -> {}, err -> {
            LOG.fine("sendEventRaw failed: " + err.getMessage());
            eventBus.unregister(connection);
        });
    }
}
```

- [ ] **Step 3: Write DebateWebSocket integration test**

Create `server/runtime/src/test/java/io/casehub/drafthouse/DebateWebSocketTest.java`:

```java
package io.casehub.drafthouse;

import static org.assertj.core.api.Assertions.assertThat;

import java.net.URI;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

import jakarta.inject.Inject;
import jakarta.websocket.*;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

import io.quarkus.test.common.http.TestHTTPResource;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class DebateWebSocketTest {

    private static final Pattern DEBATE_ID_PATTERN =
            Pattern.compile("\"debateSessionId\":\"([^\"]+)\"");

    @TestHTTPResource("/api/ws")
    URI wsUri;

    @Inject DebateMcpTools tools;
    @Inject ObjectMapper mapper;

    private String activeDebateSessionId;
    private Session wsSession;

    @BeforeEach
    void setUp() {
        activeDebateSessionId = null;
        wsSession = null;
    }

    @AfterEach
    void tearDown() {
        if (wsSession != null && wsSession.isOpen()) {
            try { wsSession.close(); } catch (Exception ignored) {}
        }
        if (activeDebateSessionId != null) {
            tools.endDebate(activeDebateSessionId, false);
        }
    }

    @Test
    void connect_receives_reconnected_and_sessions() throws Exception {
        TestClient client = new TestClient(2);
        wsSession = connectWebSocket(client);

        assertThat(client.awaitMessages(5)).isTrue();
        List<JsonNode> messages = client.parsedMessages(mapper);
        assertThat(messages).anyMatch(m -> "reconnected".equals(m.get("topic").asText()));
        assertThat(messages).anyMatch(m -> "sessions".equals(m.get("topic").asText()));
    }

    @Test
    void subscribe_to_debate_triggers_catch_up() throws Exception {
        String startResult = tools.startDebate("test-spec.md", null);
        activeDebateSessionId = extractDebateId(startResult);
        tools.raisePoint(activeDebateSessionId, "REV", 1,
                "Test point", "HIGH", null, null);

        TestClient client = new TestClient(5);
        wsSession = connectWebSocket(client);
        client.awaitMessages(5);
        client.resetLatch(4);

        wsSession.getBasicRemote().sendText(
                "{\"op\":\"subscribe\",\"dataset\":\"debate:" + activeDebateSessionId + "\"}");

        assertThat(client.awaitMessages(5)).isTrue();
        List<JsonNode> messages = client.parsedMessages(mapper);
        assertThat(messages).anyMatch(m -> "debate-entries".equals(m.get("topic").asText()));
        assertThat(messages).anyMatch(m -> "context-usage".equals(m.get("topic").asText()));
        assertThat(messages).anyMatch(m -> "documents-changed".equals(m.get("topic").asText()));
    }

    @Test
    void subscribe_to_nonexistent_session_silently_ignored() throws Exception {
        TestClient client = new TestClient(2);
        wsSession = connectWebSocket(client);
        client.awaitMessages(5);
        client.resetLatch(1);

        wsSession.getBasicRemote().sendText(
                "{\"op\":\"subscribe\",\"dataset\":\"debate:00000000-0000-0000-0000-000000000000\"}");

        // Should NOT receive any catch-up events — just silence
        assertThat(client.awaitMessages(2)).isFalse();
    }

    @Test
    void unrecognized_dataset_silently_ignored() throws Exception {
        TestClient client = new TestClient(2);
        wsSession = connectWebSocket(client);
        client.awaitMessages(5);
        client.resetLatch(1);

        wsSession.getBasicRemote().sendText(
                "{\"op\":\"subscribe\",\"dataset\":\"_events\"}");

        assertThat(client.awaitMessages(2)).isFalse();
    }

    @Test
    void malformed_json_does_not_crash() throws Exception {
        TestClient client = new TestClient(2);
        wsSession = connectWebSocket(client);
        client.awaitMessages(5);

        wsSession.getBasicRemote().sendText("not json at all");
        wsSession.getBasicRemote().sendText("{\"no_op_field\": true}");

        // Connection should still be alive
        assertThat(wsSession.isOpen()).isTrue();
    }

    private Session connectWebSocket(TestClient client) throws Exception {
        WebSocketContainer container = ContainerProvider.getWebSocketContainer();
        return container.connectToServer(client, wsUri);
    }

    private String extractDebateId(String result) {
        Matcher m = DEBATE_ID_PATTERN.matcher(result);
        assertThat(m.find()).isTrue();
        return m.group(1);
    }

    @ClientEndpoint
    public static class TestClient {
        final CopyOnWriteArrayList<String> received = new CopyOnWriteArrayList<>();
        volatile CountDownLatch latch;

        TestClient(int expectedMessages) {
            this.latch = new CountDownLatch(expectedMessages);
        }

        @OnMessage
        public void onMessage(String msg) {
            received.add(msg);
            latch.countDown();
        }

        boolean awaitMessages(int timeoutSeconds) throws InterruptedException {
            return latch.await(timeoutSeconds, TimeUnit.SECONDS);
        }

        void resetLatch(int expectedMessages) {
            received.clear();
            latch = new CountDownLatch(expectedMessages);
        }

        List<JsonNode> parsedMessages(ObjectMapper mapper) {
            return received.stream().map(s -> {
                try { return mapper.readTree(s); } catch (Exception e) { return null; }
            }).filter(n -> n != null).toList();
        }
    }
}
```

- [ ] **Step 4: Run tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DebateWebSocketTest`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```
feat: add DebateWebSocket endpoint — /api/ws with subscribe/catch-up lifecycle

Refs #87
```

---

### Task 3: Wire server-side producers

Connect `DebateChannelBackend`, `DebateEventResource`, and `DebateMcpTools` to `WebSocketEventBus`. After this task, events flow end-to-end from Qhorus to WebSocket clients.

**Files:**
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/DebateChannelBackend.java`
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/debate/DebateStreamEntry.java` — add `from(OutboundMessage)` factory
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/DebateEventResource.java` — metadata push uses WebSocketEventBus
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/DebateMcpTools.java` — session lifecycle broadcasts
- Modify: `server/runtime/src/test/java/io/casehub/drafthouse/DebateChannelBackendFactoryTest.java` — verify push

**Interfaces:**
- Consumes: `WebSocketEventBus` (Task 1), `OutboundMessage`, `DebateStreamEntry`
- Produces: Live event delivery path — ChannelBackend.post() → WebSocketEventBus → browser

- [ ] **Step 1: Add DebateStreamEntry.from(OutboundMessage)**

Add to `server/runtime/src/main/java/io/casehub/drafthouse/debate/DebateStreamEntry.java` after the existing `from(Message)` method:

```java
public static DebateStreamEntry from(io.casehub.qhorus.api.gateway.OutboundMessage msg) {
    if (msg.content() == null) return null;
    Map<String, String> meta = DebateProtocol.parseMeta(msg.content());
    String entryTypeStr = meta.get("entryType");
    if (entryTypeStr == null) return null;

    EntryType entryType;
    try {
        entryType = EntryType.valueOf(entryTypeStr);
    } catch (IllegalArgumentException e) {
        return null;
    }

    String agentStr = meta.get(ConversationProtocol.ROLE);
    if (agentStr == null) agentStr = meta.get("agent");
    AgentType agentRole = null;
    if (agentStr != null) {
        try {
            agentRole = AgentType.valueOf(agentStr);
        } catch (IllegalArgumentException e) {
            return null;
        }
    } else if (entryType != EntryType.RESTART_CONTEXT) {
        return null;
    }

    int round = DebateProtocol.parseRound(meta);
    String body = DebateProtocol.bodyContent(msg.content());

    boolean isSubTask = entryType == EntryType.SUB_TASK_REQUEST
            || entryType == EntryType.SUB_TASK_FINDING
            || entryType == EntryType.SUB_TASK_ERROR;

    String correlationId = msg.correlationId() != null ? msg.correlationId().toString() : null;
    String pointId = isSubTask ? meta.get("pointId") : correlationId;
    String subTaskId = isSubTask ? correlationId : null;

    Priority priority = parsePriority(meta.get("priority"));
    String scope = meta.get("scope");
    String location = meta.get("location");

    return new DebateStreamEntry(
            entryType, agentRole, round, body,
            pointId, subTaskId,
            priority, scope,
            location != null && !location.isBlank() ? location : null,
            msg.sender(),
            Instant.now());
}
```

- [ ] **Step 2: Modify DebateChannelBackend to push all messages**

In `DebateChannelBackend.java`, add `@Inject WebSocketEventBus eventBus;` field and update `post()`:

```java
@Inject WebSocketEventBus eventBus;

@Override
public void post(ChannelRef channel, OutboundMessage message) {
    // Push all message types to WebSocket watchers
    DebateStreamEntry entry = DebateStreamEntry.from(message);
    if (entry != null) {
        eventBus.pushDebateEntries(channel.id(), java.util.List.of(entry));
    }

    // Still fire CDI event for SUB_TASK_REQUEST (agent dispatch — orthogonal)
    Map<String, String> meta = DebateProtocol.parseMeta(message.content());
    if (!"SUB_TASK_REQUEST".equals(meta.get("entryType"))) return;

    DebateSession session = registry.find(channel.id()).orElse(null);
    if (session == null) {
        LOG.warning("DebateChannelBackend: SUB_TASK_REQUEST on " + channel.id()
                + " — no active session, dropped");
        return;
    }

    String correlationId = message.correlationId() != null
            ? message.correlationId().toString() : UUID.randomUUID().toString();
    channelAgentEvent.fireAsync(new ChannelAgentRequest(channel.id(), correlationId, message));
}
```

Update the test constructor to accept `WebSocketEventBus`:
```java
DebateChannelBackend(Event<ChannelAgentRequest> channelAgentEvent,
                     DebateSessionRegistry registry,
                     WebSocketEventBus eventBus) {
    this.channelAgentEvent = channelAgentEvent;
    this.registry = registry;
    this.eventBus = eventBus;
}
```

- [ ] **Step 3: Modify DebateEventResource metadata push methods**

In `DebateEventResource.java`, add `@Inject WebSocketEventBus eventBus;` and change the four push methods to call `eventBus` directly:

```java
@Inject WebSocketEventBus eventBus;

public void pushContextSnapshot(UUID channelId, ContextSnapshot snapshot) {
    try {
        eventBus.pushMetadata(channelId, "context-usage", java.util.Map.of(
                "serverContributionChars", snapshot.serverContributionChars(),
                "agentReportedPercent", snapshot.agentReportedPercent(),
                "effectivePercent", snapshot.effectivePercent(),
                "messageCount", snapshot.messageCount(),
                "thresholdExceeded", snapshot.thresholdExceeded()
        ));
    } catch (Exception e) {
        LOG.warning("Failed to push context snapshot: " + e.getMessage());
    }
}

public void pushDocumentsChanged(UUID channelId, DebateSession session) {
    try {
        String docsJson = DocumentSetJson.documentsToJson(session.documents());
        eventBus.pushMetadata(channelId, "documents-changed",
                mapper.readTree("{\"documents\":" + docsJson + "}"));
    } catch (Exception e) {
        LOG.warning("Failed to push documents-changed: " + e.getMessage());
    }
}

public void pushComparisonChanged(UUID channelId, ComparisonPair cp) {
    try {
        if (cp != null) {
            eventBus.pushMetadata(channelId, "comparison-changed",
                    java.util.Map.of("pathA", cp.pathA(), "pathB", cp.pathB()));
        } else {
            eventBus.pushMetadata(channelId, "comparison-changed",
                    java.util.Map.ofNullable("pathA", null));
        }
    } catch (Exception e) {
        LOG.warning("Failed to push comparison-changed: " + e.getMessage());
    }
}
```

And update the private `pushSelectionEvent` similarly:
```java
private void pushSelectionEvent(UUID channelId, SelectionScope scope) {
    try {
        eventBus.pushMetadata(channelId, "selection-scope", java.util.Map.of(
                "side", scope.side().name(),
                "startLine", scope.startLine(),
                "endLine", scope.endLine(),
                "selectedText", scope.selectedText()
        ));
    } catch (Exception e) {
        LOG.warning("Failed to push selection event: " + e.getMessage());
    }
}
```

Remove the four `ConcurrentHashMap` fields (`pendingContextSnapshots`, `pendingSelections`, `pendingDocuments`, `pendingComparisons`).

- [ ] **Step 4: Add session lifecycle broadcasts to DebateMcpTools**

In `DebateMcpTools.java`, add `@Inject WebSocketEventBus eventBus;` field.

In `startDebate()`, after `registry.put(session)` and before `channelGateway.initChannel(...)`, add:

```java
eventBus.broadcast("session-created", new DebateEventResource.SessionInfo(
        session.debateSessionId(), session.channelName(), specPath, session.agentId()));
```

In `endDebate()`, after `registry.remove(channelId)`, add:

```java
eventBus.broadcast("session-ended", java.util.Map.of("debateSessionId", debateSessionId));
```

- [ ] **Step 5: Update DebateChannelBackendFactoryTest**

Add a mock `WebSocketEventBus` to the test setup in `DebateChannelBackendFactoryTest.java`:

```java
@SuppressWarnings("unchecked")
private WebSocketEventBus eventBus = mock(WebSocketEventBus.class);
```

Update the `setUp()` method to pass `eventBus` to the `DebateChannelBackend` constructor:

```java
debateBackend = new DebateChannelBackend(channelAgentEvent, debateRegistry, eventBus);
```

Add a test verifying push:

```java
@Test
void post_pushesAllMessageTypesToEventBus() {
    UUID channelId = UUID.randomUUID();
    ChannelRef ref = new ChannelRef(channelId, "debate-channel");
    OutboundMessage msg = new OutboundMessage(
            UUID.randomUUID(), "rev-agent", MessageType.COMMAND,
            "---\nentryType: POINT\nrole: REV\nround: 1\npriority: HIGH\n---\nTest content",
            null, null, ActorType.AGENT);
    debateBackend.post(ref, msg);
    verify(eventBus).pushDebateEntries(eq(channelId), anyList());
}
```

- [ ] **Step 6: Run all tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime`
Expected: All existing tests still pass, new test passes

- [ ] **Step 7: Commit**

```
feat: wire ChannelBackend, metadata, and lifecycle to WebSocketEventBus

DebateChannelBackend.post() pushes all message types to WebSocket watchers.
DebateEventResource metadata push methods call WebSocketEventBus directly.
DebateMcpTools broadcasts session-created/session-ended on lifecycle events.

Refs #87
```

---

### Task 4: Client-side WebSocket migration

Replace all SSE/polling with a single WebSocket connection. Delete `sse-bridge.ts`, rewrite session discovery in `index.ts`, update panels.

**Files:**
- Delete: `server/runtime/src/main/webui/src/sse-bridge.ts`
- Modify: `server/runtime/src/main/webui/src/index.ts`
- Modify: `server/runtime/src/main/webui/src/panels/drafthouse-debate.js`
- Modify: `server/runtime/src/main/webui/src/panels/drafthouse-review-tracker.js`
- Modify: `server/runtime/src/main/webui/src/panels/drafthouse-context-gauge.js`
- Modify: `server/runtime/src/main/webui/src/panels/drafthouse-diff.js`

**Interfaces:**
- Consumes: `@casehubio/pages-data` `createWebSocketSource` (via `@casehubio/pages-runtime` transitive dep)
- Produces: WebSocket connection that dispatches `pages-event` CustomEvents to panels

- [ ] **Step 1: Update panels — reconnection signal and context-gauge type guard**

In `drafthouse-debate.js`: replace `topic === 'sse-reconnect'` with `topic === 'reconnected'`.

In `drafthouse-review-tracker.js`: replace `topic === 'sse-reconnect'` with `topic === 'reconnected'`.

In `drafthouse-context-gauge.js`: replace `topic === 'sse-reconnect'` with `topic === 'reconnected'`, and remove the `if (data.type !== 'context-usage') return` guard in `#handleMeta()` (under WebSocket, payload does not include a `type` field — the topic-based filter in the listener already provides discrimination).

- [ ] **Step 2: Remove EventSource from drafthouse-diff.js**

Remove all `EventSource` creation code and `onmessage` handlers. Add a `pages-event` listener for topic `file-changed`:

```javascript
connectedCallback() {
    // ... existing code ...
    document.addEventListener('pages-event', this.#fileChangeHandler);
}

disconnectedCallback() {
    // ... existing code ...
    document.removeEventListener('pages-event', this.#fileChangeHandler);
}

#fileChangeHandler = (e) => {
    const { topic, payload } = e.detail;
    if (topic !== 'file-changed') return;
    const path = payload.path;
    if (this.#pathA && path === this.#pathA) this.loadFile('a', path);
    if (this.#pathB && path === this.#pathB) this.loadFile('b', path);
};
```

Remove any `EventSource` close/cleanup in `disconnectedCallback`.

- [ ] **Step 3: Delete sse-bridge.ts**

Delete `server/runtime/src/main/webui/src/sse-bridge.ts`.

- [ ] **Step 4: Rewrite index.ts — WebSocket connection and session discovery**

Replace the session discovery and SSE bridge sections in `index.ts`:

```typescript
import { createWebSocketSource } from "@casehubio/pages-data/dist/dataset/external/sources/websocket-source.js";

// ── WebSocket connection ─────────────────────────────────────────────
const wsProto = location.protocol === "https:" ? "wss:" : "ws:";
const wsSource = createWebSocketSource(`${wsProto}//${location.host}/api/ws`, {
  eventTarget: document.documentElement as HTMLElement,
});

// Dummy subscription to establish the connection — listener is a no-op.
// Server silently ignores unrecognized dataset patterns.
const noOp = () => {};
wsSource.subscribe("_events" as any, { uuid: "_events" as any }, { snapshot: noOp, append: noOp, replace: noOp, remove: noOp }, noOp);

let currentSessionId: string | null = null;
let watchedFiles: string[] = [];

function connectDebateSession(sessionId: string): void {
    if (currentSessionId) {
        wsSource.unsubscribe(("debate:" + currentSessionId) as any);
        watchedFiles.forEach(f => wsSource.unsubscribe(("file:" + f) as any));
        watchedFiles = [];
    }
    currentSessionId = sessionId;
    wsSource.subscribe(("debate:" + sessionId) as any,
        { uuid: ("debate:" + sessionId) as any }, { snapshot: noOp, append: noOp, replace: noOp, remove: noOp }, noOp);

    const debateEl = document.querySelector("drafthouse-debate") as any;
    const reviewEl = document.querySelector("drafthouse-review-tracker") as any;
    const diffEl = document.querySelector("drafthouse-diff") as any;

    if (debateEl) debateEl.configure({ debateSessionId: sessionId });
    if (reviewEl) reviewEl.configure({ debateSessionId: sessionId });

    // Fetch initial documents — comparison comes via catch-up events
    fetch(`/api/debate/${sessionId}/documents`)
        .then(r => r.json())
        .then((data: any) => {
            if (data.currentComparison) {
                if (data.currentComparison.pathA && diffEl) diffEl.loadFile("a", data.currentComparison.pathA);
                if (data.currentComparison.pathB && diffEl) diffEl.loadFile("b", data.currentComparison.pathB);
                watchFiles(data.currentComparison.pathA, data.currentComparison.pathB);
            }
        })
        .catch(() => {});
}

function watchFiles(...paths: (string | null)[]): void {
    paths.filter(Boolean).forEach(p => {
        if (!watchedFiles.includes(p!)) {
            watchedFiles.push(p!);
            wsSource.subscribe(("file:" + p) as any,
                { uuid: ("file:" + p) as any }, { snapshot: noOp, append: noOp, replace: noOp, remove: noOp }, noOp);
        }
    });
}

export function getSessionId(): string | null {
    return currentSessionId;
}

// Session lifecycle — auto-connect or show picker
document.addEventListener("pages-event", ((e: CustomEvent) => {
    const { topic, payload } = e.detail;
    if (topic === "sessions" && !currentSessionId) {
        if (payload.length === 1) {
            connectDebateSession(payload[0].debateSessionId);
        } else if (payload.length > 1) {
            showSessionPicker(payload);
        }
    }
    if (topic === "session-created" && !currentSessionId) {
        connectDebateSession(payload.debateSessionId);
    }
}) as EventListener);
```

Remove the `import { connectSSE, getSessionId } from "./sse-bridge.js"` line and the `startSessionDiscovery()` function and its call.

Update the `comparison-changed` handler to also manage file watches:

```typescript
if (topic === "comparison-changed") {
    const diff = document.querySelector("drafthouse-diff") as any;
    if (diff) {
        // Unwatch old files
        watchedFiles.forEach(f => wsSource.unsubscribe(("file:" + f) as any));
        watchedFiles = [];
        if (payload.pathA) { diff.loadFile("a", payload.pathA); }
        if (payload.pathB) { diff.loadFile("b", payload.pathB); }
        watchFiles(payload.pathA, payload.pathB);
    }
}
```

Update the `selection-changed` handler to use the module-level `currentSessionId` instead of `getSessionId()`:

```typescript
const sessionId = currentSessionId;
```

Handle the `debateParam` URL parameter by subscribing instead of calling `connectSSE`:

```typescript
if (debateParam) {
    connectDebateSession(debateParam);
}
// No else — sessions event from WebSocket handles auto-discovery
```

- [ ] **Step 5: Build webui**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml package -DskipTests -pl runtime`
Expected: BUILD SUCCESS (Quinoa rebuilds the webui)

- [ ] **Step 6: Commit**

```
feat: client-side WebSocket migration — replace SSE bridge and session polling

Delete sse-bridge.ts. Rewrite index.ts to use pages WebSocket source.
Update panels: sse-reconnect → reconnected, remove context-gauge type guard.
Remove per-file EventSource from drafthouse-diff.js.

Refs #87
```

---

### Task 5: Remove old SSE infrastructure + E2E test migration

Clean up the old SSE/polling code and migrate existing tests.

**Files:**
- Delete: `server/runtime/src/main/java/io/casehub/drafthouse/WatchResource.java`
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/DebateEventResource.java` — remove SSE endpoint + helpers
- Modify: `server/runtime/src/test/java/io/casehub/drafthouse/DebateEventResourceTest.java` — remove SSE tests, keep REST tests
- Modify: `server/runtime/src/test/java/io/casehub/drafthouse/e2e/DebatePanelE2ETest.java` — migrate SSE → WebSocket assertions (if SSE-specific assertions exist)

**Interfaces:**
- Consumes: none (removal task)
- Produces: clean codebase with no SSE infrastructure

- [ ] **Step 1: Delete WatchResource.java**

Delete `server/runtime/src/main/java/io/casehub/drafthouse/WatchResource.java`.

- [ ] **Step 2: Remove SSE endpoint from DebateEventResource**

Remove from `DebateEventResource.java`:
- The `events()` method (lines 98-155)
- The `serializeMessages()` method
- The `serializeContextSnapshot()` method
- The four `ConcurrentHashMap` field declarations (if not already removed in Task 3)
- Imports for `Multi`, `Uni`, `Duration`, `AtomicLong`, `io.smallrye.common.annotation.Blocking`

Keep:
- `activeSessions()` REST endpoint
- `postSelection()`, `deleteSelection()`, `getDocuments()`, `postComparison()` REST endpoints
- The metadata push methods (now calling WebSocketEventBus)
- `SessionInfo`, `SelectionRequest`, `ComparisonRequest` records

- [ ] **Step 3: Remove SSE-specific tests from DebateEventResourceTest**

Remove tests that connect to the SSE endpoint:
- `catchUp_deliversHistoricalEvents` (catch-up now tested via WebSocket in `DebateWebSocketTest`)
- `initialContextSnapshot_emittedOnConnect` (now tested via WebSocket)
- `pushedContextSnapshot_deliveredViaSse` (now tested via WebSocket)
- `selectionScope_deliveredViaSse` (now tested via WebSocket)
- `invalidSessionId_returns404` (SSE endpoint no longer exists)

Keep:
- `activeSessions_returnsCurrentDebates`
- `activeSessions_emptyWhenNoDebates`
- `selectionPost_storesSelection`
- `selectionDelete_clearsSelection`
- `selectionPost_invalidSession_returns404`

- [ ] **Step 4: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```
refactor: remove SSE infrastructure — WatchResource, SSE endpoint, polling loop

Delete WatchResource.java (file watching now via WebSocket).
Remove DebateEventResource.events() SSE endpoint and 500ms polling loop.
Remove SSE-specific tests (coverage moved to DebateWebSocketTest).

Closes #87
```

---

## File Change Summary

| Action | File |
|--------|------|
| Create | `server/runtime/src/main/java/io/casehub/drafthouse/WebSocketEventBus.java` |
| Create | `server/runtime/src/main/java/io/casehub/drafthouse/DebateWebSocket.java` |
| Create | `server/runtime/src/test/java/io/casehub/drafthouse/WebSocketEventBusTest.java` |
| Create | `server/runtime/src/test/java/io/casehub/drafthouse/DebateWebSocketTest.java` |
| Modify | `server/runtime/pom.xml` — quarkus-websockets-next dependency |
| Modify | `server/runtime/src/main/java/io/casehub/drafthouse/DraftHouseConfig.java` — debate().catchUpLimit() |
| Modify | `server/runtime/src/main/java/io/casehub/drafthouse/debate/DebateStreamEntry.java` — from(OutboundMessage) |
| Modify | `server/runtime/src/main/java/io/casehub/drafthouse/DebateChannelBackend.java` — push all messages |
| Modify | `server/runtime/src/main/java/io/casehub/drafthouse/DebateEventResource.java` — WebSocketEventBus + remove SSE |
| Modify | `server/runtime/src/main/java/io/casehub/drafthouse/DebateMcpTools.java` — lifecycle broadcasts |
| Modify | `server/runtime/src/test/java/io/casehub/drafthouse/DebateChannelBackendFactoryTest.java` — verify push |
| Modify | `server/runtime/src/test/java/io/casehub/drafthouse/DebateEventResourceTest.java` — remove SSE tests |
| Modify | `server/runtime/src/main/webui/src/index.ts` — WebSocket connection |
| Modify | `server/runtime/src/main/webui/src/panels/drafthouse-debate.js` — reconnected signal |
| Modify | `server/runtime/src/main/webui/src/panels/drafthouse-review-tracker.js` — reconnected signal |
| Modify | `server/runtime/src/main/webui/src/panels/drafthouse-context-gauge.js` — reconnected + type guard |
| Modify | `server/runtime/src/main/webui/src/panels/drafthouse-diff.js` — remove EventSource |
| Delete | `server/runtime/src/main/webui/src/sse-bridge.ts` |
| Delete | `server/runtime/src/main/java/io/casehub/drafthouse/WatchResource.java` |
