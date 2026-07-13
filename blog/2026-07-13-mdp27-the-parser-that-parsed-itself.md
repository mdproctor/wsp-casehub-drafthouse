---
layout: post
title: "The Parser That Parsed Itself"
date: 2026-07-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-drafthouse]
tags: [design-review, jsonl, parser, evidence-verification]
series: issue-96-design-review-structured
---

The design-review skill has a structural problem I've been aware of since #95 shipped the workspace replay adapter: two independent regex parsers — one Python, one Java — extract the same structure from the same markdown, and they must stay in sync. Every time the Python parser evolves, the Java parser silently diverges.

The deeper issue is that the agent output format carries no explicit structure. When a reviewer raises an issue about §4.1, that location reference is buried in prose. When an implementor claims to have fixed something, the section reference is extracted by a heuristic — `verify_against_diff()` — that searches for a section number string inside a git diff. It doesn't actually verify anything meaningful. It checks whether the text "§4.1" appears in the diff output, not whether §4.1's content changed.

I wanted to fix this at the root rather than patch the symptoms. The approach: add structured metadata lines to the agent markdown (LOCATION, PRIORITY, DEPENDS for reviewer issues; EVIDENCE for implementor fixes), then emit JSONL sidecar files as typed domain events that both sides can consume.

The JSONL schema was the interesting design decision. Claude and I went back and forth on whether the sidecar should be a serialized dump of the Python parser's internal data structures or a deliberate contract of typed domain events. The dump approach is simpler — you just serialize whatever the parser produces. But it couples the Java consumer to the Python parser's internals: when the parser refactors, the schema changes, and Java breaks.

We went with typed events: `issue_raised`, `issue_fixed`, `issue_rejected`, `confirmation`, `round_signal`. Each line is self-describing. The Python parser can change its internals freely as long as it produces the same events. The Java side reads events, not parse results. This aligns with how channels work in the platform — producers and consumers agree on event types, not on each other's internal data structures.

The other decision worth naming: `verify_against_diff()` is deleted, not updated. The new `verify_evidence()` is a fundamentally different mechanism — structured evidence claims from the implementor, mechanically verified against git. The old function searched for a string in a diff; the new one parses the spec to find section boundaries, parses the diff to find modified line ranges, and checks for overlap. No fallback to the old heuristic. Pre-release means the right answer is to clean it out at the root.

The confirmation dataclass also changed — from a `(resolved: bool, accepted: bool)` pair that allowed contradictory states to a single `verdict` discriminator: `"resolved" | "accepted" | "contested"`. Small change, but it removes a class of bugs where both booleans could be true simultaneously.

The cross-repo nature of this work surfaced a practical gap: IntelliJ MCP's structural editing tools don't support Python. Opening a Python project in IntelliJ succeeds, but `ide_edit_member` and friends return "Language not supported." Combined with hooks that block `Edit` on source files, this created a deadlock — neither tool could modify the file. Worth knowing for anyone doing cross-language platform work with this toolset.
