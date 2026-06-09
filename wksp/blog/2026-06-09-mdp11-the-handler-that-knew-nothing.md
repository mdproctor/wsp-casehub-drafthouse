---
layout: post
title: "The handler that knew nothing"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-drafthouse]
tags: [architecture, cdi, quarkus, patterns]
---

The sub-agent architecture for DraftHouse debate channels is done. The idea was simple enough: main agents accumulate judgment across rounds; sub-agents get a deliberately minimal context window and return focused findings. Separate accumulated expertise from fresh perspective. Keep both.

The interesting question was what shape the code should take. I had a monolithic orchestrator in the first spec — one class, a switch statement, six assembly methods. That's not wrong, it just means the pattern is anonymous. When devtown confirmed they'd reach for the same thing for their PR review agents, the switch became a liability. We redesigned toward `ChannelAgentHandler` — a six-line interface — with an abstract base class carrying `handles()` and `buildResponse()`, and six concrete handler beans implementing only `taskType()` and `prepareTask()`. Each is independently testable via reflection injection without touching a CDI container. The extraction to a shared patterns repo when devtown is ready becomes straightforward.

The encoding mismatch was the most interesting bug, caught in code review rather than runtime. The existing debate entry types were encoded as lowercase strings in `DebateMcpTools` — `entryType=raise`, `entryType=flag-human` — and the new sub-agent types came in as uppercase enum names. The projection dispatches on `EntryType.valueOf()` now, but before the fix it was a string switch, and mixing `raise` with `MEMO` would have discarded half the messages silently. No error, no warning. I standardised everything to `EntryType.name()` throughout — the wire format and the Java enum are now the same thing — and updated the projection to use the type-safe switch that the compiler enforces exhaustively.

Claude ran into a handful of API surprises during implementation. `@ObservesAsync` is a parameter annotation, not a method annotation — the plan had it on the method, which compiles silently but never fires. `ChatLanguageModel` is now `ChatModel` in the LangChain4j version we're on. `jakarta.enterprise.inject.DefaultBean` doesn't exist; it's `io.quarkus.arc.DefaultBean`. Claude caught and corrected each of these during implementation; I mention them because they'll save time the next time the same stack is involved.

One non-obvious thing in the DebateProtocol sentinel: `META_SENTINEL = "DHMETA:"` contains a SOH byte (U+0001) that the Claude Code Read tool swallows entirely. It renders as `"DHMETA:"` — seven characters — when the actual constant is eight. I found this when debugging a sentinel mismatch; the only way to see it is `python3 -c "with open(f,'rb') as f: print(repr(f.read()))"`. Worth knowing before spending an hour on a string comparison that looks correct.
