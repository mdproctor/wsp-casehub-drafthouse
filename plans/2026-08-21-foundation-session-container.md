# Foundation: Session Container & Facet Interface — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #117 — Composable capability architecture with voice-first drafting mode
**Issue group:** #117

**Goal:** Create the DraftHouseSession container, Facet interface, ArtifactSpec record, DraftHouseSessionStore SPI, session-level MCP tools, and REST endpoints — the foundation that all subsequent facet plans build on.

**Architecture:** A thin DraftHouseSession in the api module holds shared state (DocumentSet, workingDirectory, metadata) and manages a map of active Facet instances. Facet implementations are CDI beans in the runtime module that receive ToolManager and WebSocketEventBus via @Inject. Session-level MCP tools (create_session, activate_facet, etc.) are registered statically via @Tool and manage the session lifecycle. Document management tools (add_document, remove_document, etc.) are lifted from DebateMcpTools to session-level.

**Tech Stack:** Java 21, Quarkus 3.34.3, Quarkus MCP Server (ToolManager), casehub-pages-push (WebSocket)

## Global Constraints

- Java package: `io.casehub.drafthouse`
- api module: pure Java — no Quarkus, no JPA, no CDI annotations
- runtime module: Quarkus CDI beans, JPA, MCP tools
- All new public types need Javadoc on the class/interface (one line max)
- Test package mirrors source package
- Build: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests` then `/opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime`
- Existing E2E tests must continue to pass (379 tests in runtime)

---

## Batch 1: Domain Model (api module)

### Task 1: Facet interface and ArtifactSpec record

**Files:**
- Create: `server/api/src/main/java/io/casehub/drafthouse/Facet.java`
- Create: `server/api/src/main/java/io/casehub/drafthouse/ArtifactSpec.java`
- Test: `server/api/src/test/java/io/casehub/drafthouse/ArtifactSpecTest.java`

**Interfaces:**
- Produces: `Facet` interface with `name()`, `activate(DraftHouseSession)`, `deactivate(DraftHouseSession)`, `inputs()`, `outputs()`
- Produces: `ArtifactSpec` record with `pathPattern`, `description`, `required`

- [ ] **Step 1: Write tests for ArtifactSpec**

```java
package io.casehub.drafthouse;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ArtifactSpecTest {

    @Test
    void twoArgConstructorDefaultsRequiredToFalse() {
        var spec = new ArtifactSpec("notes/accumulated.md", "Accumulated voice notes");
        assertEquals("notes/accumulated.md", spec.pathPattern());
        assertEquals("Accumulated voice notes", spec.description());
        assertFalse(spec.required());
    }

    @Test
    void threeArgConstructorSetsRequired() {
        var spec = new ArtifactSpec("stages/*.md", "Draft stages", true);
        assertTrue(spec.required());
    }

    @Test
    void pathPatternIsRequired() {
        assertThrows(NullPointerException.class,
            () -> new ArtifactSpec(null, "desc"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl api -Dtest=ArtifactSpecTest`
Expected: Compilation failure — ArtifactSpec does not exist

- [ ] **Step 3: Create ArtifactSpec record**

```java
package io.casehub.drafthouse;

import java.util.Objects;

/** Declares an artifact a facet reads or writes, relative to the session working directory. */
public record ArtifactSpec(String pathPattern, String description, boolean required) {
    public ArtifactSpec {
        Objects.requireNonNull(pathPattern);
    }

    public ArtifactSpec(String pathPattern, String description) {
        this(pathPattern, description, false);
    }
}
```

- [ ] **Step 4: Create Facet interface**

```java
package io.casehub.drafthouse;

import java.util.List;

/** A composable session facet — independently activatable with its own tools, state, and artifacts. */
public interface Facet {
    String name();
    void activate(DraftHouseSession session);
    void deactivate(DraftHouseSession session);
    List<ArtifactSpec> inputs();
    List<ArtifactSpec> outputs();
}
```

- [ ] **Step 5: Run tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl api -Dtest=ArtifactSpecTest`
Expected: PASS (3 tests)

- [ ] **Step 6: Commit**

```bash
git add server/api/src/main/java/io/casehub/drafthouse/Facet.java server/api/src/main/java/io/casehub/drafthouse/ArtifactSpec.java server/api/src/test/java/io/casehub/drafthouse/ArtifactSpecTest.java
git commit -m "feat(#117): add Facet interface and ArtifactSpec record

Refs #117"
```

### Task 2: DraftHouseSession container

**Files:**
- Create: `server/api/src/main/java/io/casehub/drafthouse/DraftHouseSession.java`
- Test: `server/api/src/test/java/io/casehub/drafthouse/DraftHouseSessionTest.java`

**Interfaces:**
- Consumes: `Facet` interface (from Task 1), `DocumentSet` (existing)
- Produces: `DraftHouseSession` — `id()`, `created()`, `documentSet()`, `workingDirectory()`, `setWorkingDirectory(Path)`, `activeFacets()`, `activateFacet(Facet)`, `deactivateFacet(String)`, `findFacet(String)`, `metadata()`

- [ ] **Step 1: Write tests for DraftHouseSession**

```java
package io.casehub.drafthouse;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;
import java.nio.file.Path;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class DraftHouseSessionTest {

    @TempDir Path workDir;

    private final Facet stubFacet = new Facet() {
        boolean activated = false;
        @Override public String name() { return "test"; }
        @Override public void activate(DraftHouseSession s) { activated = true; }
        @Override public void deactivate(DraftHouseSession s) { activated = false; }
        @Override public List<ArtifactSpec> inputs() { return List.of(); }
        @Override public List<ArtifactSpec> outputs() { return List.of(); }
    };

    @Test
    void newSessionHasIdAndTimestamp() {
        var session = new DraftHouseSession("s1");
        assertEquals("s1", session.id());
        assertNotNull(session.created());
        assertTrue(session.activeFacets().isEmpty());
    }

    @Test
    void activateFacetCallsActivateAndRegisters() {
        var session = new DraftHouseSession("s1");
        session.activateFacet(stubFacet);
        assertTrue(session.findFacet("test").isPresent());
        assertEquals(1, session.activeFacets().size());
    }

    @Test
    void deactivateFacetCallsDeactivateAndRemoves() {
        var session = new DraftHouseSession("s1");
        session.activateFacet(stubFacet);
        session.deactivateFacet("test");
        assertTrue(session.findFacet("test").isEmpty());
    }

    @Test
    void duplicateActivationThrows() {
        var session = new DraftHouseSession("s1");
        session.activateFacet(stubFacet);
        assertThrows(IllegalStateException.class,
            () -> session.activateFacet(stubFacet));
    }

    @Test
    void deactivateUnknownFacetThrows() {
        var session = new DraftHouseSession("s1");
        assertThrows(IllegalArgumentException.class,
            () -> session.deactivateFacet("unknown"));
    }

    @Test
    void workingDirectoryIsSettable() {
        var session = new DraftHouseSession("s1");
        assertNull(session.workingDirectory());
        session.setWorkingDirectory(workDir);
        assertEquals(workDir, session.workingDirectory());
    }

    @Test
    void documentSetIsShared() {
        var session = new DraftHouseSession("s1");
        session.documentSet().add("/doc.md", "Doc");
        assertEquals(1, session.documentSet().documents().size());
    }

    @Test
    void activeFacetsReturnsUnmodifiableView() {
        var session = new DraftHouseSession("s1");
        session.activateFacet(stubFacet);
        assertThrows(UnsupportedOperationException.class,
            () -> session.activeFacets().put("x", stubFacet));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl api -Dtest=DraftHouseSessionTest`
Expected: Compilation failure — DraftHouseSession does not exist

- [ ] **Step 3: Create DraftHouseSession**

```java
package io.casehub.drafthouse;

import java.nio.file.Path;
import java.time.Instant;
import java.util.Collections;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

/** Unified session container — owns shared state and manages independently activatable facets. */
public class DraftHouseSession {

    private final String id;
    private final Instant created;
    private final DocumentSet documentSet;
    private final Map<String, Facet> activeFacets = new java.util.concurrent.ConcurrentHashMap<>();
    private final Map<String, Object> metadata = new java.util.concurrent.ConcurrentHashMap<>();
    private volatile Path workingDirectory;

    public DraftHouseSession(String id) {
        this.id = id;
        this.created = Instant.now();
        this.documentSet = new DocumentSet();
    }

    public String id() { return id; }
    public Instant created() { return created; }
    public DocumentSet documentSet() { return documentSet; }
    public Map<String, Object> metadata() { return metadata; }

    public Path workingDirectory() { return workingDirectory; }
    public void setWorkingDirectory(Path dir) { this.workingDirectory = dir; }

    public void activateFacet(Facet facet) {
        if (activeFacets.containsKey(facet.name())) {
            throw new IllegalStateException("Facet already active: " + facet.name());
        }
        activeFacets.put(facet.name(), facet);
        facet.activate(this);
    }

    public void deactivateFacet(String name) {
        Facet facet = activeFacets.remove(name);
        if (facet == null) {
            throw new IllegalArgumentException("Facet not active: " + name);
        }
        facet.deactivate(this);
    }

    public Optional<Facet> findFacet(String name) {
        return Optional.ofNullable(activeFacets.get(name));
    }

    public Map<String, Facet> activeFacets() {
        return Collections.unmodifiableMap(activeFacets);
    }
}
```

- [ ] **Step 4: Run tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml test -pl api -Dtest=DraftHouseSessionTest`
Expected: PASS (8 tests)

- [ ] **Step 5: Commit**

```bash
git add server/api/src/main/java/io/casehub/drafthouse/DraftHouseSession.java server/api/src/test/java/io/casehub/drafthouse/DraftHouseSessionTest.java
git commit -m "feat(#117): add DraftHouseSession container with facet lifecycle

Refs #117"
```

### Task 3: DraftHouseSessionStore SPI

**Files:**
- Create: `server/api/src/main/java/io/casehub/drafthouse/DraftHouseSessionStore.java` (includes nested `SessionSnapshot` record)
- Test: none needed (SPI interface only — tested via implementations)

**Interfaces:**
- Consumes: `DraftHouseSession` (from Task 2)
- Produces: `DraftHouseSessionStore` SPI — `save(DraftHouseSessionSnapshot)`, `load(String)`, `remove(String)`, `loadAll()`

- [ ] **Step 1: Create DraftHouseSessionStore SPI**

```java
package io.casehub.drafthouse;

import java.nio.file.Path;
import java.time.Instant;
import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.Optional;

/** Persists session metadata, active facet set, and working directory path. */
public interface DraftHouseSessionStore {
    void save(SessionSnapshot snapshot);
    Optional<SessionSnapshot> load(String sessionId);
    void remove(String sessionId);
    Collection<SessionSnapshot> loadAll();

    record SessionSnapshot(
        String id,
        Instant created,
        Path workingDirectory,
        List<String> activeFacetNames,
        Map<String, Object> metadata
    ) {}
}
```

- [ ] **Step 2: Commit**

```bash
git add server/api/src/main/java/io/casehub/drafthouse/DraftHouseSessionStore.java
git commit -m "feat(#117): add DraftHouseSessionStore SPI

Refs #117"
```

---

## Batch 2: Runtime Infrastructure

### Task 4: Session registry and NoOp store

**Files:**
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/DraftHouseSessionRegistry.java`
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/NoOpDraftHouseSessionStore.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/DraftHouseSessionRegistryTest.java`

**Interfaces:**
- Consumes: `DraftHouseSession` (Task 2), `DraftHouseSessionStore` (Task 3)
- Produces: `DraftHouseSessionRegistry` — `create(String)`, `find(String)`, `remove(String)`, `activeSessions()`

- [ ] **Step 1: Write tests for DraftHouseSessionRegistry**

```java
package io.casehub.drafthouse;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class DraftHouseSessionRegistryTest {

    DraftHouseSessionRegistry registry;

    @BeforeEach
    void setup() {
        registry = new DraftHouseSessionRegistry(new NoOpDraftHouseSessionStore());
    }

    @Test
    void createAndFind() {
        DraftHouseSession session = registry.create("s1");
        assertEquals("s1", session.id());
        assertTrue(registry.find("s1").isPresent());
    }

    @Test
    void duplicateCreateThrows() {
        registry.create("s1");
        assertThrows(IllegalStateException.class, () -> registry.create("s1"));
    }

    @Test
    void removeSession() {
        registry.create("s1");
        registry.remove("s1");
        assertTrue(registry.find("s1").isEmpty());
    }

    @Test
    void activeSessionsReturnsAll() {
        registry.create("s1");
        registry.create("s2");
        assertEquals(2, registry.activeSessions().size());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DraftHouseSessionRegistryTest`
Expected: Compilation failure

- [ ] **Step 3: Create NoOpDraftHouseSessionStore**

```java
package io.casehub.drafthouse;

import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.Collection;
import java.util.List;
import java.util.Optional;

@DefaultBean
@ApplicationScoped
public class NoOpDraftHouseSessionStore implements DraftHouseSessionStore {
    @Override public void save(SessionSnapshot snapshot) {}
    @Override public Optional<SessionSnapshot> load(String sessionId) { return Optional.empty(); }
    @Override public void remove(String sessionId) {}
    @Override public Collection<SessionSnapshot> loadAll() { return List.of(); }
}
```

- [ ] **Step 4: Create DraftHouseSessionRegistry**

```java
package io.casehub.drafthouse;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Collection;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

/** Manages active DraftHouseSession instances. */
@ApplicationScoped
public class DraftHouseSessionRegistry {

    private final ConcurrentHashMap<String, DraftHouseSession> sessions = new ConcurrentHashMap<>();
    private final DraftHouseSessionStore store;

    @Inject
    public DraftHouseSessionRegistry(DraftHouseSessionStore store) {
        this.store = store;
    }

    public DraftHouseSession create(String sessionId) {
        DraftHouseSession session = new DraftHouseSession(sessionId);
        if (sessions.putIfAbsent(sessionId, session) != null) {
            throw new IllegalStateException("Session already exists: " + sessionId);
        }
        return session;
    }

    public Optional<DraftHouseSession> find(String sessionId) {
        return Optional.ofNullable(sessions.get(sessionId));
    }

    public void remove(String sessionId) {
        DraftHouseSession session = sessions.remove(sessionId);
        if (session != null) {
            new java.util.ArrayList<>(session.activeFacets().keySet())
                .forEach(session::deactivateFacet);
            store.remove(sessionId);
        }
    }

    public Collection<DraftHouseSession> activeSessions() {
        return sessions.values();
    }
}
```

- [ ] **Step 5: Run tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=DraftHouseSessionRegistryTest`
Expected: PASS (4 tests)

- [ ] **Step 6: Commit**

```bash
git add server/runtime/src/main/java/io/casehub/drafthouse/DraftHouseSessionRegistry.java server/runtime/src/main/java/io/casehub/drafthouse/NoOpDraftHouseSessionStore.java server/runtime/src/test/java/io/casehub/drafthouse/DraftHouseSessionRegistryTest.java
git commit -m "feat(#117): add DraftHouseSessionRegistry with NoOp store

Refs #117"
```

### Task 5: Session REST endpoints

**Files:**
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/SessionResource.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/SessionResourceTest.java`

**Interfaces:**
- Consumes: `DraftHouseSessionRegistry` (Task 4)
- Produces: `GET /api/sessions`, `POST /api/sessions`, `DELETE /api/sessions/{id}`

- [ ] **Step 1: Write test for SessionResource**

```java
package io.casehub.drafthouse;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class SessionResourceTest {

    @Test
    void createAndListSessions() {
        String id = given()
            .contentType(ContentType.JSON)
            .body("{\"id\": \"test-session-1\"}")
            .when().post("/api/sessions")
            .then().statusCode(200)
            .extract().path("id");

        given()
            .when().get("/api/sessions")
            .then().statusCode(200)
            .body("$.size()", greaterThanOrEqualTo(1))
            .body("id", hasItem("test-session-1"));

        given()
            .when().delete("/api/sessions/test-session-1")
            .then().statusCode(204);
    }

    @Test
    void deleteNonExistentReturns404() {
        given()
            .when().delete("/api/sessions/nonexistent")
            .then().statusCode(404);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=SessionResourceTest`
Expected: 404 — endpoint does not exist

- [ ] **Step 3: Create SessionResource**

```java
package io.casehub.drafthouse;

import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import java.util.List;
import java.util.Map;
import java.util.UUID;

@Path("/api/sessions")
@Produces(MediaType.APPLICATION_JSON)
public class SessionResource {

    @Inject DraftHouseSessionRegistry registry;

    @GET
    public List<Map<String, Object>> list() {
        return registry.activeSessions().stream()
            .map(s -> Map.<String, Object>of(
                "id", s.id(),
                "created", s.created().toString(),
                "facets", s.activeFacets().keySet().stream().toList()
            ))
            .toList();
    }

    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    public Map<String, Object> create(Map<String, String> body) {
        String id = body != null && body.containsKey("id")
            ? body.get("id")
            : UUID.randomUUID().toString();
        DraftHouseSession session = registry.create(id);
        return Map.of("id", session.id(), "created", session.created().toString());
    }

    @DELETE
    @Path("/{id}")
    public Response delete(@PathParam("id") String id) {
        if (registry.find(id).isEmpty()) {
            return Response.status(404).build();
        }
        registry.remove(id);
        return Response.noContent().build();
    }
}
```

- [ ] **Step 4: Run tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=SessionResourceTest`
Expected: PASS (2 tests)

- [ ] **Step 5: Run all existing tests to verify no regressions**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime`
Expected: All existing tests pass

- [ ] **Step 6: Commit**

```bash
git add server/runtime/src/main/java/io/casehub/drafthouse/SessionResource.java server/runtime/src/test/java/io/casehub/drafthouse/SessionResourceTest.java
git commit -m "feat(#117): add session REST endpoints (GET/POST/DELETE /api/sessions)

Refs #117"
```

---

## Batch 3: Session-Level MCP Tools

### Task 6: Session and facet lifecycle MCP tools

**Files:**
- Create: `server/runtime/src/main/java/io/casehub/drafthouse/SessionMcpTools.java`
- Test: `server/runtime/src/test/java/io/casehub/drafthouse/SessionMcpToolsTest.java`

**Interfaces:**
- Consumes: `DraftHouseSessionRegistry` (Task 4), `DraftHouseSession` (Task 2)
- Produces: MCP tools `create_session`, `set_working_directory`, `list_facets`, `activate_facet`, `deactivate_facet`

- [ ] **Step 1: Write test for session MCP tools**

```java
package io.casehub.drafthouse;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkiverse.mcp.server.test.McpTestClient;
import org.junit.jupiter.api.Test;
import jakarta.inject.Inject;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class SessionMcpToolsTest {

    @Inject McpTestClient mcpClient;
    @Inject DraftHouseSessionRegistry registry;

    @Test
    void createSessionViaMcp() {
        var result = mcpClient.callTool("create_session", "{}");
        assertNotNull(result);
        assertFalse(result.isError());
    }

    @Test
    void listFacetsOnNewSession() {
        var createResult = mcpClient.callTool("create_session", "{}");
        String sessionId = extractSessionId(createResult);

        var listResult = mcpClient.callTool("list_facets",
            "{\"sessionId\": \"" + sessionId + "\"}");
        assertFalse(listResult.isError());
    }

    private String extractSessionId(Object result) {
        // Extract from tool response text content
        return result.toString().replaceAll(".*\"id\":\"([^\"]+)\".*", "$1");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=SessionMcpToolsTest`
Expected: Failure — tool `create_session` does not exist

- [ ] **Step 3: Create SessionMcpTools**

```java
package io.casehub.drafthouse;

import io.quarkiverse.mcp.server.Tool;
import io.quarkiverse.mcp.server.ToolArg;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.nio.file.Path;
import java.util.Map;
import java.util.UUID;

@ApplicationScoped
public class SessionMcpTools {

    @Inject DraftHouseSessionRegistry registry;

    @Tool(description = "Create a new DraftHouse session. Returns the session ID.")
    String create_session(
            @ToolArg(description = "Optional session ID. Auto-generated if omitted.") String sessionId,
            @ToolArg(description = "Optional working directory path for artifacts.") String workingDirectory) {
        String id = (sessionId != null && !sessionId.isBlank()) ? sessionId : UUID.randomUUID().toString();
        DraftHouseSession session = registry.create(id);
        if (workingDirectory != null && !workingDirectory.isBlank()) {
            session.setWorkingDirectory(Path.of(workingDirectory));
        }
        return "Session created: " + id;
    }

    @Tool(description = "Set or change the working directory for a session's artifact space.")
    String set_working_directory(
            @ToolArg(description = "Session ID") String sessionId,
            @ToolArg(description = "Absolute path to working directory") String path) {
        DraftHouseSession session = requireSession(sessionId);
        session.setWorkingDirectory(Path.of(path));
        return "Working directory set: " + path;
    }

    @Tool(description = "List available and active facets for a session.")
    String list_facets(@ToolArg(description = "Session ID") String sessionId) {
        DraftHouseSession session = requireSession(sessionId);
        var active = session.activeFacets().keySet();
        return "Active facets: " + (active.isEmpty() ? "none" : String.join(", ", active));
    }

    @Tool(description = "Activate a facet on a session. Registers the facet's MCP tools.")
    String activate_facet(
            @ToolArg(description = "Session ID") String sessionId,
            @ToolArg(description = "Facet name: voice, brainstorm, draft, review") String facetName) {
        DraftHouseSession session = requireSession(sessionId);
        // Facet lookup will be wired in Plan 2/3 when facet implementations exist
        return "Facet activation not yet wired — facet implementations needed. Requested: " + facetName;
    }

    @Tool(description = "Deactivate a facet on a session. Deregisters the facet's MCP tools.")
    String deactivate_facet(
            @ToolArg(description = "Session ID") String sessionId,
            @ToolArg(description = "Facet name") String facetName) {
        DraftHouseSession session = requireSession(sessionId);
        session.deactivateFacet(facetName);
        return "Facet deactivated: " + facetName;
    }

    private DraftHouseSession requireSession(String sessionId) {
        return registry.find(sessionId)
            .orElseThrow(() -> new IllegalArgumentException("Session not found: " + sessionId));
    }
}
```

- [ ] **Step 4: Run tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=SessionMcpToolsTest`
Expected: PASS

- [ ] **Step 5: Run all tests to verify no regressions**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add server/runtime/src/main/java/io/casehub/drafthouse/SessionMcpTools.java server/runtime/src/test/java/io/casehub/drafthouse/SessionMcpToolsTest.java
git commit -m "feat(#117): add session-level MCP tools (create_session, list_facets, activate/deactivate)

Refs #117"
```

### Task 7: Document management tools (lift from DebateMcpTools)

**Files:**
- Modify: `server/runtime/src/main/java/io/casehub/drafthouse/SessionMcpTools.java` — add document tools
- Modify: `server/runtime/src/test/java/io/casehub/drafthouse/SessionMcpToolsTest.java` — add document tests

**Interfaces:**
- Consumes: `DraftHouseSession.documentSet()` (Task 2)
- Produces: MCP tools `add_document_to_session`, `remove_document_from_session`, `list_session_documents`, `set_session_comparison`

Note: These tools operate on the session-level DocumentSet. The existing `add_document`, `remove_document`, `list_documents`, `set_comparison` tools on DebateMcpTools remain unchanged for now — they will be removed in Plan 2 when ReviewFacet takes over. The session-level tools use a `_to_session`/`_from_session`/`_session_` prefix to avoid MCP tool name collisions during the transition. They can be renamed once the old tools are removed.

- [ ] **Step 1: Add document management tests**

Add to `SessionMcpToolsTest`:

```java
@Test
void addAndListDocuments() {
    var createResult = mcpClient.callTool("create_session", "{}");
    String sessionId = extractSessionId(createResult);

    mcpClient.callTool("add_document_to_session",
        "{\"sessionId\": \"" + sessionId + "\", \"path\": \"/tmp/doc.md\", \"label\": \"Test Doc\"}");

    var listResult = mcpClient.callTool("list_session_documents",
        "{\"sessionId\": \"" + sessionId + "\"}");
    assertFalse(listResult.isError());
}
```

- [ ] **Step 2: Add document tools to SessionMcpTools**

Add methods to `SessionMcpTools.java`:

```java
@Tool(description = "Add a document to the session's working set.")
String add_document_to_session(
        @ToolArg(description = "Session ID") String sessionId,
        @ToolArg(description = "File path") String path,
        @ToolArg(description = "Display label") String label) {
    DraftHouseSession session = requireSession(sessionId);
    boolean added = session.documentSet().add(path, label != null ? label : path);
    return added ? "Document added: " + path : "Document already in set: " + path;
}

@Tool(description = "Remove a document from the session's working set.")
String remove_document_from_session(
        @ToolArg(description = "Session ID") String sessionId,
        @ToolArg(description = "File path") String path) {
    DraftHouseSession session = requireSession(sessionId);
    session.documentSet().remove(path);
    return "Document removed: " + path;
}

@Tool(description = "List documents in the session's working set.")
String list_session_documents(@ToolArg(description = "Session ID") String sessionId) {
    DraftHouseSession session = requireSession(sessionId);
    var docs = session.documentSet().documents();
    if (docs.isEmpty()) return "No documents in session.";
    StringBuilder sb = new StringBuilder();
    for (var doc : docs) {
        sb.append("- ").append(doc.path()).append(" (").append(doc.label()).append(")\n");
    }
    return sb.toString();
}

@Tool(description = "Set the comparison pair for the session.")
String set_session_comparison(
        @ToolArg(description = "Session ID") String sessionId,
        @ToolArg(description = "Path A") String pathA,
        @ToolArg(description = "Path B") String pathB) {
    DraftHouseSession session = requireSession(sessionId);
    session.documentSet().setComparison(pathA, pathB);
    return "Comparison set: " + pathA + " vs " + pathB;
}
```

- [ ] **Step 3: Run tests**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime -Dtest=SessionMcpToolsTest`
Expected: PASS

- [ ] **Step 4: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f server/pom.xml install -DskipTests && /opt/homebrew/bin/mvn -f server/pom.xml test -pl runtime`
Expected: All tests pass — no regressions

- [ ] **Step 5: Commit**

```bash
git add server/runtime/src/main/java/io/casehub/drafthouse/SessionMcpTools.java server/runtime/src/test/java/io/casehub/drafthouse/SessionMcpToolsTest.java
git commit -m "feat(#117): add session-level document management MCP tools

Refs #117"
```

---

## References

- [2026-08-20-composable-capability-architecture-design.md](../specs/issue-117-composable-capabilities/2026-08-20-composable-capability-architecture-design.md) — design spec (§1 Unified Session Container, §2 Facet Interface, §8 Module Boundaries)
- [decisions.md](../specs/issue-117-composable-capabilities/decisions.md) — D1 (session model), D5 (ToolManager), D6 (facet naming)
- `server/api/src/main/java/io/casehub/drafthouse/DocumentSet.java` — existing shared document state
- `server/api/src/main/java/io/casehub/drafthouse/DebateSessionStore.java` — existing SPI pattern for session persistence
- `server/api/src/main/java/io/casehub/drafthouse/DebateSessionRegistry.java` — existing registry pattern
- `server/runtime/src/main/java/io/casehub/drafthouse/DebateSessionRegistryImpl.java` — existing registry implementation
- `server/runtime/src/main/java/io/casehub/drafthouse/NoOpDebateSessionStore.java` — existing NoOp store pattern
- `server/runtime/src/main/java/io/casehub/drafthouse/WebSocketEventBus.java` — existing event bus (session-scoped events deferred to Plan 6: Layout)
- GitHub #117
