---
layout: post
title: "Watching the watcher"
date: 2026-07-20
type: phase-update
entry_type: note
subtype: diary
projects: [DraftHouse]
tags: [file-watching, workspace-adapter, design-review]
---

DraftHouse can replay completed design-review workspaces — load the workspace,
parse all the response files, replay them as debate entries in the channel feed.
That works. But a design review takes 10–30 minutes across multiple rounds, and
until now there was no way to watch one in progress. You had to wait for it to
finish and then load the result.

The gap was obvious once I started using the replay adapter regularly. I'd kick
off a design review, switch to DraftHouse to watch it, and find... nothing. The
review was running. DraftHouse couldn't see it yet.

## The shape of the problem

The replay adapter does everything in one pass: parse the workspace, dispatch all
entries, push to WebSocket. For live watching, you need the opposite — sit on a
directory, notice when a new file appears, parse just that file, and push the delta.

Two things made this non-trivial. First, the replay adapter's `replay()` method was
monolithic — a single 200-line method that looped over rounds and dispatched six
different entry types inline. The watcher needed to call the same dispatch logic for
individual rounds as they arrived. Second, the watcher thread is a plain Java thread
with no CDI request context — and `MessageService.dispatch()` needs one.

## What we built

`WorkspaceWatcher` sits between `directory-watcher` (Greg Methvin's library — native
macOS FSEvents via JNA, no polling) and the existing dispatch infrastructure. When
`load_workspace` runs, it now checks whether the review is still in progress by
scanning the last 20 lines of `progress.log` for a terminal state. If there's no
`REVIEW DONE` / `PAUSED` / `FAILED`, it replays what exists and hands off to the
watcher.

The watcher filters events to three paths: `responses/reviewer-N.md` (parse round,
dispatch RAISE entries), `responses/implementor-N.md` (dispatch response entries),
and `progress.log` (tail new lines, push metadata). A `<workspace-status>` element
in the topbar renders the metadata — agent activity, elapsed time, cost per round.

The refactoring landed naturally. I extracted `dispatchIssues()`, `dispatchResponses()`,
`dispatchConfirmations()`, and the rest from the monolithic `replay()` into
package-visible methods. Both replay and watcher now call the same methods. A
`tenancyId` field on the adapter handles the CDI context gap — the watcher captures
it from the channel at construction time and passes it explicitly on every
`MessageDispatch`, bypassing the `CurrentPrincipal` fallback that doesn't exist on
the watcher thread.

## The dedup bug

The one genuine bug was in the catch-up reconciliation. The watcher starts with a
scan to close the TOCTOU window between replay completion and watcher registration.
It iterates expected round numbers and calls `processFile(stem)` for each. The dedup
guard — `processedFiles.add(stem)` — fires before the file existence check. When the
implementor file doesn't exist yet (the round hasn't produced it), the add succeeds
but `waitForFile()` fails. The stem is now permanently in the dedup set. When the
file actually appears, the async handler sees it's already "processed" and skips it.

The fix is one line: `processedFiles.remove(stem)` in the failure path. But the
failure mode is genuinely non-obvious — the dedup-first pattern is the standard
concurrent guard, and it works perfectly when both paths process *actual* items. It
breaks when one path processes *expected* items that might not exist yet.

## What the design review caught

The adversarial review ran 8 rounds and drove significant improvements into the
spec before I wrote a line of implementation code. The biggest catches: CDI request
context activation on the watcher thread (`Arc.container().requestContext()`),
`ReplayResult` extension with `raiseMessageIds` and `lastMessageId` for incremental
WebSocket push, and the `processedFiles` dedup keyed by filename stem rather than
round number (each round has two independent files that arrive separately).

The review also caught that the `dispatchRoundSnapshot` method needed the repo path
and spec path passed through from the replay result, and that `endDebate()` needed
to stop the watcher before removing the session from the registry. Details that would
have surfaced during implementation — but surfacing them in the spec meant the
implementation was straightforward.

The `directory-watcher` library is a good fit here. Native FSEvents on macOS, no
polling, and the API is minimal — builder pattern, `watchAsync()`, `close()`. The
file appearance rate for design reviews is so low (one file every 2–8 minutes) that
any watching mechanism would work, but FSEvents means zero CPU cost between events.
