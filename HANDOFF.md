# Handover — 2026-06-04

**Branch:** `main` (clean)

## Last Session

Closed branch `issue-5-quality-cleanup`: four issues done (#5, #19, #32, #24). Delivered `DraftHouseMcpTools` (`start_review`, `update_selection`, `query_review`, `end_review`) replacing the deprecated REST scaffold. Fixed stale-snapshot bug in `ReviewerChannelBackend` (was holding snapshot, now reads live from registry on each `post()`). Simplified `ReviewSession` — docs on record not DataService. Also fixed work-start/work-end skills to support batched `covers:` field for multi-issue branches, and synced that change to cc-praxis + installed locally.

## Immediate Next Step

Run `/work` for issue #25 — integration test: full QUERY→Commitment→RESPONSE lifecycle with H2 Qhorus datasource (`@QuarkusTest`, MODE=PostgreSQL).

## What's Left

- **#25** — Integration test: QUERY→Commitment→RESPONSE lifecycle · M · Med
- **#33** — `DraftHouseMcpTools`: orphaned reviewer instance on `start_review` partial failure (no `deregister` on InstanceService) · S · Low
- **parent#169** — PLATFORM.md + APPLICATIONS.md need DraftHouseMcpTools capability row (filed on casehubio/parent) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #25 | Integration test: QUERY→Commitment→RESPONSE | M | Med | Prereq for #24 to be truly production-ready |
| #27 | Debate channel: map DebateChannel to Qhorus type | M | Med | Blocked by #24 completing — now unblocked |
| #26 | Review loop: session continuity, sub-agent architecture | L | High | Design issue; needs brainstorm before any code |
| #31 | Migrate DebateChannel parser to ChannelProjection SPI | — | — | Hard-blocked: casehubio/qhorus#230 must ship first |

## References

| Context | Where |
|---|---|
| Feature backlog | `docs/FEATURES.md` |
| MCP tools spec (v2) | `docs/superpowers/specs/2026-06-04-drafthouse-mcp-tools-design.md` |
| ADRs 0002, 0003 | `wksp/adr/` |
| Blog mdp05 | `wksp/blog/2026-06-04-mdp05-mcp-surface-and-a-stale-snapshot.md` |
| GitHub | `casehubio/drafthouse` |
| Protocols PP-20260604 | `docs/protocols/mcp-tool-error-strings.md`, `mcp-tool-llm-prompt-injection.md` |
