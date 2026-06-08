# Handover — 2026-06-08

**Branch:** `main` (clean)

*Updated: #25, #27, #31, parent#169 closed — removed from backlog.*

## Last Session

Completed #27 (debate channel): implemented full debate MCP tool surface (`start_debate`, `raise_point`, `respond_to`, `flag_human`, `get_debate_summary`, `end_debate`) with `DebateChannelBackend`, `DebateChannelBackendFactory`, `DebateSessionRegistry` SPI + impl, and `DebateChannelProjection`/`ReviewChannelProjection` split. Extracted `DraftHouseInstances`. Synced `ARC42STORIES.MD` and design spec. Also closed #25 (integration test lifecycle) and #31 (ChannelProjection SPI migration).

## Immediate Next Step

Run `/work` for #33 — orphaned reviewer instance on `start_review` partial failure (small, low complexity, no unknowns).

## What's Left

- **#33** — `DraftHouseMcpTools`: orphaned reviewer instance on `start_review` partial failure (no `deregister` on InstanceService) · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #33 | Orphaned reviewer instance on start_review partial failure | S | Low | Quick bug fix |
| #26 | Review loop: session continuity, context management, sub-agent architecture | L | High | Design issue; needs brainstorm before any code |

## References

| Context | Where |
|---|---|
| Feature backlog | `docs/FEATURES.md` |
| MCP tools spec (v2) | `docs/superpowers/specs/2026-06-04-drafthouse-mcp-tools-design.md` |
| Debate design spec | `docs/superpowers/specs/2026-06-05-debate-channel-design.md` |
| ADRs 0002, 0003 | `wksp/adr/` |
| Blog mdp05 | `wksp/blog/2026-06-04-mdp05-mcp-surface-and-a-stale-snapshot.md` |
| GitHub | `casehubio/drafthouse` |
| Protocols PP-20260604 | `docs/protocols/mcp-tool-error-strings.md`, `mcp-tool-llm-prompt-injection.md` |
