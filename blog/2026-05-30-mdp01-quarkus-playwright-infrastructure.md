---
layout: post
title: "The test suite that should have been there from the start"
date: 2026-05-30
type: phase-update
entry_type: note
subtype: diary
projects: [drafthouse]
tags: [playwright, quarkus, testing]
---

The Electron Playwright suite from md-compare got deleted when we migrated to DraftHouse. The intention was always to replace it — "Quarkus Playwright, properly" — but it got filed under Phase 2 and sat there for two sessions. This branch was the reckoning.

The choice was straightforward: Java Playwright via the quarkus-playwright quarkiverse extension, tests sitting alongside the existing REST-Assured suite in `server/src/test/`, one `mvn test` to run everything. The alternative was Node.js Playwright pointing at localhost:9002, which would have meant maintaining two toolchains and resurrecting the `JavaServer` process manager in a new context. Not worth it.

What I didn't anticipate was how much the implementation would teach us about the application's own behaviour.

## The SSE problem

Claude ran DiffRenderingE2ETest and came back with tests 1 and 2 passing, test 3 timing out after 30 seconds with a "element not found" error. That looks like a test ordering problem — a state leak, probably. But tests passed individually every time. The issue only appeared at position 3 in the sequence.

The actual cause: `UiResource` opens a Server-Sent Events connection to `/api/watch` on every page load. Each `context.newPage()` call that isn't followed by `page.close()` leaves one of those connections open. By test 3, the Quarkus server's HTTP connection pool was full. `page.navigate()` blocked waiting for a slot; Playwright's timeout fired first.

The fix is `@AfterEach void closePage() { if (page != null) page.close(); }` — one method, every test class. The root cause is now a garden entry and two local protocols. The symptom was completely misleading.

## LCS and adjacent diffs

The diff summary test (`~N −N +N`) was supposed to be straightforward: load a file pair with modified, deleted, and inserted blocks; assert that all three symbols appear. It wasn't. The first two runs showed `~4` with no `−` or `+` at all.

The reason: DraftHouse's LCS diff algorithm merges adjacent del+ins pairs into a single mod chunk. The `## Summary` section (A-only) and `## New Section` section (B-only) were surrounded by shared content and always adjacent — they always merged. To get standalone del and ins chunks, the fixtures needed isolated content: a `## Preface` section at the very beginning of diff-a.md (before any shared content), and a `## Appendix B` at the very end of diff-b.md. Position matters; adjacency is the trap.

## What we have

37 tests passing: 6 REST-Assured, 31 Playwright E2E across 8 classes. The diff legend is implemented (closes #17 — it needed a test to make it happen). The scroll sync tests confirm `getScrollAnchors()` builds the right number of interior anchors when headings match, and falls back to endpoint-only interpolation when they don't.

`PlaywrightFixtures.waitForRender()` waits for `[data-diff-chunk]` — the attribute set by `annotateRendered()` after both markdown rendering and LCS diff annotation complete. Any earlier signal races with the annotation step. That's now a protocol.

CI still needs the Playwright browser install step — tracked in #19.
