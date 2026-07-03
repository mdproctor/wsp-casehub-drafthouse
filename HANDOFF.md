# Handover — 2026-07-03

**Branch:** `main` (clean)

## Last Session

Fixed qhorus API drift (#86) — 5 classes moved from `runtime` to `api` packages, 3 became records. 16 files updated across main and test source. Also fixed Safari `marked` crash (missing import) and set up `~/drafthouse-demo/` with `.mcp.json` (SSE transport) for end-to-end MCP testing. Demo works — Claude in demo folder drives the browser diff viewer live.

## Immediate Next Step

Start a new session and pick up #87 — replace SSE polling with pages WebSocket push. This is a brainstorm-first task (new server endpoint + client pipeline integration). Unpushed commit `e23cbfc` on project main needs pushing.

## What's Left

- Push `e23cbfc` to project remote · XS · Low
- Test source still uses `new Message()` with setters — `Message` is now a record with builder pattern. Tests will fail until migrated · M · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #87 | Replace SSE polling with pages WebSocket push | L | Med | Pages infra exists; needs server WS endpoint |
| #53 | Brainstorming UI — richer option exploration | L | High | Unblocked by #75 |
| #72 | Pipeline orchestration — sequential multi-perspective | L | High | Server-side |
| #71 | Claude-to-Claude continuous conversation protocol | L | High | Server-side |
| #85 | Document badge dropdown for A/B slot assignment | S | Low | Deferred UI polish |

## References

- Blog: `blog/2026-07-03-mdp23-making-drafthouse-work.md`
- Demo folder: `~/drafthouse-demo/` (`.mcp.json`, `CLAUDE.md`, sample files)
- Garden: GE-20260703-e32c1d (Claude Code MCP type:url), GE-20260703-adad41 (quarkus dual transport)
