---
layout: post
title: "Making Brainstorming Visible"
date: 2026-07-22
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-drafthouse]
tags: [brainstorming, lit-panels, interactive-ui, casehub-pages]
series: issue-53-brainstorming-ui
---

DraftHouse already had the brainstorming backend — MCP tools, session model, terminal endpoint. What it didn't have was anything to look at. You'd run a brainstorm in the terminal and the options existed only as JSON payloads on the wire, invisible to anyone not reading WebSocket frames.

The question that shaped this work wasn't "how do we render cards" — that's mechanical. It was: what is DraftHouse actually for? I'd been thinking of it as a diff viewer with debate features bolted on. But the real answer is collaborative document authoring with Claude. Brainstorming is one workflow. Design review is another. A business plan, a compliance doc, an app specification — they're all "guided document improvement with goals." The brainstorming UI is the first visual workflow panel, not a one-off feature.

One concrete target we keep coming back to: making it easier for newcomers to build CaseHub applications. Someone opens a browser, describes what they want to build, and the visual experience guides them from idea to validated design spec without needing CLI fluency. The platform knowledge lives in the skills and docs, not the panel — the panel just makes the process accessible.

## The service extraction that should have happened earlier

The MCP tools had all the mutation logic — create session, add options, set recommendation, mark eliminated, converge. Seven methods, each doing validation, state mutation, and event push inline. Adding REST endpoints for browser actions would have doubled all of it.

We extracted `BrainstormService` as a CDI bean. Both MCP tools and REST endpoints delegate to it. The MCP tools became thin wrappers that parse JSON and catch exceptions. The service owns all state transitions with `synchronized (session)` blocks — necessary now that browser requests and MCP tool calls can race on the same session.

The state transition guards were the design review's strongest catch. `BrainstormOption` had a plain `setStatus()` setter — any status to any status, no questions asked. That was fine when only the LLM drove transitions (it self-polices). With browser buttons, an eliminated option could be recommended again with one click. Claude caught the gap in round 1 and the transition table went through three rounds of refinement — the final version has ELIMINATED and SELECTED as terminal states, self-transitions as idempotent no-ops, and single-recommendation enforcement that reverts the previous pick to EXPLORED.

## What the browser can do

Three actions per option card: Recommend, Eliminate, Select. One PATCH endpoint routes all three by the status in the request body. The endpoint broadcasts a `brainstorm-user-action` event after each mutation, and the terminal-inject bridge in index.ts picks it up and writes `[User eliminated 'Event sourcing approach' via browser]` into the terminal. Claude sees this on its next turn — belt and suspenders with the MCP tool response also reflecting the updated state.

The session picker in the topbar handles the "20 brainstorming sessions open" scenario without 20 browser windows. Click a session, the options panel reconnects to that session's WebSocket events. Same pattern as `<doc-picker>` for document comparisons.

## What came along for the ride

The blocks 0.2-SNAPSHOT updated `ThreadEntry` to a 9-parameter record (added `sender` and `createdAt`). Every test file constructing a `ThreadEntry` broke — and there were many. `ConversationFold.createPoint()`, `respondToPoint()`, and `flagHuman()` all gained the same two parameters. `ReviewChannelProjection` needed updating. None of this was related to brainstorming, but it was blocking the build.

## What's next

The panel renders what the MCP tools push. The free-text browser input — typing a nuanced response instead of clicking a button — is deferred. It's a genuinely hard coordination problem: two conversation channels (terminal and browser) that need to merge without confusing the LLM about where input came from. Worth its own design work, not a bolted-on text field.
