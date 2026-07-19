# Live Workspace Watching — Design Spec

**Issue:** #99
**Date:** 2026-07-20
**Epic:** #93 (Document Workbench)

## Problem

`load_workspace` replays completed design-review workspaces as interactive debate
sessions. But design-review runs take 10-30 minutes across multiple rounds — there
is no way to watch a review in progress. The user must wait for completion, then
load the workspace after the fact.

## Solution

Extend `load_workspace` to auto-detect whether a review is still running. If it is,
replay historical rounds (existing behaviour), then start a `WorkspaceWatcher` that
monitors the workspace directory for new files. When a new response file appears,
parse the new round and dispatch debate entries in near-real-time. Progress updates
from `progress.log` are pushed as metadata events on a separate topic.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Trigger | Auto-detect in `load_workspace` | Single entry point — user doesn't need to know if the review is running or complete |
| File watching | `io.methvin:directory-watcher` | Native macOS Carbon FS Events via JNA. Cross-platform. Already used in the ecosystem |
| Progress events | Separate metadata topic | Progress is transient operational status, not part of the review conversation. Keeps the debate feed clean |
| Component boundary | New `WorkspaceWatcher` class | Different lifecycle from replay adapter (long-lived vs one-shot). Composes with existing parsing and dispatch infrastructure |
| UI | New `<workspace-status>` topbar element | Dedicated to progress display. Context-gauge stays focused on token tracking |
| blocks-ui | Nothing to extract | All components are tied to design-review workspace semantics. No generic pattern worth extracting yet |

## Component Architecture

```
load_workspace (MCP tool — DebateMcpTools)
  │
  ├─► WorkspaceParser.parse()           — full workspace parse (existing)
  ├─► WorkspaceReplayAdapter.replay()   — dispatch historical rounds (existing)
  │
  └─► if review still running:
      └─► WorkspaceWatcher.start()      — watch for new files
            ├─► new response file  → parseRound(N) → dispatch debate entries
            ├─► progress.log change → tail new lines → push metadata
            └─► terminal state     → stop watching, push completion event
```

### WorkspaceWatcher

New class: `io.casehub.drafthouse.debate.WorkspaceWatcher`

**Responsibilities:**
- Hold a `DirectoryWatcher` handle from `io.methvin:directory-watcher`
- Track state: `lastReplayedRound`, `progressLogOffset`, `existingIssueIds`
- On new response file: parse single round via WorkspaceParser, dispatch entries
- On progress.log change: tail new lines, parse and push metadata events
- On terminal state: stop watching, push completion event, remove from registry

**Dependencies (constructor-injected):**
- `MessageService` — dispatch channel messages
- `InstanceService` — register senders
- `ChannelGateway` — channel init (already done by replay)
- `WebSocketEventBus` — push debate entries and metadata to browser
- `DebateSession` — the session being watched

**Lifecycle:**
- `start(Path workspacePath, int startFromRound, Set<String> existingIssueIds)` — begin watching
- `stop()` — close DirectoryWatcher, clean up
- Implements `Closeable`

### WorkspaceParser Changes

Expose per-round parsing methods as package-visible (currently private):
- `parseRoundFromMarkdown(Path responsesDir, int roundNum, Set<String> existingIds)` → `ParsedRound`
- `parseRoundFromJsonl(Path responsesDir, int roundNum)` → `ParsedRound`
- `discoverMaxRound(Path responsesDir)` → `int`
- `parseTracker(Path workspaceDir)` → `List<ParsedTrackerEntry>` (already static)

No new public API. These are package-internal for WorkspaceWatcher.

### Active Watcher Registry

`ConcurrentHashMap<String, WorkspaceWatcher>` field on `DebateMcpTools`, keyed by
debate session ID.

- **Idempotency:** repeated `load_workspace` calls for the same workspace detect the
  existing watcher and return `"status":"already_watching"`
- **Cleanup on completion:** watcher removes itself when terminal state detected
- **Cleanup on end_debate:** `DebateMcpTools` checks map, stops watcher if present
- **Server shutdown:** `@PreDestroy` closes all active watchers

## File Watching — Event Flow

### Watched paths

DirectoryWatcher monitors the workspace directory recursively. Events are filtered to:

| Path pattern | Event | Action |
|-------------|-------|--------|
| `responses/reviewer-N.md` or `.jsonl` | CREATE | Parse round N, dispatch RAISE entries |
| `responses/implementor-N.md` or `.jsonl` | CREATE | Re-parse round N for responses, dispatch QUALIFY/COUNTER/FLAG_HUMAN entries |
| `progress.log` | MODIFY | Tail new lines, push metadata events |
| `tracker.md` | MODIFY | Re-parse tracker for updated issue statuses |
| Everything else | * | Ignored |

### New response file flow

```
reviewer-3.md appears
  → parseRoundFromMarkdown(responsesDir, 3, existingIssueIds)
  → Dispatch RAISE entries for new issues
  → Dispatch CONFIRMATION entries (verdicts on previous round's responses)
  → Dispatch MEMO entries (assumptions, settled decisions)
  → Update existingIssueIds with new issue IDs
  → Push debate entries via eventBus.pushDebateEntries()

implementor-3.md appears  
  → Re-parse round 3 (responses now available)
  → Dispatch QUALIFY/COUNTER/FLAG_HUMAN entries (response entries only)
  → Re-parse tracker.md for status updates
  → Push debate entries via eventBus.pushDebateEntries()
  → Emit ROUND_SNAPSHOT if spec commit found in tracker
  → Update lastReplayedRound = 3
```

Reviewer files always appear before implementor files (reviewer runs first, then
implementor responds). The watcher processes them in arrival order.

**Partial round at replay boundary:** When replay runs, it processes all existing
files. If `reviewer-N.md` exists but `implementor-N.md` does not, the replay adapter
dispatches RAISE/CONFIRMATION/MEMO entries for round N. The watcher then starts and
only needs to handle `implementor-N.md` when it appears. The watcher responds to
file CREATE events — it never re-processes files that existed before watching started.

### Progress.log tailing

On each MODIFY event for progress.log:
- Read from `lastProgressOffset` to EOF
- Parse new lines against known patterns
- Push metadata events via `eventBus.pushMetadata(channelId, "workspace-progress", payload)`
- Update `lastProgressOffset`

**Parsed line patterns:**

| Pattern | Metadata type |
|---------|--------------|
| `[HH:MM:SS]   Reviewer/Implementor (fresh session)...` | `AGENT_START` |
| `[HH:MM:SS]     [Ns] reviewer/implementor: <text>` | `AGENT_STATUS` |
| `[HH:MM:SS]   Reviewer/Implementor done ($X.XX)` | `AGENT_COMPLETE` |
| `[HH:MM:SS]   N new issue(s) raised` | `ISSUES_RAISED` |
| `[HH:MM:SS]   Round N complete — ~$X.XX/round, $X.XX cumulative` | `ROUND_COMPLETE` |
| `REVIEW DONE` / `REVIEW PAUSED` / `REVIEW FAILED` | `REVIEW_TERMINAL` |

### Metadata event payload

```json
{"type": "AGENT_STATUS", "round": 2, "agent": "reviewer",
 "message": "Reading ARC42STORIES and exploring architecture docs",
 "elapsed": 60}

{"type": "AGENT_COMPLETE", "round": 2, "agent": "reviewer",
 "cost": 1.93}

{"type": "ROUND_COMPLETE", "round": 1, "cost": 4.99,
 "cumulativeCost": 4.99, "issuesRaised": 13}

{"type": "REVIEW_TERMINAL", "finalState": "DONE"}
```

### Terminal state handling

When `REVIEW DONE`, `REVIEW PAUSED`, or `REVIEW FAILED` is detected in progress.log:
1. Push `REVIEW_TERMINAL` metadata event
2. Call `stop()` — close DirectoryWatcher
3. Remove from active watchers map in DebateMcpTools

## load_workspace Integration

**Modified flow in `DebateMcpTools.loadWorkspace()`:**

After the existing replay logic completes:

```java
// Detect review state
boolean reviewComplete = isReviewComplete(wsPath);

if (!reviewComplete) {
    int lastRound = parseResult.rounds().size();
    Set<String> existingIds = collectIssueIds(parseResult);
    
    var watcher = new WorkspaceWatcher(
        messageService, instanceService, channelGateway, eventBus,
        session, revSender, impSender);
    watcher.start(wsPath, lastRound, existingIds);
    activeWatchers.put(session.debateSessionId(), watcher);
}
```

`isReviewComplete()` reads the last few lines of progress.log and checks for
`REVIEW DONE` / `REVIEW PAUSED` / `REVIEW FAILED`.

**Return value changes:**
- Complete review: `"status":"loaded"` (unchanged)
- Watching review: `"status":"watching"`
- Already watching: `"status":"already_watching"`

## Thread Model

- `DirectoryWatcher.watchAsync()` runs on its own thread (CompletableFuture)
- Event callbacks fire on the watcher thread
- `MessageService.dispatch()` is CDI-managed, thread-safe
- `WebSocketEventBus` uses ConcurrentHashMap, `sendText()` via Mutiny subscribe — safe from any thread
- No Vert.x event loop interaction — all dispatch is from a background thread

## UI — Workspace Status Element

New Lit element: `<workspace-status>` in `server/runtime/src/main/webui/src/panels/workspace-status.ts`

**Behaviour:**
- Subscribes to `workspace-progress` pages events
- Only visible when a workspace is being watched (hidden otherwise)
- Renders in the topbar area

**Display states:**

| State | Render |
|-------|--------|
| Agent working | "Round 2 — reviewer (1m 30s)" with elapsed timer |
| Agent status update | Updates message text: "reviewer: Reading issues #88, #85..." |
| Agent complete | "Round 2 — reviewer done ($1.93), implementor working..." |
| Round complete | "Round 2 complete — $4.99" (brief flash, then next round or idle) |
| Review complete | "Review complete — 3 rounds, $12.50" |
| Review paused/failed | "Review paused" / "Review failed" with appropriate styling |

**Elapsed timer:** Client-side interval timer started on `AGENT_START`, stopped on
`AGENT_COMPLETE`. No server polling needed — the client counts seconds locally
between status events.

**Element lifecycle:** `configure({debateSessionId})` from workbench layout. Hidden by
default until the first `workspace-progress` event arrives. Cleaned up in
`disconnectedCallback()`.

## New Dependency

```xml
<dependency>
    <groupId>io.methvin</groupId>
    <artifactId>directory-watcher</artifactId>
</dependency>
```

Added to `server/runtime/pom.xml`. Check Maven Central for latest version.

## Testing Strategy

**Unit tests:**
- `WorkspaceWatcherTest` — mock MessageService/EventBus, create temp workspace dir,
  write files programmatically, assert correct debate entries dispatched
- Progress.log line parsing — extract metadata from known line formats
- Terminal state detection — verify watcher stops on REVIEW DONE/PAUSED/FAILED
- Idempotency — repeated load_workspace with active watcher returns already_watching

**Integration/E2E:**
- Load a workspace that has partial rounds, verify replay + watching mode activation
- Write a new response file to the watched directory, verify debate entries appear
  in the channel feed within seconds
- Write progress.log lines, verify metadata events reach the browser
- Write REVIEW DONE, verify watcher stops and completion event is pushed

## Out of Scope

- **Design-review script changes** — no POST-to-DraftHouse from the script side
- **blocks-ui extraction** — all components are workspace-specific
- **Checkpoint/decision files** — `checkpoint-N.md` and `decisions/` directory watching
  can be added later if needed
- **Multiple simultaneous watchers** — the registry supports it, but no UI for
  switching between watched workspaces (single active debate session model)
