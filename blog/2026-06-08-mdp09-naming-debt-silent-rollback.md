---
layout: post
title: "The Naming Debt and the Silent Rollback"
date: 2026-06-08
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-drafthouse]
tags: [qhorus, debate-channel, domain-model, silent-failure]
---

The goal coming in was to add the write path: MCP tools for two LLM agents to post structured debate entries into a Qhorus channel, and a projection to fold those entries back into `ReviewState`. The previous session had already built the read side. This was the write side.

It should have been straightforward. It wasn't, for two different reasons.

---

**The class I named wrong**

The existing `DebateChannelProjection` wasn't a debate projection. It was the review Q&A projection — it dispatched on `message.type()` (QUERY, RESPONSE, DECLINE, HANDOFF) and used `actorType` to classify HUMAN as REV and AGENT as IMP. I named it when the debate infrastructure didn't exist yet, so the name was aspirational rather than accurate. By the time the actual debate channel arrived, the class was doing something completely different from what its name promised.

The problem isn't the name. The problem is what happens if you change the class to match its name. A debate channel projection dispatches on metadata in the message content — a free-form key=value sidecar. Review Q&A messages carry no such metadata. Switch the dispatch logic and every review channel message is silently discarded. The conversation memory from the previous session is immediately un-fixed.

The fix is obvious in retrospect: the existing class becomes `ReviewChannelProjection`, a new `DebateChannelProjection` gets the metadata logic. But the subtlety that makes it non-trivial is that `ReviewerChannelBackend` uses the projection to build LLM context before every response. Getting the injection wrong — pointing the reviewer at the debate projection — is a silent failure. Responses still arrive, just without any history.

---

**The wrong terminal state**

During the design, I had `dispute` mapping to `ReviewStatus.DECLINED`. That's what the review channel uses when the AI refuses to answer a question — the point is done, the conversation on that topic is over. It renders with 🚫 and strikethrough.

A disputed debate point is not done. The implementer disputes the reviewer's position, the reviewer counters, they qualify, eventually one of them agrees or a human is flagged. `DECLINED` with strikethrough while the debate is live is actively wrong. We needed `ReviewStatus.DISPUTED` — non-terminal, no strikethrough, rendered as ⚡ to signal live contestation rather than closure.

This is the kind of semantic collision that's easy to miss because the word "decline" is genuinely ambiguous. An AI declining to answer an out-of-scope question is a refusal. An agent declining to accept a position in a structured debate is an argument. The domain model needs to tell them apart even though both map to `MessageType.DECLINE` in Qhorus.

---

**The silent rollback**

The implementation went smoothly until the integration test. `start_debate`, `raise_point`, `respond_to` all dispatched correctly. `get_debate_summary` returned an empty state. No errors. No exceptions. Nothing in the logs.

`MessageDispatch.Builder` has an `artefactRefs(String)` method that accepts any string at compile time. We were passing key=value pairs — `"entryType=raise|agent=REV|round=1|priority=P1|scope=ISOLATED"` — as sidecar metadata for the debate projection to parse. This compiles. It runs. The dispatch call returns normally.

What it doesn't do is store the message. The Qhorus runtime calls `ArtefactRefParser.parse()` which splits on commas and calls `UUID::fromString` on each token. Any non-UUID token fails. The transaction rolls back. The exception is caught internally. The caller sees nothing.

The field is documented as carrying references to shared artefacts — UUIDs pointing at things in `SharedData`. Using it for free-form application metadata is semantically wrong and also fails silently in the worst possible way.

The fix was to move the metadata into the message `content` field using a `META:` header prefix. `META:entryType=raise|agent=REV|round=1|...\n\n<body>`. The projection strips and parses the header, stores the body text in the thread entry. It's not elegant but it works, and the alternative — a two-step write to `SharedData` followed by a UUID reference in `artefactRefs` — adds more complexity than it solves.

The collision risk is real: if an agent writes a debate entry that opens with the literal text `META:` the projection misparsed it silently. In practice this doesn't happen because agents generating debate text have no reason to start there, but it's an unguarded assumption documented in #39 for a future cleanup.

What `artefactRefs` is actually for — and whether there's a cleaner path for per-message application metadata — is worth understanding before building more debate infrastructure on top of this encoding.
