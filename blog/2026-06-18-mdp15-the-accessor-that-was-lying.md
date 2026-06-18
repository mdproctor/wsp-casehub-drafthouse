---
title: "The Accessor That Was Lying"
date: 2026-06-18
author: Mark Proctor
projects: [casehub-drafthouse]
tags: [encapsulation, persistence, domain-model, jpa, quarkus]
---

# The Accessor That Was Lying

`session.documentSet()` looked harmless. A public accessor returning a domain object — standard Java. Callers used it to add documents, remove documents, set comparisons. Each method on `DocumentSet` was individually `synchronized`. Thread-safe, right?

Not when the caller needs to do three things in sequence and the result depends on all three being atomic. `removeDocument()` in the MCP tool layer was reading the comparison before the remove, calling remove, reading the comparison after, then computing whether it was cleared. Four synchronized calls, four separate lock acquisitions, four opportunities for another thread to interleave. The accessor was exposing an implementation detail and pretending it was an API.

The fix was to stop pretending. `DebateSession` now owns all document operations — `addDocument()`, `removeDocument()`, `setComparison()`. `DocumentSet` is package-private. Callers never touch it. The compound `removeDocument()` runs under a single `synchronized` block on the `DocumentSet` instance, and returns a boolean telling the caller whether the comparison was cleared. One lock, one result, no interleaving.

What makes this interesting is the error signaling choice. The original code had two error paths through two mechanisms: an exception for the primary-document guard, and a return value (`false`) for not-found. Every caller had to handle errors in two different ways. The review pushed back on this — both conditions are invalid input. The method now throws `IllegalArgumentException` for both. Call sites collapsed from a try-catch plus a boolean check to just the try-catch.

## Persistence that knows its place

With the domain model cleaned up, adding persistence was straightforward. `DebateSessionSnapshot` captures all durable state — documents, comparison, participants — and `DebateSession.fromSnapshot()` reconstitutes a live session. The Store SPI operates on snapshots only; it never sees the live object.

The CDI wiring had one surprise. DraftHouse is an application repo, not a foundation library. The standard platform pattern — `@DefaultBean` in one module, `@ApplicationScoped` in another, classpath activation — doesn't work when both beans are in the same Maven module. The no-op default and the JPA implementation are both in `runtime/`. With both always on the classpath, the JPA bean always wins and the no-op is dead code.

The fix was `@IfBuildProperty` — a build-time gate that excludes the JPA bean from the CDI graph entirely when the property isn't set. Tests get the no-op automatically. Production enables persistence with one property. The platform protocol says "don't use `@IfBuildProperty` to switch backends" — but that's about consumer-side switching. Gating the provider within its own module is the standard Quarkus pattern for optional features.

## The named persistence unit trap

The JPA store shares the existing `qhorus` named datasource. Two things had to be set for Hibernate to discover the entity: `@PersistenceUnit("qhorus")` on the EntityManager injection, and `quarkus.hibernate-orm.qhorus.packages` extended to include the entity's package. Missing either one produces silent failure — queries return empty, no error, no warning. The compound nature is the trap: each setting looks correct in isolation.

The `@Transactional` gap was the review catch. `load()` and `loadAll()` had no transaction annotation. For `loadAll()` called from `@PostConstruct` — outside any request scope — that's a failure waiting for someone to flip the persistence flag in production.

## What landed

`DocumentEntry` and `ComparisonPair` are top-level domain records. `DocumentSet` is hidden. `DebateSession` encapsulates all compound operations with proper synchronization. A pluggable Store SPI with snapshot serialization makes session state durable when the datasource is configured for it. The encapsulation simplified the MCP tool layer — net reduction of 19 lines across the migration, with stronger thread safety and clearer error contracts.
