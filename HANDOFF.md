# Handover — 2026-07-04

**Branch:** `main` (clean)

## Last Session

Replaced all SSE endpoints and polling with a single WebSocket connection at `/api/ws` (#87). WebSocketEventBus routes events from ChannelBackend, metadata push, and file watcher to connected browser clients — zero-latency, zero polling. Panels unchanged (transport swap invisible). Deleted WatchResource, sse-bridge.ts, SSE endpoint, 500ms polling loop, four ConcurrentHashMap queues. 375 tests pass. Design went through 4-round adversarial review (21 issues, all verified). Implementation via subagent-driven development (5 tasks, per-task reviews + whole-branch Opus review). Filed pages#97 epic for event subscription infrastructure improvements.

## What's Left

- Test source still uses `new Message()` with setters in some test files — `Message` is now a record with builder pattern. Fixed in the files this branch touched, but other test files may still use the old pattern · S · Low
- Reconnection cycle tests (#88) — full disconnect/reconnect/catch-up verification not yet covered · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #88 | WebSocket reconnection cycle and concurrent push tests | S | Med | Test gap from #87 |
| #53 | Brainstorming UI — richer option exploration | L | High | Unblocked by #75 |
| #72 | Pipeline orchestration — sequential multi-perspective | L | High | Server-side |
| #71 | Claude-to-Claude continuous conversation protocol | L | High | Server-side |
| #85 | Document badge dropdown for A/B slot assignment | S | Low | Deferred UI polish |

## Cross-Repo

**Pages improvements filed:** casehubio/casehub-pages#97 (epic), #98 (listen/unlisten), #99 (event connection API), #100 (Java push types) — mechanical migration when delivered, no architectural change to drafthouse.

## References

- Spec: `docs/superpowers/specs/2026-07-03-websocket-push-design.md`
- Blog: `blog/2026-07-04-mdp24-killing-the-polling-loop.md`
- Garden: GE-20260704-73bebb (pages event op skips lastSeq tracking)
- Demo folder: `~/drafthouse-demo/` (`.mcp.json`, `CLAUDE.md`, sample files)
