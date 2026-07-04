# Handover — 2026-07-04

**Branch:** `main` (clean)

## Last Session

Closed #88 (WebSocket reconnection tests) and #85 (document badge dropdown) on one branch. Three integration tests now verify the reconnection catch-up contract, stale subscription handling, and concurrent push serialization. `<drafthouse-doc-picker>` replaces the dead badge with a Shadow DOM dropdown for A/B slot assignment — no server-side changes, uses existing REST endpoint. 379 tests pass. Design went through 3-round adversarial review (10 issues, 9 verified, 1 accepted).

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #53 | Brainstorming UI — richer option exploration | L | High | Unblocked by #75 |
| #72 | Pipeline orchestration — sequential multi-perspective | L | High | Server-side |
| #71 | Claude-to-Claude continuous conversation protocol | L | High | Server-side |

## Cross-Repo

Pages improvements filed: casehubio/casehub-pages#97 (epic), #98–#100 — mechanical migration when delivered, no drafthouse architectural impact.

## References

- Spec: `docs/superpowers/specs/2026-07-04-websocket-tests-and-badge-dropdown-design.md`
- Blog: `blog/2026-07-04-mdp25-closing-the-small-gaps.md`
- Garden: GE-20260704-73bebb (pages event op skips lastSeq tracking)
