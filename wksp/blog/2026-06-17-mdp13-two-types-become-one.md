---
layout: post
title: "DraftHouse — Two Types Become One"
date: 2026-06-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-drafthouse]
tags: [selection, sse, refactoring, type-unification]
---

# DraftHouse — Two Types Become One

**Date:** 2026-06-17
**Type:** phase-update

---

## What I was trying to achieve: close two gaps, one test and one feature

DraftHouse's context-usage SSE path — the stream that delivers context-window usage data to the browser gauge — had no integration test. The `DebateMcpToolsTest` verified that `pushContextSnapshot()` was *called* (Mockito mock), but nothing tested the actual delivery from `ConcurrentHashMap` through the 500ms live tick to a connected SSE client. That's a meaningful gap: the composition of three `Multi<String>` streams in `DebateEventResource.events()` is non-trivial, and the initial-snapshot-first ordering is an API contract the gauge relies on.

The second gap was structural. The diff panel emits a `selection-changed` event when the user selects text — side, start line, end line — but nothing consumes it. The user selects a passage, and the debate has no idea what they're looking at. Selection-scoped conversations were in the project vision from the start; the wiring just hadn't been done.

## What I believed going in: two separate pieces of work

I expected the SSE test to be fast and the selection feature to be a clean three-layer add: domain record, REST endpoint, browser wiring. The spec review changed that. It surfaced a fourth concern I hadn't considered.

## The type that shouldn't have been two types

The review path already had a selection model — `ReviewSession` stored `selectionSide` and `selectionText` as separate fields, with a compact constructor invariant enforcing both-null-or-both-non-null. The debate path was about to get its own `currentSelection` field with a `SelectionScope` record. Two selection representations for the same concept.

The fix was obvious once surfaced: `SelectionScope` becomes the single type for both paths. `ReviewSession` collapses from nine fields to eight — two selection fields become one nullable record. The compact constructor invariant moves inside `SelectionScope` where it belongs, alongside the line-range validation the debate path needs. The review path passes `startLine=0, endLine=0` when line numbers aren't known. The debate path passes real coordinates from the browser.

The migration touched eleven files and zero test logic. Every caller that previously passed `(DocumentSide, String)` now passes `SelectionScope` — the breakage is mechanical and the compiler catches every site. `ReviewerChannelBackend.buildSelectionContext()` reads `session.selection().selectedText()` instead of `session.selectionText()`. `DraftHouseMcpTools.updateSelection()` builds a `SelectionScope` with zeroed line numbers. The test helpers change from nine-arg constructors to eight-arg. Nothing conceptually changed; the type just stopped being duplicated.

## The live tick that was about to get worse

The SSE live tick had four conditional branches — context snapshot present with entries, context only, entries only, heartbeat. Adding selection would have doubled the combinatorics to eight branches. Instead we refactored to a collect-then-emit pattern: drain all pending metadata maps into a list, append message entries, emit the list. Heartbeat only if the list is empty. Adding a third or fourth metadata type now means adding one `remove` call, not restructuring the emission logic.

## What it is now

The SSE delivery path is tested end-to-end: initial context snapshot on connect, pushed snapshots via `reportContext()`. Selection flows from browser mouseup through the diff panel's enriched `selection-changed` event (with `selectedText` and uppercase `side`), through the shell's `fetch` POST to `/api/debate/{id}/selection`, into the `DebateSession`'s volatile `SelectionScope` field, and out via SSE as a `selection-scope` metadata event. The debate summary includes an "Active Selection" section — with line numbers for the debate path, without for the review path.

The selection is deliberately sticky: it persists until the user selects different text or the session ends. The DELETE endpoint exists for programmatic clearing, but the browser doesn't call it on deselection. "The user was last looking at lines 5–12" is useful grounding context for an LLM even after the visual highlight is gone.

One thing worth noting: the `startLine=0` sentinel for "lines not known" is a convention, not a type-system guarantee. If line-aware selection ever matters on the review path, the MCP tool already accepts the coordinates — it just passes zeroes today. The type is ready; the caller isn't.
