# Channel-based HIL — Design Spec

**Issue:** #100
**Date:** 2026-07-29
**Branch:** issue-100-channel-based-hil

## Summary

Enable concurrent human-in-the-loop participation in adversarial debates. Instead
of the stop-the-world FLAG_HUMAN model, the human participates as a first-class
channel participant — commenting, overriding, raising points, reprioritising, and
batch-approving alongside running agents.

Human actions flow into the Qhorus channel (for real-time display in all panels)
and are written to `decisions/human-round-{n}.md` files in the workspace directory
(for design-review agents to read at round start).

## Domain Model Changes

### AgentType (api module)

Add `HUMAN` to the enum:

```java
public enum AgentType {
    REV, IMP, SUPERVISOR, MODERATOR, SELECTOR, HUMAN
}
```

The human is a distinct debate participant, not an alias for an existing role.
Platform `ActorType.HUMAN` already exists and is set on every human message dispatch.

### EntryType (api module)

Add three new entry types:

```java
COMMENT,          // response to a point, no status change
HUMAN_OVERRIDE,   // human terminal status override
REPRIORITISE,     // priority change on existing point
```

Existing types reused:
- `RAISE` — for human-raised points (distinguished by `AgentType.HUMAN` in meta)
- `VERIFIED` / `DEFERRED` — for batch accept/reject

### DebateChannelProjection

Extend the hooks in `statusAfter()`:
- `"COMMENT"` → `null` (thread entry, no status change)
- `"HUMAN_OVERRIDE"` → `"HUMAN_OVERRIDE"` (terminal)

Override `apply()` to intercept `REPRIORITISE` before base class processing (same
pattern as `ROUND_SNAPSHOT`). `REPRIORITISE` modifies `PointClassification` on the
target point — builds a new `ConversationPoint` with updated priority, returns
updated state. Wrapped in try-catch per PP-20260610-a47ef5.

Update `DEBATE_CONFIG`:
- `statusEmoji`: `"HUMAN_OVERRIDE" → "👤"`
- `resolvedStatuses`: add `"HUMAN_OVERRIDE"` (terminal)
- `entryTypeLabel`: `"COMMENT" → "commented"`, `"HUMAN_OVERRIDE" → "overrode"`, `"REPRIORITISE" → "reprioritised"`
- `roleLabel`: add `"HUMAN" → "HUM"`

### DebateSession

Add `volatile String workspacePath` field, set by `loadWorkspace` in `DebateMcpTools`.
Null if no workspace loaded — file writing silently skipped.

Human participant registered lazily via `registerIfAbsent(HUMAN, ...)` on first
action. Deregistered by existing `endDebate` cleanup (iterates `participants()`).

### DebateStreamEntry

No changes needed — `EntryType.valueOf()` and `AgentType.valueOf()` handle new
values automatically.

## REST API — HumanActionResource

New JAX-RS resource at `/api/debate/{id}/human`. All endpoints:
1. Parse session UUID, resolve DebateSession (404 if missing)
2. Register human sender lazily via `session.registerIfAbsent(HUMAN, ...)`
3. Derive current round from projected channel state (max round across points)
4. Encode with `DebateProtocol.META_SENTINEL`
5. Dispatch via `messageService.dispatch(...)` with `actorType(ActorType.HUMAN)`
6. Project + push via WebSocket (trackAndPush pattern)
7. Write to decisions file via `DecisionFileWriter`
8. Return `200 {"status":"ok"}` or `400`/`404`

### Endpoints

**POST /api/debate/{id}/human/comment**
```json
{ "pointId": "uuid", "content": "string" }
```
Dispatches `COMMENT` entry. `inReplyTo` resolved via `findByCorrelationId(pointId)`.
MessageType: `RESPONSE`.

**POST /api/debate/{id}/human/raise**
```json
{
  "content": "string",
  "priority": "P1|P2|P3",
  "location": "string (optional)",
  "side": "A|B (optional)",
  "startLine": 45,
  "endLine": 52,
  "selectedText": "string (optional)"
}
```
Dispatches `RAISE` entry with `role=HUMAN`. Generates a new `correlationId` (pointId).
Selection fields stored in meta if present. MessageType: `QUERY`.

**POST /api/debate/{id}/human/override**
```json
{ "pointId": "uuid", "reason": "string" }
```
Dispatches `HUMAN_OVERRIDE` entry. `inReplyTo` resolved via `findByCorrelationId`.
MessageType: `DONE`.

**POST /api/debate/{id}/human/prioritise**
```json
{ "pointId": "uuid", "newPriority": "P1|P2|P3" }
```
Dispatches `REPRIORITISE` entry with `newPriority` in meta. `inReplyTo` resolved.
MessageType: `RESPONSE`.

**POST /api/debate/{id}/human/batch**
```json
{ "pointIds": ["uuid1", "uuid2"], "verdict": "VERIFIED|DEFERRED" }
```
Dispatches one entry per pointId with the verdict as entry type. All with
`actorType(ActorType.HUMAN)`, `role=HUMAN`. MessageType: `DONE` for VERIFIED,
`DECLINE` for DEFERRED.

### Dependencies

Injected: `DebateSessionRegistry`, `MessageService`, `InstanceService`,
`ProjectionService`, `DebateChannelProjection`, `WebSocketEventBus`,
`DebateEventResource`, `DecisionFileWriter`.

### Round Derivation

Human actions don't carry an explicit round. The resource projects the channel
state (same fold as `trackAndPush`) and takes the max round number from point
entries. Defaults to 1 if no entries exist.

### sender() Pattern

Not extracted from DebateMcpTools — replicated as the same 3-line pattern:
`session.registerIfAbsent(HUMAN, () -> instanceService.register(id, desc, tags))`.
The instance ID follows the existing convention: `"drafthouse-human-{sessionId}"`.

## Decisions File Writer

### DecisionFileWriter

Utility class that appends human decisions to structured markdown files in the
workspace's `decisions/` directory.

```java
decisionFileWriter.append(workspacePath, round, section, pointId, content)
```

- Creates `decisions/` directory if it doesn't exist
- Opens `decisions/human-round-{n}.md` — creates with header if new, appends if exists
- Appends entry under the correct `## Section` heading (Comments, Overrides,
  New Points, Priority Changes, Batch Decisions)
- Section heading written once per type per round (on first entry of that type)
- Synchronous file I/O (human actions are low-frequency)
- Silently skipped if `session.workspacePath()` is null

### File Format

```markdown
# Human Decisions — Round 3

## Comments

### R1-05
The original wording was intentional — this is a legal requirement.

## Overrides

### R1-03 → HUMAN_OVERRIDE
Both agents missed that this is required by compliance.

## New Points

### H-<uuid-short> — §3.2, lines 45-52
**Priority:** P1
The error handling path doesn't account for network timeouts.

## Priority Changes

### R1-07 → P1 (was P3)
This is more critical than the reviewer assessed.

## Batch Decisions

Approved: R1-01, R1-02, R1-04
```

### WorkspaceWatcher Interaction

The watcher's `RESPONSE_FILE` pattern matches `reviewer-round-N.md` and
`implementer-round-N.md`. Files in `decisions/` don't match — no loop risk.

## UI Changes

### Channel Feed (`<channel-feed>`)

- `AGENT_LABELS`: add `HUMAN → "Human"`
- `ENTRY_TYPE_LABELS`: add `COMMENT → "commented"`, `HUMAN_OVERRIDE → "overrode"`,
  `REPRIORITISE → "reprioritised"`
- Human entries get a distinct badge colour (warm accent, not agent blue/green)
  and a `👤` icon prefix on the role badge
- No structural rendering changes — human entries are DebateStreamEntry like all others

### Review Tracker (`<review-tracker>`)

- `ENTRY_TO_STATUS`: add `HUMAN_OVERRIDE → "HUMAN_OVERRIDE"`, `COMMENT → null` (no
  status change — handled by not adding to the map)
- `STATUS_ICON`: add `HUMAN_OVERRIDE → "👤"`
- `STATUS_ORDER`: add `HUMAN_OVERRIDE` after `DEFERRED`
- `RESOLVED_STATUSES`: add `"HUMAN_OVERRIDE"`
- `ACTION_SHORT`: add `COMMENT → "commented"`, `HUMAN_OVERRIDE → "overrode"`,
  `REPRIORITISE → "reprioritised"`
- `AGENT_SHORT`: add `HUMAN → "HUM"`
- **Action buttons per unresolved point:**
  - 💬 Comment — inline text input below the point
  - 👤 Override — dropdown with reason field
  - ↕ Priority — P1/P2/P3 dropdown
- Action buttons hidden on resolved points
- All buttons POST to `/api/debate/{id}/human/*` endpoints

### Document Diff (`<document-diff>`)

- **"Raise Point" button** — appears when text is selected in either panel
- Opens inline form: content field (pre-filled with selection context), priority
  dropdown (P1/P2/P3), location auto-populated from selection position
- Submit POSTs to `/api/debate/{id}/human/raise`
- Uses local selection state directly (side, startLine, endLine, selectedText)

### Batch Accept (Review Tracker)

- At round boundaries (new entries with higher round than previous), a batch
  action bar appears at the top of the tracker:
  "N low-priority points open — Accept all / Defer all"
- Dispatches `POST /api/debate/{id}/human/batch` with collected pointIds
- Only shown when 2+ unresolved LOW-priority points exist

## Protocols Honoured

| Protocol | How |
|----------|-----|
| PP-20260607-508f7b (actorType) | `actorType(ActorType.HUMAN)` on every dispatch |
| PP-20260608-d94c7d (sentinel) | All encoding via `DebateProtocol.META_SENTINEL` |
| PP-20260610-a47ef5 (apply no throw) | REPRIORITISE handler in try-catch |
| PP-20260604-6e8d5d (MCP errors) | N/A — REST endpoints use HTTP status codes |
| PP-20260608-21c69f (instance cleanup) | Human instance via registerIfAbsent, cleaned by endDebate |

## Out of Scope

- **design-review skill changes** — reading `decisions/`, prompt updates (soredium,
  separate approval required)
- **Authentication** — DraftHouse runs locally, consistent with existing endpoints
- **Human action persistence beyond channel + decisions/ files** — channel projection
  already persists ConversationState
