---
layout: post
title: "Chunking the Implementor"
date: 2026-07-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-drafthouse]
tags: [design-review, orchestration, chunking]
series: issue-97-chunked-orchestration-research
---

Design-review runs in rounds. The reviewer raises issues — sometimes nine or more — and the implementor addresses all of them in one batch response. Each turn takes 5-10 minutes. The human sees nothing until the full response arrives.

The alternative: chunk by priority. The reviewer still raises everything at once. But the implementor runs once per priority tier — HIGH first, then MEDIUM, then LOW. Between chunks, the human decides whether to continue.

I built `--chunked` behind a flag in review.py and ran both modes on the same spec to compare.

## What the reviewers found

Both modes reviewed the same 137-line research spec. The batch reviewer raised 13 issues. The chunked reviewer raised 12. Six core problems appeared in both: the spec treating existing code as future work, acceptance criteria contradicting the parent issue, missing safety mechanisms, zero test coverage, deferred questions that were actually design decisions, and Phase 1a scoping issues.

The interesting divergence was in what each mode found uniquely. The batch reviewer caught more methodological weaknesses — the go/no-go gate relying on a single sample, no definition of "quality" for the comparison, and the research structure conflating design choices with measurements. The chunked reviewer caught more implementation-specific bugs — resume/rebuild ignoring chunk files, evidence verification being skipped entirely, and the DraftHouse parser not handling chunk output.

Neither missed something critical the other caught. They emphasised different angles of the same problems.

## How the implementors responded

The batch implementor addressed all 13 issues in one pass. Each fix was competent — it verified claims against the code, updated the spec, and committed. Standard quality.

The chunked implementor did something I didn't expect. The HIGH chunk was given 4 focus items but addressed all 12 — it read the reviewer's full file and responded to everything. The prompt's focus constraint was ignored. This is a prompt design flaw worth fixing, but it reveals something useful: the implementor naturally wants to address what it sees, regardless of scoping.

The MEDIUM chunk then re-addressed R1-05 through R1-09 with more precision than either the batch implementor or the HIGH chunk. For example, on timeout-retry (R1-05): the batch implementor wrote "the existing chunked implementation is missing timeout-retry." The MEDIUM chunk distinguished two failure modes — timeout with partial output (where retry is correct) versus human abort (where the current `break` is correct). It traced both code paths with line references. The narrower scope gave it room to go deeper.

The LOW chunk produced straightforward responses — adequate but unremarkable. Two rejections with clear reasoning, one fix adding a test case.

## The cost

Batch: $4.99 (one reviewer + one implementor). Chunked: $7.96 (one reviewer + three implementor chunks at $1.87, $2.30, $1.83). Chunked costs 60% more per round.

The early-termination hypothesis — that skipping LOW items saves money — would have saved $1.83 here, bringing chunked to $6.13. Still 23% more than batch. The cost advantage only appears if HIGH fixes resolve enough issues to make MEDIUM/LOW unnecessary, which didn't happen in this test.

## What this means

I expected chunked to produce equivalent results — the retrospective analysis showed 90% of fixes are self-contained. Instead, chunked produced *deeper* fixes on the middle tier because the implementor had fewer items competing for attention. The trade-off is cost: 60% more per round, partially offset by early termination.

The reviewer finds similar issues regardless of mode — it sees the same spec. The implementor's fix quality is at least as good chunked, and arguably better for the middle tier where the narrower scope lets it verify claims more thoroughly. The gap is on the HIGH chunk, where the implementor ignored the focus constraint and addressed everything — diluting the depth advantage. That's a prompt fix, not an architectural problem.

One real bug surfaced: chunk response files overwrite each other because the prompt tells the implementor to write to the standard filename. The chunked code expects `implementor-1-chunk-0.md` but the agent writes `implementor-1.md`. Each chunk overwrites the previous. The responses were processed before being overwritten, so the tracker got the right data, but resume and evidence verification are broken. This needs fixing before pilot use.
