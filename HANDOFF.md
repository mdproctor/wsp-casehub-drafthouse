# Handover — 2026-07-29

**Branch:** `main` (#100 closed)

## What Happened This Session

Implemented channel-based HIL (#100) — concurrent human participation in
adversarial debates. Full brainstorm → design review ($12.91, 16 issues) →
7-task implementation → code review → squash → push.

New `HumanActionResource` with 5 REST endpoints (comment, raise, override,
prioritise, batch). Three new entry types (COMMENT, HUMAN_OVERRIDE,
REPRIORITISE) with projection support. `DecisionFileWriter` bridges to
design-review workspace via `decisions/human-round-{n}.md`. UI panels
updated with action buttons, human badge, and batch accept bar.

Cross-repo: `ConversationFold.reprioritisePoint()` added to casehub-blocks.
Also fixed pre-existing eidos SNAPSHOT API breaks in SimplePromptRenderer,
DraftHouseMcpTools, and ReviewerDescriptorSeederTest.

578 tests green, 0 failures.

## Follow-up

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| #113 | Exclude stale ledger identity CDI beans | XS | Low | Workaround applied; root fix is in casehub-ledger |
| #93 | Document workbench (epic) | XL | High | #100 now closed; next piece TBD |
| — | design-review skill reads decisions/ | M | Med | soredium change — agents read human decisions at round start |

## References

| Context | Where |
|---------|-------|
| Blog entry | blog/2026-07-29-mdp30-the-human-in-the-channel.md |
| Design spec | docs/specs/issue-100-channel-based-hil/2026-07-29-channel-based-hil-design.md |
| Design review | ~/adr/casehub-drafthouse/channel-based-hil-20260729-182103/ |
