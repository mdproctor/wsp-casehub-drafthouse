---
layout: post
title: "The Tests That Pointed at the Wrong Field"
date: 2026-07-24
type: phase-update
entry_type: note
subtype: diary
projects: [DraftHouse]
tags: [testing, cdi, playwright, blocks-api]
---

Three handler tests had been failing since the blocks API refactoring landed. The error messages were clear enough — `systemPrompt()` didn't contain the expected text — but the real question was whether the tests were wrong or the handlers were wrong.

I traced each handler's `prepareTask` implementation. The design is clean: `systemPrompt` carries the static role instruction ("You are a spec analyst..."), `assembledInput` carries the dynamic data (agreed points, location hints, spec content). The handlers put content in the right place. The tests were asserting against `systemPrompt()` for data that moved to `assembledInput()` during the `buildSystemPrompt`-to-`prepareTask` refactoring.

The third failure was different. `VerifyHandlerTest.does_not_include_thread_history_beyond_raise` bypassed the test's own `requestFor()` helper — constructing a `ChannelAgentRequest` manually with `correlationId` set but no meta content. The handler extracts `pointId` from the META_SENTINEL encoding, not from `correlationId`. The helper method had already been updated; this test just wasn't using it.

The more interesting detour was the CDI blocker. Running any `@QuarkusTest` failed with an `UnsatisfiedResolutionException` for `ReactiveAgentIdentityVerificationService` — a type DraftHouse doesn't use at all. The dependency chain runs `qhorus → ledger → platform-identity`, and the platform-identity SNAPSHOT had removed the class while the ledger SNAPSHOT still injected it. No compilation error. The CDI container validated the injection point at augmentation time and failed.

The fix was `quarkus.arc.exclude-types=io.casehub.ledger.runtime.service.identity.**` — exclude the consumer rather than provide the missing target. That unblocked the test infrastructure for the brainstorm E2E tests.

The E2E tests themselves were straightforward once the CDI issue was resolved. Start a `BrainstormService` session server-side, navigate to `?mode=brainstorm`, present options — the WebSocket auto-connection wiring handles the rest. Six tests covering card rendering, eliminate/recommend/select actions, convergence banner, and summary counter updates. The shadow DOM piercing in Playwright works cleanly with the LitElement panels.

555 tests green, 0 failures. The three handler tests that were failing are now fixed, and the brainstorm panel has browser-level verification for the first time.
