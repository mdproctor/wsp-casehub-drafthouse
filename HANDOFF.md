# Handover — 2026-07-06

**Branch:** `issue-94-data-model-mapping`

## What Happened This Session

### #94 Research complete and closed

Produced a field-by-field data model mapping between design-review (Python, file-based) and DraftHouse (Java, channel-based). The research document is at `specs/issue-94-data-model-mapping/research.md` in the workspace and posted on the GitHub issue.

**Critical findings that shape #95:**

1. **ADDRESSED ≠ AGREED.** design-review's FIXED/ADDRESSED is non-terminal (awaits reviewer verification). DraftHouse's AGREED is terminal. The adapter must emit QUALIFY for FIXED responses — not AGREE — so the point stays active until the reviewer verifies. This means in the review tracker, implementor-addressed issues show as "active" (amber, ⟳) not "resolved" (green, ✓) until the reviewer's confirmation round.

2. **VERIFIED status must be added.** When the reviewer confirms a fix is correct (`resolved` confirmation in reviewer-N+1.md), the adapter emits a new VERIFIED entry type. This needs:
   - `statusAfter("VERIFIED") → "VERIFIED"` in `DebateChannelProjection`
   - `"VERIFIED"` added to `resolvedStatuses` in `ConversationRendererConfig`
   - `VERIFIED` added to `EntryType` enum
   - Review tracker panel: add VERIFIED to `ENTRY_TO_STATUS` map, `STATUS_ORDER`, `STATUS_ICON`

3. **DEFERRED status must be added.** For auto-escalated issues (2+ contested rounds) or explicitly deferred issues. Same scope as VERIFIED — projection, renderer config, panel.

4. **Entry emission order within a round:**
   - Reviewer RAISE entries (from reviewer-N.md)
   - Implementor QUALIFY/COUNTER/FLAG_HUMAN entries (from implementor-N.md)
   - Reviewer VERIFIED/DISPUTE/AGREE entries (from reviewer-N+1.md confirmations)
   - MEMO entries (assumptions, settled decisions, round summary)

5. **Point identity:** The adapter uses the design-review issue ID string (e.g., `R1-02`) directly as the `pointId` / `correlationId`. No UUID generation needed — DraftHouse's `ConversationPoint.id` is a String.

6. **Location mapping:** design-review's `section_ref` (e.g., `"4.2"`) maps to `PointClassification.location` as `§4.2`. The heading matcher in `drafthouse-diff.js` already handles `§N.N` format.

## Immediate Next Step

**Start #95 — Build replay adapter for completed design-review workspaces.**

### What #95 builds

A Java parser in DraftHouse's `server/runtime` that reads a completed design-review workspace directory and projects it as `DebateStreamEntry` events into a `DebateSession`.

### Architecture

```
load_workspace(path)
  │
  ├─ Read workspace metadata
  │   ├─ .spec-path → primary document path
  │   ├─ .mode → session type label
  │   └─ context.md → optional initial MEMO
  │
  ├─ Create DebateSession
  │   ├─ channelId = new UUID
  │   ├─ debateSessionId = workspace directory name
  │   ├─ Register REV + IMP participants
  │   └─ Add spec as primary document
  │
  ├─ Parse response files (round by round)
  │   ├─ reviewer-N.md → RAISE entries (one per ### heading)
  │   │   Fields: entryType=RAISE, role=REV, round=N
  │   │   pointId=R{N}-{seq}, content=body, location=§{section_ref}
  │   │
  │   ├─ implementor-N.md → response entries (one per ### R{N}-{seq}: STATUS)
  │   │   FIXED → entryType=QUALIFY, role=IMP (NOT AGREE — see finding #1)
  │   │   REJECTED → entryType=COUNTER, role=IMP
  │   │   ESCALATED → entryType=FLAG_HUMAN, role=IMP
  │   │
  │   └─ reviewer-(N+1).md confirmations → terminal entries
  │       "resolved" → entryType=VERIFIED, role=REV (new — see finding #2)
  │       "accepted" → entryType=AGREE, role=REV
  │       "still open" → entryType=DISPUTE, role=REV
  │
  ├─ Parse tracker.md for metadata
  │   ├─ commit_before / commit_after → metadata on entries
  │   ├─ assumptions → MEMO entries
  │   └─ settled decisions → MEMO entries
  │
  └─ Push all entries via WebSocketEventBus.pushDebateEntries()
```

### Implementation approach — reimplement parsing in Java

The design-review Python parser (`parser.py`) uses regex to extract issues, responses, confirmations, assumptions, and settled decisions. The adapter reimplements this in Java because:
- Self-contained — no Python subprocess dependency
- The regex patterns are well-defined and stable (read from parser.py source in #94)
- The adapter targets `DebateStreamEntry`, not the Python dataclasses
- When #96 lands (JSONL sidecar), the Java parser becomes a fallback for old workspaces

### Key regex patterns to port (from parser.py)

| Pattern | Purpose | Python source |
|---------|---------|---------------|
| `SIGNAL: (APPROVED\|CONTINUE\|DECISION_NEEDED)` | Extract round signal | `_SIGNAL_RE` |
| `^#{2,3}\s+(.+?)\s*$` | Heading extraction for new issues | `_HEADING_RE` |
| `R(\d+)-(\d+)` | Issue ID format | `_ISSUE_ID_RE` |
| `^#{2,3}\s+R(\d+)-(\d+)\s*[:\s—-]+\s*(FIXED\|REJECTED\|ESCALATED)` | Implementor response | `_ISSUE_RESPONSE_RE` |
| `R(\d+)-(\d+).*?(resolved\|accepted\|still\s+open)` | Reviewer confirmation | `_CONFIRMATION_RE` |
| `§(\d+(?:\.\d+)*)\|Section\s+(\d+(?:\.\d+)*)` | Section reference | `_SECTION_REF_RE` |

### DraftHouse changes needed alongside the adapter

1. **`EntryType` enum** — add `VERIFIED` (and optionally `DEFERRED`)
2. **`DebateChannelProjection.statusAfter()`** — add cases for `"VERIFIED" → "VERIFIED"` and `"DEFERRED" → "DEFERRED"`
3. **`ConversationRendererConfig`** in `DebateChannelProjection` — add `"VERIFIED"` and `"DEFERRED"` to `resolvedStatuses`, add status emojis and entry type labels
4. **Review tracker panel** (`drafthouse-review-tracker.js`) — add VERIFIED and DEFERRED to `ENTRY_TO_STATUS`, `STATUS_ORDER`, `STATUS_ICON`
5. **MCP tool or REST endpoint** — `load_workspace(path)` exposed as an MCP tool via `DebateMcpTools` or as a new resource

### Test data

The example workspace at `~/adr/casehub-drafthouse/document-workbench-20260705-224726/` is the primary test dataset:
- 3 rounds, 18 issues (13 round 1, 5 round 2)
- 17 VERIFIED, 1 ACCEPTED, 0 DEFERRED
- 1 assumption, 4 settled decisions
- Total cost: $15.10

E2E test should load this workspace and verify:
- 18 review points appear in the tracker
- 17 show VERIFIED status, 1 shows AGREED (ACCEPTED)
- Section highlights work on at least one point with a §-reference
- Debate panel shows threaded conversation for a multi-round issue (e.g., R1-01: raised round 1, fixed round 1, verified round 2)

### Files to create/modify

| File | Change |
|------|--------|
| `server/runtime/.../debate/WorkspaceParser.java` | **New.** Java port of parser.py regex extraction |
| `server/runtime/.../debate/WorkspaceReplayAdapter.java` | **New.** Orchestrates parse → DebateStreamEntry list → push |
| `server/runtime/.../DebateMcpTools.java` | **Modify.** Add `load_workspace` MCP tool |
| `server/api/.../debate/EntryType.java` | **Modify.** Add VERIFIED, DEFERRED |
| `server/runtime/.../debate/DebateChannelProjection.java` | **Modify.** statusAfter + renderer config |
| `server/runtime/.../webui/src/panels/drafthouse-review-tracker.js` | **Modify.** VERIFIED/DEFERRED in status maps |
| `server/runtime/src/test/.../e2e/WorkspaceReplayE2ETest.java` | **New.** E2E test with example workspace |
| `server/runtime/src/test/.../debate/WorkspaceParserTest.java` | **New.** Unit tests for Java regex parser |

## What's Left

| # | Title | Scale | Complexity | Blocked by | Notes |
|---|-------|-------|------------|------------|-------|
| #93 | Document workbench (epic) | XL | High | — | 8 child issues |
| ~~#94~~ | ~~Research: data model mapping~~ | ~~M~~ | ~~High~~ | — | **Closed** |
| #95 | Replay adapter | L | Med | ~~#94~~ | **Start here — now unblocked** |
| #96 | design-review structured output | M | Med | #95 | Changes to soredium skill |
| #97 | Chunked orchestration research | M | High | #96 | Cost/UX tradeoff study |
| #98 | Document timeline | L | Med | #95 | Version navigation |
| #99 | Live workspace watching | M | Med | #95 | Real-time monitoring |
| #100 | Channel-based HIL | L | High | #97, #99 | Concurrent human participation |
| #101 | Panel extraction | XL | High | all above | @casehubio components |
| #92 | Adopt pages-push typed protocol SDK | S | Low | — | Independent |
| #89 | AgentProvider migration | M | Med | — | Independent |

## References

| Context | Where |
|---------|-------|
| #94 research document | `specs/issue-94-data-model-mapping/research.md` (workspace) |
| #94 GitHub comment | casehubio/drafthouse#94 (full document posted) |
| Exploration spec | `docs/superpowers/specs/2026-07-05-document-workbench-exploration.md` |
| Example workspace | `~/adr/casehub-drafthouse/document-workbench-20260705-224726/` |
| design-review parser | `~/.claude/skills/design-review/parser.py` |
| design-review tracker | `~/.claude/skills/design-review/tracker.py` |
| P5 conversation protocol | casehub-blocks `io.casehub.blocks.conversation` |
| DraftHouse debate model | `server/api/src/main/java/io/casehub/drafthouse/debate/` |
| DebateStreamEntry | `server/runtime/.../debate/DebateStreamEntry.java` |
| DebateChannelProjection | `server/runtime/.../debate/DebateChannelProjection.java` |
| Review tracker panel | `server/runtime/.../webui/src/panels/drafthouse-review-tracker.js` |
| Epic | casehubio/drafthouse#93 |
