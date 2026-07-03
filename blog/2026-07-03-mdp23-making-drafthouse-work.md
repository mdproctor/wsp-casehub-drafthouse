---
layout: post
title: "Making DraftHouse Actually Work"
date: 2026-07-03
type: phase-update
entry_type: note
subtype: diary
projects: [DraftHouse]
tags: [mcp, qhorus, safari, marked, websocket]
---

The workbench shipped last session — casehub-pages, Quinoa, the whole lot. Clean swap, 366 tests green. I opened the browser expecting to see something. Empty panels.

DraftHouse is MCP-driven. The UI is a viewer, not an app. Without an LLM client calling `start_debate` and `set_comparison`, the browser just sits there waiting. So the first task this session was making it possible for someone to actually use the thing: start the server, point Claude at it, compare two documents.

I created a demo folder at `~/drafthouse-demo/` — two sample markdown files (the "What Is a Rule Engine?" article in formal and casual versions), a `.mcp.json` for the MCP connection, and a `CLAUDE.md` explaining the workflow. The idea is simple: `cd ~/drafthouse-demo && claude`, and Claude has access to all the debate tools.

Two things went wrong immediately.

The diff panel crashed in Safari — `undefined is not an object (evaluating 'e.replace')`. The error pointed at marked.js, which was misleading. The actual problem was that `drafthouse-diff.js` used `marked.parse()` as a bare global without importing it. It worked in Chrome because the symbol leaked from the pages-runtime bundle into global scope. Safari didn't leak it. One import statement and an explicit `marked` dependency in `package.json` fixed it.

The MCP connection was more interesting. I configured `.mcp.json` with `"type": "url"` — the MCP spec's transport type for Streamable HTTP. Claude Code rejected it: "unknown MCP server type." It only recognises `stdio` and `sse`. But `quarkus-mcp-server-http` silently serves both transports — Streamable HTTP at `/mcp` and SSE at `/mcp/sse`. Neither the extension name nor the docs mention the SSE fallback. Switching to `"type": "sse"` with `/mcp/sse` got the connection working.

Then the tools failed. `start_debate` threw `NoClassDefFoundError: ChannelCreateRequest`. `list_reviewers` hit a method signature mismatch on `InstanceService.register()`. Qhorus had evolved — five classes moved from `runtime` packages to `api` packages and three of them became Java records. The uber-jar was built against the old API.

Once qhorus was rebuilt (it had its own compilation issues that blocked us for a day), the fix in drafthouse was mechanical: update imports for the five moved classes, then convert all field accesses to record accessors — `.id` to `.id()`, `.name` to `.name()`, `.content` to `.content()` across 16 files. We got it compiling clean and committed as #86.

The demo worked end-to-end after that. Claude in the demo folder called `start_debate`, added both documents, set the comparison — and the browser diff viewer updated live showing the two article versions side by side with the debate feed populating on the right.

One UX problem is immediately obvious: there's a ~5 second lag between when the MCP creates a session and when the browser notices. Session discovery polls `/api/debate/sessions` on a timer. Pages already has WebSocket infrastructure with subscribe/unsubscribe, reconnection, and `pages-event` dispatch. Replacing the SSE bridge and polling with a persistent WebSocket connection through the pages data pipeline would make it instant. That's #87.
