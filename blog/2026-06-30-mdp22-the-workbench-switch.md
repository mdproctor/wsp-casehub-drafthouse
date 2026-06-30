---
layout: post
title: "The Workbench Switch"
date: 2026-06-30
type: phase-update
entry_type: note
subtype: diary
projects: [DraftHouse]
series: issue-75-adopt-casehub-pages-quinoa
---

*Part of a series on [#75 — Adopt casehub-pages for UI composition via Quinoa](https://github.com/casehubio/drafthouse/issues/75). Previous: [The Half-Extracted Fold](2026-06-29-mdp21-the-half-extracted-fold.md).*

DraftHouse's index.html was 430 lines of inline JavaScript — session discovery, panel orchestration, draggable dividers, keyboard shortcuts, document badge dropdowns, all tangled together in one file. It worked, but adding anything meant understanding everything. I wanted to swap it out for casehub-pages so the same component model runs across DraftHouse, Claudony, and whatever comes next.

The bet was that the existing panels — diff viewer, debate feed, review tracker, context gauge — wouldn't need to change. They already followed the `configure(props)` + `connectedCallback()` protocol that `hostPanel` expects. That turned out to be exactly right. Four `registerPanel()` calls, a `loadSite()` layout using `split()` and `hostPanel()`, and the shell was replaced. The panels didn't know the difference.

The more interesting part was the event bus. DraftHouse had a `DebateEventBus` singleton — one SSE connection, multiple subscribers, each panel reaching into a shared object. The pages-event pattern is simpler: an SSE bridge dispatches `CustomEvent`s on `document`, and panels subscribe with `addEventListener`. No singleton, no subscriber set, no coupling between panels. Each panel declares what topics it cares about and ignores everything else.

What Claude caught during the final review was worth the cost of running it. The session discovery endpoint returns `debateSessionId`, not `id` — a property name mismatch that would have silently broken auto-connection. The sync button was missing its initial `class="active"` even though the diff panel defaults sync to true. The marked.js fallback had been quietly changed from `highlightAuto` to `plaintext`, which would have killed syntax highlighting on untagged code blocks. None of these showed up in the E2E tests because the tests don't exercise session discovery or check CSS classes — they verify panel rendering and cross-panel event flow.

The one thing I expected to be hard was Quinoa itself. It wasn't. Add the extension, create `src/main/webui/` with a package.json and an esbuild config, delete `UiResource.java`, and Quarkus serves the built output automatically. The only surprise was that quarkus-quinoa 2.5.2 — the version I'd assumed from the convention doc — doesn't exist. The actual latest was 2.8.3, and it wants an explicit `node-version` property that isn't documented anywhere.

The workbench is now 310 lines of TypeScript instead of 430 lines of inline JavaScript. But that's not the point. The point is that `split()`, `hostPanel()`, and `pages-event` are shared primitives — the same layout and communication model Claudony will use when it migrates. One component model for the platform, not one per app.
