---
layout: post
title: "The Checkpoint That Was Always There"
date: 2026-08-04
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-drafthouse]
tags: [design-review, orchestration, hil, dimensional-review]
series: issue-108-context-aware-chunking
---

*Part of a series on [#108 — Explore context-aware chunking](https://github.com/casehubio/drafthouse/issues/108). Previous: [Chunking the Implementor](2026-07-18-mdp27-chunking-the-implementor.md).*

## The problem that chunking was actually solving

I came into this session planning to design context-aware chunking — passing prior chunk summaries to later chunks so the implementor has cross-chunk context. That was the follow-up from the chunking research two weeks ago, where I'd built `--chunked` mode and found that cross-issue coupling was rare enough (5%) that isolated chunks produce equivalent or better fixes.

But the architecture has moved on since then. Design reviews now run in dimensions — coherence, structure, and robustness as three parallel processes, with a cross-cutting pass that reads all three trackers at the end. Each dimension is focused on one concern, each runs its own reviewer-implementor loop at the selected degree.

The dimensional model already does what chunking was reaching for. Each dimension is a narrower scope that produces deeper analysis. Cross-issue coupling within a dimension is lower than it was in the old monolithic pass because the reviewer is only looking at one concern. The implementor responds to fewer, more focused issues.

So why was the issue still open?

## The real problem was always UX

Chunking was never primarily about quality. The research showed quality was equivalent or better. The real motivation — the one that kept surfacing in every conversation about it — was that the human sits idle for 20 minutes watching a progress bar. Launch the review, wait, get results. No engagement until everything finishes.

The dimensional model parallelised the work but didn't change this. Three dimensions running simultaneously still means the human waits for all three to finish before seeing anything actionable. The watchdog cron monitors completion and auto-launches cross-cutting. The human's first decision point is at the end.

## Two checkpoints, not a new architecture

Once we identified the actual problem — "give the human something useful to do sooner" — the solution was obvious. The existing dimensional review flow is fine. It doesn't need restructuring. It needs two decision points inserted into the watchdog:

**Round 1 checkpoint.** After all three dimensions complete their first round, the watchdog presents the findings. The human sees what each dimension found and can kill dimensions that aren't producing useful issues. At standard degree, this happens ~4-5 minutes in instead of ~20 minutes. The dimensions keep running — they were never stopped. Killing a dimension just terminates its background process.

**Pre-cross-cutting gate.** After all surviving dimensions finish their full round loop, the human sees the final results and decides whether cross-cutting synthesis is worth the cost. Previously, cross-cutting launched automatically. Now the human can skip it when the dimensional results are already sufficient.

Four actions at the round 1 checkpoint: accept all (dimensions keep running), refuse all (kill everything), refuse a subset (kill specific dimensions), or discuss specific findings before deciding. At light degree the checkpoints collapse — one round, one checkpoint, one decision.

## The design iteration worth noting

I didn't arrive at this directly. The first design split reviews into explicit phases: phase 1 runs all dimensions at `--degree light`, then phase 2 resumes survivors at the selected degree via `--workspace`. Two new CLI flags, workspace resume logic, session continuity mechanisms.

Claude ran a light design review on that spec. Three independent reviewers — coherence, structure, robustness — converged on the same core problems: workspace resolution breaks across phases because of timestamped paths, `--start-round` duplicates existing resume logic, the session model conflates reviewer and implementor sessions. Thirty-one issues across three dimensions, and the strongest signal was convergence — the same architectural objections surfacing independently.

I addressed those findings, but then I hardcoded `--degree light` for phase 1 — giving the most important round the lowest budget. Round 1 is where issues are found. Nerfing it defeats the purpose. The fix was to use `--degree {selected} --max-rounds 1` instead, but that was still overcomplicating things. The existing system already has degrees that control rounds. Light is already 1 round.

The final design is much simpler: launch dimensions at the selected degree exactly as before. No `--max-rounds`, no `--degree light`, no workspace resume. The only change is the watchdog cron, which now monitors for round 1 completion events and presents a checkpoint before continuing to monitor for overall completion. The JSONL events I added to review.py — `dimension_start`, `round_findings`, `round_end`, `dimension_done` — give the watchdog the round-level visibility it needs.

The review.py round loop is untouched. The launch commands are untouched. The degree presets are untouched.

## What this opens up

The JSONL events are useful beyond the watchdog. DraftHouse's `<workspace-status>` panel already watches progress.log files and renders real-time status. With round-level events, it can show per-dimension progress bars, issue counts, and cost while the review is running. The human isn't just waiting for checkpoints — they're watching the review unfold.

The pre-cross-cutting gate also changes the cost model. Cross-cutting is the most expensive pass because it reads all three dimension trackers and reasons across them. Making it optional means a standard review can end after dimensions — spending on cross-cutting only when the dimensional findings suggest cross-cutting would find something the dimensions missed.
