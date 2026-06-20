---
layout: post
title: "DraftHouse — Four Layers of CI Rot"
date: 2026-06-20
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-drafthouse]
tags: [ci, github-actions, flyway, debugging]
---

# DraftHouse — Four Layers of CI Rot

**Date:** 2026-06-20
**Type:** phase-update

---

## What I was trying to achieve: composite primary keys on two collection tables

Small chore — #69. The `debate_session_document` and `debate_session_participant` tables had no primary key in the DDL. Hibernate managed them correctly, but defensive DDL should have composite PKs to prevent data corruption from direct SQL. A Flyway V101 migration, five lines of SQL.

## The CI that was already broken

I went to push and discovered casehubio/drafthouse was red on main. Not from my change — from the session before. The failure looked like a dependency resolution problem: `Could not find artifact io.casehub:casehub-qhorus-api:jar:0.2-SNAPSHOT`. Locally everything built fine because `~/.m2/repository` had the artifact. CI had a clean cache and nowhere to look.

## Four layers deep

The first fix was obvious: `server/pom.xml` had no `<repositories>` section pointing to GitHub Packages. Qhorus publishes there; drafthouse just never declared it could pull from there. Added the same `casehubio/*` wildcard repository that qhorus uses.

That got CI to the next failure: 401 Unauthorized. Maven was hitting GitHub Packages but had no credentials. The `setup-java` action writes a Maven `settings.xml` with `server-password: GITHUB_TOKEN` — but that's a reference to an environment variable name, not the token itself. The Playwright browser install step ran Maven without `env: GITHUB_TOKEN`, so Maven got no credentials. Every step that invokes Maven needs the env var, not just the main build step.

Third layer: even with auth fixed, the Playwright install step failed because it tried to resolve `casehub-drafthouse-api:0.2-SNAPSHOT` — the sibling module that hadn't been built yet. The workflow ran Playwright install before `mvn install -DskipTests`. Restructured the pipeline: build first, Playwright install second, tests third.

Fourth: the pre-existing `DebatePanelE2ETest.restartContext_rendersCenteredBranchMarker` test was timing out — on CI and locally. This had been failing before my change. The RESTART_CONTEXT message dispatches without an `agent` meta field (it's a system marker). `DebateStreamEntry.from()` required agent on all entry types and returned null when it was missing. The entry never reached the UI; the Playwright test waited 30 seconds for an element that would never appear. The unit test codified the bug: `assertThat(DebateStreamEntry.from(msg)).isNull()`.

## What it is now

All four fixes landed in PR #70. CI green. The composite PK migration was the smallest change in the branch. The `setup-java` env var indirection went into the garden — it's the kind of thing you hit once per org and never again, unless you don't write it down.
