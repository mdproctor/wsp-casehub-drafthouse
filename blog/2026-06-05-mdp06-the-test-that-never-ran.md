---
layout: post
title: "The test that never ran, and the delivery we thought was synchronous"
date: 2026-06-05
type: phase-update
entry_type: note
subtype: diary
projects: [drafthouse]
tags: [qhorus, testing, virtual-threads]
---

## The test that never ran

The branch started with an existing test class: `ReviewSessionLifecycleIT.java`. It had five tests, complete assertions, a real `@QuarkusTest` setup. I'd been treating it as a baseline to build on.

It had never run.

`*IT.java` is the suffix maven-failsafe uses for integration tests run against a packaged artifact. Maven Surefire — the thing that runs `mvn test` — ignores them. No failure, no warning, just zero tests reported. The class had been sitting there since the previous branch, accumulating test logic against an assumption that was never verified.

This is GE-20260512-493c90 in the garden. Worth reading before you name your next test class anything ending in IT.

## I was wrong about virtual threads

The bigger finding wasn't the naming. It was the delivery model.

In designing the spec, I traced `MessageService.dispatch()` through bytecode and concluded that `channelGateway.fanOut()` was called synchronously. The reasoning was sound — dispatch does call fanOut in the same thread — but I stopped one level too early. Inside `fanOut()`, each backend receives its message via `Thread.ofVirtual().start()`. The fan-out to backends is asynchronous. By the time `dispatch()` returns to the test, the backend may not have fired yet.

The original test's busy-wait loop — which I'd dismissed as wrong-headed — was actually correct in principle. It was just using `Thread.sleep` instead of Awaitility.

We found this by reading the actual `ChannelGateway` source, not the bytecode. `fanOut()` at line 168:

```java
Thread.ofVirtual().start(() -> {
    try {
        backend.post(ref, message);
    } catch (Exception ex) {
        LOG.errorf(ex, ...);
    }
});
```

The spec got rewritten. The `@TestTransaction` I'd specified turned out to make things worse — a channel created in one transaction isn't visible to the virtual thread's response dispatch, which runs in its own transaction context. UUID isolation is the right model here, not rollback.

Two garden entries came out of this: GE-20260605-494ed0 for the virtual thread delivery pattern, and GE-20260605-73c9d6 for `CommitmentState.DECLINED` (not `CANCELLED`, which is the natural word — Qhorus mirrors the message type name).

## What the spec review caught

The spec went through two external review rounds, each catching genuine bugs before any code was written.

Round one caught `CommitmentState.CANCELLED` (doesn't exist), the wrong method for discovering DECLINE messages (`findResponseByCorrelationId` filters on RESPONSE type, silently returns empty for DECLINE), and a fragile regex for extracting the sessionId from `startReview()`'s JSON response.

Round two caught `@DefaultBean` on `ReviewSessionRegistryImpl` — a library pattern that signals "override me," not an application pattern. The Javadoc on `ReviewSessionRegistry` still pointed to `ReviewerChannelBackendFactory` as the implementation after the refactor. And the cleanup claim that `@TestTransaction` would roll back "all JPA state" was incomplete — `LedgerWriteService.record()` runs in `REQUIRES_NEW` and commits independently.

The `ReviewSessionRegistryImpl` extraction wasn't in the original spec. It emerged from the review as the cleaner design: the factory shouldn't own both session storage and backend registration. Separating them gave the registry contract a proper home, made the factory's single responsibility clear (observe `ChannelInitialisedEvent`, wire a backend), and made the registry tests cheap plain JUnit without Quarkus startup.

## What closed

Chapter 4 is complete. Epic #20 — Phase 2 critique backend — closes with this branch. The Qhorus commitment lifecycle is proven end-to-end: QUERY dispatched, virtual-thread backend picks it up, reviews the document, dispatches RESPONSE or DECLINE, commitment state transitions accordingly. The sanitization test also passes — a raw exception message containing a secret key doesn't reach the channel.

`DebateRoundTripIT` got renamed in the same pass. Same naming bug, same consequence. That one's tracked as a separate cleanup issue (removing the unnecessary `@QuarkusTest` from what's really a plain domain logic test), but the naming fix landed here.
