# Research: design-review → DraftHouse Data Model Mapping

**Issue:** #94
**Date:** 2026-07-06
**Status:** Complete

## 1. Source Material

### design-review (Python, file-based)
- **parser.py** — regex extraction of signals, issues, confirmations, responses, assumptions, settled decisions
- **tracker.py** — issue lifecycle state machine (`Tracker`, `TrackedIssue`, `IssueStatus` enum), convergence detection, round summaries, tracker.md rendering
- **File layout:** `responses/reviewer-N.md`, `responses/implementor-N.md`, `tracker.md`, `progress.log`, `spec.md` (symlink), `context.md`, `decisions/`, `handovers/`, `agents/`

### DraftHouse (Java, channel-based)
- **DebateStreamEntry** — single event record: entryType, agentRole, round, content, pointId, subTaskId, priority, scope, location, sender, timestamp
- **P5 ConversationState** — fold accumulator: points (Map<String, ConversationPoint>), humanFlags, memos, subTaskFindings
- **ConversationPoint** — id, classification (priority, scope, location), thread (List<ThreadEntry>), status
- **ThreadEntry** — entryId, role, round, entryType, content
- **DebateChannelProjection** — folds messages into ConversationState via sentinel/isPointInitiator/statusAfter hooks
- **Review tracker panel** — client-side fold: filters entries by pointId, maps last entryType → status via ENTRY_TO_STATUS table

---

## 2. Field-by-Field Mapping

### 2.1 Issue / Review Point

| Field | design-review (`TrackedIssue`) | DraftHouse (`ConversationPoint` / `DebateStreamEntry`) | Gap |
|-------|-------------------------------|-------------------------------------------------------|-----|
| **Identity** | `issue_id: str` (e.g. `R1-02`) — round + sequence | `ConversationPoint.id` = `correlationId` from Qhorus message (UUID) | Adapter must generate stable pointIds from issue_id. UUID or the R-id string itself. |
| **Summary/title** | `summary: str` — from heading text | First line of RAISE entry `content` (extracted client-side in tracker panel) | design-review has an explicit title field; DraftHouse derives it from content. Adapter should set content's first line = title. |
| **Body** | `Issue.body: str` — markdown under heading | `ThreadEntry.content: String` — full content of the RAISE entry | Direct mapping. |
| **Round raised** | `round_raised: int` — from filename number (reviewer-N) | `ThreadEntry.round: int` — explicit metadata field | Direct mapping. |
| **Status** | `IssueStatus` enum: OPEN, ADDRESSED, VERIFIED, REJECTED, ACCEPTED, CONTESTED, DEFERRED | `ConversationPoint.status: String`: OPEN, AGREED, ACTIVE, DISPUTED, ESCALATED, DECLINED | **Major gap** — see §3 state machine mapping. |
| **Contested rounds** | `contested_rounds: int` — counter, auto-escalates at 2 | No equivalent. ConversationPoint has no escalation counter. | Gap: adapter could track via repeated DISPUTE entries, but auto-escalation logic has no DraftHouse equivalent. |
| **Commit evidence** | `commit_before: str`, `commit_after: str` — git SHAs | Not modeled. | **Gap.** DraftHouse has no commit evidence fields. Must be added to support VERIFIED semantics. |
| **Section reference** | `section_ref: str` — e.g. `"4.2"` (regex-extracted from body) | `PointClassification.location: String` — metadata field | Partial mapping. design-review's section_ref is structured (`§N.N`); DraftHouse's location is a freeform string. The adapter can set location = `§{section_ref}`. |
| **Rationale** | `rationale: str` — implementor's reasoning for FIXED/REJECTED | Part of ThreadEntry.content in AGREE/COUNTER response entries | In design-review this is a separate field; in DraftHouse it's embedded in response content. Adapter maps it into content. |
| **Notes** | `notes: str` — e.g. auto-escalation reason | No equivalent field. | Minor gap. Could be appended to content or added as metadata. |
| **Priority** | Not modeled (issues have no priority) | `PointClassification.priority: Priority` (HIGH, MEDIUM, LOW) | **Reverse gap.** DraftHouse has priority; design-review doesn't. Exploration spec §6.2 recommends design-review add structured PRIORITY metadata. |
| **Scope** | Not modeled | `PointClassification.scope: String` | Reverse gap. DraftHouse has scope; design-review doesn't. |

### 2.2 Agent Roles

| design-review role | DraftHouse `AgentType` | P5 protocol | Notes |
|--------------------|----------------------|-------------|-------|
| Reviewer (file: `reviewer-N.md`) | `REV` | String `"REV"` | Direct mapping. |
| Implementor (file: `implementor-N.md`) | `IMP` | String `"IMP"` | Direct mapping. |
| Python PM (orchestrator) | No equivalent | — | The PM is the orchestrator, not a conversation participant. No entry needed. |
| — | `SUPERVISOR` | String `"SUPERVISOR"` | DraftHouse-only role. Not present in design-review. |
| — | `MODERATOR` | String `"MODERATOR"` | DraftHouse-only role. |
| — | `SELECTOR` | String `"SELECTOR"` | DraftHouse-only role. |

### 2.3 Entry Types

| design-review signal | Extraction source | DraftHouse `EntryType` | Adapter mapping |
|---------------------|-------------------|----------------------|-----------------|
| New issue (### heading in reviewer) | `extract_new_issues()` → `Issue(issue_id, title, body)` | `RAISE` | Create DebateStreamEntry with entryType=RAISE, role=REV, round=N, content=body, pointId=issue_id |
| FIXED response | `extract_issue_responses()` → `IssueResponse(status="FIXED")` | `AGREE` | Create entry with entryType=AGREE, role=IMP, content=rationale+body |
| REJECTED response | `extract_issue_responses()` → `IssueResponse(status="REJECTED")` | `COUNTER` | Create entry with entryType=COUNTER, role=IMP, content=rationale |
| ESCALATED response | `extract_issue_responses()` → `IssueResponse(status="ESCALATED")` | `FLAG_HUMAN` | Create entry with entryType=FLAG_HUMAN, role=IMP |
| "resolved" confirmation | `extract_confirmations()` → `Confirmation(is_resolved=True)` | No direct equivalent | **Gap.** This is VERIFIED — reviewer confirms fix exists. See §3. |
| "accepted" confirmation | `extract_confirmations()` → `Confirmation(is_accepted=True)` | `AGREE` | Entry from role=REV, meaning reviewer accepts the rejection. |
| "still open" confirmation | `extract_confirmations()` → `Confirmation(is_resolved=False)` | `DISPUTE` | Reviewer doesn't accept the fix. Creates a DISPUTE entry. |
| SIGNAL: APPROVED | `extract_signal()` → `Signal(signal_type="APPROVED")` | No equivalent | Orchestrator-level signal, not a conversation entry. Could be mapped to MEMO. |
| SIGNAL: CONTINUE | `extract_signal()` → `Signal(signal_type="CONTINUE")` | No equivalent | Same — orchestrator signal. |
| SIGNAL: DECISION_NEEDED | `extract_signal()` → `Signal(signal_type="DECISION_NEEDED")` | `FLAG_HUMAN` | Maps to FLAG_HUMAN with the description as content. |
| ASSUMPTION | `extract_assumptions()` → `[str]` | No equivalent | **Gap.** Could map to MEMO from REV role, or a new ASSUMPTION entry type. |
| SETTLED decision | `extract_settled_decisions()` → `SettledDecision(text, from_issue)` | No equivalent | **Gap.** Could map to MEMO, but loses the structured from_issue link. |

### 2.4 Session / Workspace

| Field | design-review (workspace) | DraftHouse (`DebateSession`) | Gap |
|-------|--------------------------|------------------------------|-----|
| **Session identity** | Workspace directory name (timestamped) | `debateSessionId: String` | Adapter generates a session ID from workspace name. |
| **Channel** | N/A — file-based | `channelId: UUID`, `channelName: String` | Adapter must create/reuse a Qhorus channel. |
| **Project name** | `Tracker.project_name: str` | Embedded in channel name | Minor — adapter passes project name. |
| **Start date** | `Tracker.start_date: str` | Not explicitly stored (derived from first entry timestamp) | Minor gap. |
| **Current round** | `Tracker.current_round: int` | Derived from max round in entries | No gap — both derive it. |
| **Documents** | `spec.md` (symlink to actual file), `.spec-path` (absolute path) | `DocumentSet` → `List<DocumentEntry>` | Adapter reads `.spec-path`, adds as primary document. |
| **Comparison** | N/A (single-file review) or implicit via git commits | `ComparisonPair(pathA, pathB)` | **Gap.** For replay, adapter needs to synthesize comparison from git commits (before/after per round). Requires `DocumentSnapshot` extension (§4.1 of exploration spec). |
| **Participants** | Reviewer + Implementor (implicit from file names) | `ConcurrentHashMap<AgentType, String>` | Adapter registers REV and IMP participants. |
| **Selection scope** | N/A | `SelectionScope(side, startLine, endLine, selectedText)` | Not applicable for replay. |

### 2.5 Round-Level Data

| Field | design-review (`RoundSummary`) | DraftHouse equivalent | Gap |
|-------|-------------------------------|-----------------------|-----|
| **Round number** | `round_num: int` | `ThreadEntry.round` / `DebateStreamEntry.round` | Direct mapping at entry level; no aggregate. |
| **Issues raised** | `raised: int` | Derivable (count RAISE entries in round) | No gap. |
| **Issues verified** | `verified: int` | Not derivable (no VERIFIED status yet) | Gap until VERIFIED is added. |
| **Issues accepted** | `accepted: int` | Derivable (count reviewer AGREE entries) | No gap. |
| **Issues open** | `open: int` | Derivable from ConversationState | No gap. |
| **Issues contested** | `contested: int` | Derivable (count DISPUTE entries) | No gap. |
| **Issues deferred** | `deferred: int` | Not modeled | Gap — no DEFERRED status in DraftHouse. |
| **Round cost** | progress.log `$X.XX/round` | Not modeled | **Gap.** Operational metric — adapter could emit as MEMO. |
| **Round duration** | progress.log timestamps | Not modeled | Gap — same as cost. |

### 2.6 Metadata and Ancillary

| design-review artifact | DraftHouse equivalent | Adapter handling |
|----------------------|----------------------|------------------|
| `progress.log` | No equivalent | Parse for timing/cost → emit as MEMOs or a new metadata topic. |
| `context.md` | Not modeled | Session context. Could become initial MEMO entry. |
| `.mode` (e.g. "spec-review") | Not modeled | Could inform phase/session type. |
| `.source-dirs` | Not modeled | Build context. Not needed for replay. |
| `agents/` directory | `ResolvedReviewer` (reviewer instructions) | Adapter reads agent configs → maps to reviewer metadata. |
| `decisions/` directory | Not modeled | HIL responses — channel → file direction (future). |
| `handovers/` directory | Not modeled | Not needed for replay. |

---

## 3. State Machine Mapping

### 3.1 design-review states

```
OPEN ──► ADDRESSED ──► VERIFIED (terminal)
  │          │
  │          └──► CONTESTED ──► ADDRESSED (retry)
  │                   │
  │                   └──► DEFERRED (terminal, auto after 2 contests)
  │
  ├──► REJECTED ──► ACCEPTED (terminal)
  │         │
  │         └──► CONTESTED ──► ADDRESSED
  │
  └──► DEFERRED (terminal, explicit)
```

Valid transitions (from tracker.py `_VALID_TRANSITIONS`):
- OPEN → ADDRESSED, REJECTED, DEFERRED
- ADDRESSED → VERIFIED, CONTESTED
- REJECTED → ACCEPTED, CONTESTED
- CONTESTED → ADDRESSED, DEFERRED
- VERIFIED → (terminal)
- ACCEPTED → (terminal)
- DEFERRED → (terminal)

### 3.2 DraftHouse statuses (from `DebateChannelProjection.statusAfter()`)

```
OPEN ──► AGREED (terminal: AGREE entry)
  │
  ├──► ACTIVE (COUNTER or QUALIFY entry)
  │
  ├──► DISPUTED (DISPUTE entry)
  │
  ├──► ESCALATED (FLAG_HUMAN entry)
  │
  └──► DECLINED (terminal: DECLINED entry)
```

Note: DraftHouse statuses are **last-write-wins** — the last entry type determines the status via `statusAfter()`. There is no explicit state machine with validated transitions. Any point can transition from any non-terminal status to any other status based on the next entry received.

### 3.3 Mapping table

| design-review status | design-review transition | Entry type to emit | DraftHouse status after | Notes |
|---------------------|-------------------------|-------------------|------------------------|-------|
| OPEN | (initial, issue raised) | RAISE | OPEN | Direct. |
| ADDRESSED | Implementor says FIXED | AGREE (from IMP) | AGREED | **Problem:** AGREED is terminal in DraftHouse. ADDRESSED is not terminal in design-review — it awaits reviewer verification. See §3.4. |
| VERIFIED | Reviewer confirms fix | — | — | **No DraftHouse equivalent.** See §3.4. |
| REJECTED | Implementor says REJECTED | COUNTER (from IMP) | ACTIVE | Works. |
| ACCEPTED | Reviewer accepts rejection | AGREE (from REV) | AGREED | Direct. Terminal in both. |
| CONTESTED | Reviewer disputes fix/rejection | DISPUTE (from REV) | DISPUTED | Direct. |
| DEFERRED | Auto-escalated or explicit | FLAG_HUMAN + note | ESCALATED | Approximate. DEFERRED is terminal; ESCALATED is not. See §3.4. |

### 3.4 Critical mismatches

**Mismatch 1: ADDRESSED ≠ AGREED**

design-review's ADDRESSED means "implementor claims fix is done" — a non-terminal intermediate state awaiting reviewer verification. DraftHouse's AGREED means "this point is resolved" — a terminal status.

If the adapter emits AGREE for FIXED, the review tracker will show it as resolved (checkmark, line-through, counted in progress bar). But the reviewer hasn't verified yet. This gives a false sense of completion.

**Options:**
- **(A) Emit QUALIFY for FIXED.** QUALIFY → ACTIVE status. Not terminal. The point stays active until the reviewer confirms (AGREE for VERIFIED) or disputes (DISPUTE for CONTESTED). Semantically: "the implementor has something to say, but it's not resolved yet."
- **(B) Add ADDRESSED as a new DraftHouse status.** `statusAfter("QUALIFY") → "ADDRESSED"` with a new non-terminal status. The review tracker shows it as "awaiting verification" rather than active.
- **(C) Keep AGREE but add VERIFIED.** Emit AGREE for FIXED (marking it resolved). If the reviewer confirms, emit a redundant AGREE (no-op). If the reviewer contests, emit DISPUTE (transitions from AGREED back to DISPUTED). This requires removing AGREED from the terminal set — it's only truly terminal when the reviewer also agrees.

**Recommendation: Option A** for the adapter's first pass. It's the simplest change — no new entry types or status values. The semantic imprecision (QUALIFY means "clarification" not "fix claim") is acceptable for v1. Option B is the correct long-term answer if the review tracker needs to distinguish "implementor addressed" from "reviewer active."

**Mismatch 2: VERIFIED has no DraftHouse equivalent**

design-review's VERIFIED means "reviewer checked the diff and confirmed the fix is correct." It's evidence-based terminal resolution with commit proof.

DraftHouse has AGREED (generic agreement) but nothing that carries the "verified against evidence" semantics.

**Recommendation:** Add VERIFIED as specified in the exploration spec §6.2:
- New string constant in `ConversationProtocol` (or just used as a domain entry type string)
- `statusAfter("VERIFIED") → "VERIFIED"` in `DebateChannelProjection`
- Add `"VERIFIED"` to `resolvedStatuses` in `ConversationRendererConfig`
- Reviewer-only constraint: only entries with role=REV should use VERIFIED

**Mismatch 3: DEFERRED has no DraftHouse equivalent**

design-review's DEFERRED is a terminal "we're not going to resolve this" state, triggered by explicit deferral or auto-escalation after 2 contested rounds.

DraftHouse's closest is ESCALATED (from FLAG_HUMAN), but ESCALATED is not terminal — it means "needs human attention."

**Recommendation:** Add DEFERRED as a new domain status:
- `statusAfter("DEFERRED") → "DEFERRED"` — new status string, not a new entry type
- Could reuse FLAG_HUMAN entry type with a DEFERRED status, or add a new DEFER entry type
- Add `"DEFERRED"` to `resolvedStatuses` — it's terminal
- The auto-escalation logic (2 contests → auto-defer) stays in the adapter/orchestrator, not in ConversationProjection

**Mismatch 4: Last-write-wins vs validated transitions**

design-review enforces a state machine with validated transitions (e.g., OPEN cannot jump to VERIFIED). DraftHouse uses last-write-wins — any entry can change any point's status.

This is a design difference, not a bug. DraftHouse's flexibility allows richer conversation patterns (multiple COUNTER/QUALIFY exchanges). The adapter simply needs to emit entries in the correct order to produce the right final status. No DraftHouse change needed.

---

## 4. Gap Analysis

### 4.1 DraftHouse needs to add

| Gap | Priority | Scope | Reasoning |
|-----|----------|-------|-----------|
| **VERIFIED status** | High | `DebateChannelProjection.statusAfter()`, `ConversationRendererConfig`, review tracker panel | Core design-review concept. Without it, "fix confirmed" and "rejection accepted" are indistinguishable. |
| **DEFERRED status** | Medium | Same scope as VERIFIED | Terminal resolution for unresolvable issues. Without it, deferred issues show as escalated (active). |
| **Commit evidence fields** | Medium | New fields on DebateStreamEntry or as metadata | `commit_before` / `commit_after` carry diff evidence. Critical for VERIFIED semantics. Could be metadata on AGREE/VERIFIED entries rather than first-class fields. |
| **Round summary metadata** | Low | New topic or MEMO subtype | Cost, duration, issue counts per round. Useful for the "chess clock" UX (§7.1). |
| **ASSUMPTION / SETTLED entry types** | Low | New entry type strings | Assumptions and settled decisions are structured metadata that currently has no representation. Could be MEMOs with structured content. |

### 4.2 design-review could improve

| Gap | Priority | Impact on adapter | Reasoning |
|-----|----------|-------------------|-----------|
| **Structured LOCATION per issue** | High | Eliminates regex extraction from prose | Currently location is buried in reviewer's body text. A `LOCATION: §4.2` field per issue heading would make the adapter trivial. |
| **Structured PRIORITY per issue** | Medium | Enables DraftHouse priority filtering | Currently all issues are equal. A `PRIORITY: HIGH` field maps directly to `PointClassification.priority`. |
| **JSONL sidecar** | Medium | Eliminates regex parsing entirely | A `events.jsonl` file with one JSON object per event (type, issue_id, round, role, content, location, priority) makes the adapter a line-by-line JSON reader. |
| **Explicit EVIDENCE markers** | Low | Maps to commit evidence fields | `EVIDENCE: commit=abc123, section=§4.2` in implementor responses makes commit tracking reliable. |
| **DEPENDS per issue** | Low | Enables dependency-aware rendering | Some issues depend on others being fixed first. Not modeled in either system. |

---

## 5. Batch-to-Streaming Strategy

### 5.1 The problem

design-review produces output in **batches**: one file per agent per round, with all issues addressed in a single response (typically 5–15 issues per round, delivered after 5–10 minutes). DraftHouse's event model is **per-entry streaming**: individual `DebateStreamEntry` objects pushed via WebSocket.

The adapter must translate batch files into individual entries. The question is **when** and **how**.

### 5.2 Options

**Option A: Post-hoc splitting (replay)**

Parse completed workspace → emit all entries in order → done.

- Timing: after all rounds complete
- Granularity: entire workspace → N entries, emitted sequentially
- Latency: zero (all data available immediately)
- UX: "load completed review" — immediate full rendering

**Option B: Per-round batch emission (live watching)**

Watch workspace for new response files → parse each file → emit entries for that round.

- Timing: as each response file appears (file watcher on `responses/`)
- Granularity: one file → M entries per round
- Latency: 5–10 minutes between batches (the review round duration)
- UX: "watching a review in progress" — periodic batch arrival

**Option C: Hybrid streaming (future)**

design-review emits structured events as it works (progress updates, partial findings) → adapter streams individual events in real time.

- Timing: continuous during agent execution
- Granularity: individual events as they happen
- Latency: seconds
- UX: "live streaming the debate" — real-time
- Requires: design-review to emit structured events natively (JSONL sidecar or equivalent)

### 5.3 Recommendation

**Start with Option A (replay), evolve to Option B.**

Option A is the first deliverable (#95 replay adapter). It validates the parsing, entry construction, and rendering pipeline with zero complexity. The adapter is a pure function: workspace directory → List<DebateStreamEntry>.

Option B is a natural extension: add a file watcher that triggers the same parse-and-emit logic when new files appear. The adapter code is reusable — only the trigger changes (manual load vs file watcher).

Option C is attractive but premature. It requires changes to design-review's output format (the "structured output" improvements in #96). Don't build streaming infrastructure until the output format supports it.

**Batch arrival UX:** When a round's entries arrive (5–15 entries at once), the debate panel should show a round divider ("Round 2: 5 new issues, 8 responses") rather than animating individual entries. The review tracker panel already handles batch updates correctly — it re-derives all point statuses on each render.

### 5.4 Entry emission order

Within a round, the adapter should emit entries in this order:

1. **Reviewer issues** (RAISE entries) — all new issues from reviewer-N.md
2. **Implementor responses** (AGREE/COUNTER/FLAG_HUMAN entries) — all responses from implementor-N.md
3. **Reviewer confirmations** (VERIFIED/DISPUTE/AGREE entries) — confirmations from reviewer-(N+1).md
4. **Assumptions** (MEMO entries) — if present
5. **Settled decisions** (MEMO entries) — if present
6. **Round summary** (MEMO entry) — counts and cost from progress.log

This order ensures the client-side fold produces the correct ConversationState: each point is created (RAISE), then responded to (AGREE/COUNTER), then confirmed (VERIFIED/DISPUTE).

---

## 6. Findings Summary

### What the exploration spec got right
- The entry type mapping table (§6.2) is substantially correct
- The adapter-as-permanent-infrastructure position is validated
- The bi-directional adapter design is sound
- The document timeline abstraction (§4.1) is the right direction for version support

### What the exploration spec got wrong or missed
- **ADDRESSED ≠ AGREED**: the spec's mapping of FIXED → AGREE conflates a non-terminal intermediate state with a terminal resolution. The implementor's "fixed" is a claim, not a confirmation.
- **No DEFERRED mapping**: the spec doesn't address the DEFERRED terminal state
- **No emission order specification**: the batch-to-streaming strategy needs explicit entry ordering
- **Assumptions and settled decisions**: these structured metadata types have no mapping
- **Progress.log metadata**: timing, cost, and health data has no mapping

### Validated design decisions
1. Parser, not ConversationProjection subclass — design-review files are batch, not incremental fold
2. DebateStreamEntry as adapter output, not ConversationState — consumers fold entries into state
3. VERIFIED as a new terminal status with evidence semantics and reviewer-only constraint
4. Separate sessions per phase, linked by document path

### New recommendations from this research
1. **Emit QUALIFY (not AGREE) for FIXED responses** — preserves the non-terminal nature of "implementor claims fix done" until reviewer verification
2. **Add DEFERRED as terminal status** — needed for auto-escalation and explicit deferral
3. **Design-review improvements (#96) should prioritize LOCATION and PRIORITY fields** — these have the highest adapter impact
4. **JSONL sidecar (#96) eliminates the regex parsing fragility** — the parser.py regex patterns are robust but any structured output would be simpler
