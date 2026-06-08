---
layout: post
title: "Cleanup and a Sentinel That Kept Escaping"
date: 2026-06-08
type: phase-update
entry_type: note
subtype: diary
projects: [drafthouse]
tags: [qhorus, sentinel, tdd, refactoring]
---

Three issues that had been deferred from the debate channel session: orphaned Qhorus instances (#33), the `[QUALIFY]` prefix hack (#38), and the `META:` sentinel collision (#39). On paper, small and sequential. In practice, one of them turned into a two-session argument with an invisible byte.

**Cleaning up what was skipped**

The instance deregistration fix (#33) was straightforward once I looked at what qhorus already had. `InstanceStore.delete(UUID)` was there in both the JPA and in-memory implementations — `InstanceService` just didn't expose it. We added `deregister(String instanceId)` with `@Transactional` (the find-then-delete pair has a TOCTOU window without it), and then fixed the four call sites in drafthouse: hoisting `String instanceId = null` before each try block so the catch can actually reach it.

The tricky part is that `instanceId` depends on `channel.id`, which you only get after `channelService.create()` inside the try. The original spec said to track a boolean flag — Claude caught that this didn't solve the scope problem. The fix is to declare `instanceId = null` before the try and assign it after the channel is created. In the catch, `if (instanceId != null)` is safe and idempotent, because `deregister()` is a no-op if the instance never registered.

`endReview` and `endDebate` had the same gap — they cleaned up the registry but left the instance alive until Qhorus's staleness sweep.

**Retiring the prefix hack**

The `[QUALIFY]` prefix on `DocumentReviewer` responses was a problem I'd been mildly annoyed about since it was introduced. The LLM was expected to include `[QUALIFY] ` at the start of qualifying responses, and `ReviewChannelProjection` would strip it and use its presence to set `EntryType.QUALIFY` vs `EntryType.AGREE`. Fragile, invisible to the type system, allows an impossible state.

The enum turned out to be the right answer: `ReviewResult.Outcome { AGREE, QUALIFY, DECLINE }`. Three states, zero impossible combinations (the old two-boolean design allowed `declined=true, qualify=true`).

The cleaner part is the dispatch: instead of encoding the qualify/agree distinction in message content, we let `MessageType` carry it. `AGREE` → `MessageType.DONE`; `QUALIFY` → `MessageType.RESPONSE`. `ReviewChannelProjection.apply()` switches on `message.type()` — no content encoding, no prefix stripping, no risk of the LLM forgetting the prefix. The Qhorus speech-act semantics are correct here: DONE means "this obligation is resolved", RESPONSE means "dialogue continues".

**The sentinel that kept escaping**

Issue #39 was straightforward to design: change `DebateChannelProjection.parseMeta()` from looking for `"META:"` to a constant containing SOH (U+0001) + "DHMETA:". SOH is a control character that no LLM will ever produce in natural text output.

What I did not anticipate was that writing `"DHMETA:"` in any tool parameter puts the actual SOH byte into the file — invisible in editors, invisible in `git diff`, invisible to `grep`. Claude's code review of the spec found the constant showed 7 visible characters where there should be 8. The obvious fixes — Edit tool, `replace_text_in_file`, regex — all failed with "not found". The invisible byte isn't matchable as text.

The fix was Python binary: `raw.replace(b'\x01', b'\\u0001')`. The thing worth knowing: to write the visible Java escape `` (six ASCII characters) via a JSON-parameter tool, write `\\u0001` in the parameter. The JSON parser converts `\\u` to `\u`, and `0001` follows as literal text. That got the entry into the garden.

The spec originally missed two other sentinel-related bugs: the hardcoded offset `5` in `parseMeta()` (which was `"META:".length()` and would silently corrupt headers after the change), and `bodyContent()` using the same `startsWith("META:")` check. Both caught in code review before implementation. Changing a sentinel is a two-method operation in `DebateChannelProjection`, not one.

Two protocols came out of this: one for the sentinel encoding rule (callers must use `DebateProtocol.META_SENTINEL`, never a hardcoded string), one for MCP session lifecycle cleanup (start/end methods must deregister every instance they registered, in both the catch path and normal termination). The SOH byte problem went into the garden — it'll save time next time someone needs a non-printable character in a constant.
