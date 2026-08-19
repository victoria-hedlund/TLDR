---
name: tldr
description: Condense what just happened in this conversation into a short scaffolded summary — bottom line, what was done, what was found, what's next. Use this whenever the user types /tldr, or asks for a recap, gist, digest, summary, or "what did you just do / find", "catch me up", "I skimmed that", "summarise the session", "where are we at". Also use it proactively when a long investigation or multi-step task wraps up and the user would otherwise have to reread a wall of output. This summarises the conversation itself — it does not re-run the work.
---

# TLDR

Someone asks for a TLDR when the information they need already exists but is spread across too much output to reconstruct. They stepped away, they skimmed, they're picking up a session cold, or they're about to hand it to someone else. The summary succeeds when they can act — approve, redirect, take over — without scrolling back.

That framing drives every choice below.

## Summarise; don't re-investigate

The most common way this goes wrong is treating "/tldr" as a fresh assignment: re-reading files, re-running tests, re-searching the codebase to build a thorough report. That's the opposite of what was asked. The work already happened — the answers are in the conversation. A TLDR should land in seconds, not minutes.

Read back over the transcript and write. That's the whole job.

Two narrow exceptions, both cheap:
- A single fast command to pin down a fact you'd otherwise have to hedge on — `git diff --stat`, `git log --oneline -5` — when the exact number genuinely matters to the reader.
- Re-reading a file you already touched, if you need an exact line reference and don't have it.

If you find yourself on the third tool call, stop and write the summary. An honest "3 tests failing, I didn't get to the cause" beats an accurate report that arrived after the user gave up waiting.

## Pick the scope

Default to **the most recent substantial piece of work** — usually the last response, or the last few if they were one continuous task (investigate → fix → test is one thing, not three).

Adjust when the default would be unhelpful:
- If the last turn was trivial (a one-line answer, a file listing, a yes/no), widen until you reach something worth summarising.
- If the user names a scope, honour it: `/tldr session` and "catch me up on the whole thing" mean the full conversation; `/tldr the auth bug` means that thread only, wherever it appeared.
- If you already produced a TLDR earlier in the session and the user asks again, cover what's happened since — they've read the earlier one.

When the session is long and you're summarising all of it, organise by workstream rather than replaying turn order. Chronology is how it happened; workstreams are how someone thinks about it.

## The scaffold

```
**TL;DR** — one or two sentences: the bottom line.

**Did**
- Actions taken, each tied to something concrete.

**Found**
- Findings and evidence, with `file:line` where it applies.

**Next**
- Open items, blockers, and the suggested next move.
```

The point of a fixed shape is that a returning reader knows where to look — bottom line at the top, next action at the bottom, every time. So keep the four headings and their order.

Within that, use judgement:

- **TL;DR** carries the most weight. Someone who reads only this line should get the actual answer, not a description of the topic. "Fixed it — the race was in the retry loop, tests pass" tells them something; "Investigated the flaky test and made changes" doesn't.
- **Did** and **Found** are different things and shouldn't be interleaved. Did is what changed in the world (files edited, commands run, PRs opened). Found is what's now known that wasn't before. A pure investigation may have a thin Did and a heavy Found; a mechanical refactor is the reverse.
- Drop a section that's genuinely empty rather than padding it — but only if it's truly empty. **Next** almost never is; if there really is nothing outstanding, say so in a line ("Nothing outstanding — branch is green and pushed") so the reader knows it was considered rather than forgotten.
- If a section runs past ~6 bullets, you're transcribing rather than summarising. Group or cut.

## Keep it faithful

The reader is going to act on this without checking your work, which makes accuracy matter more here than in ordinary output. Two specific ways summaries drift:

**Confidence inflation.** Hedges get smoothed away in compression. If the finding was "this is probably the cause, I didn't verify", it stays probably in the TLDR. Mark unverified claims — "(unverified)" or "suspect" — rather than letting them read as established.

**Quiet omission of bad news.** Failures, skipped steps, and things you couldn't figure out are exactly what the reader needs and exactly what's easiest to leave out. If tests failed, the TLDR says tests failed. If you skipped part of the task, that's a Next item, not a silence.

And only summarise what actually happened. If a detail isn't in the conversation, it doesn't go in the summary — don't reconstruct plausible-sounding specifics from memory of how such work usually goes.

## Length and formatting

Scale to the work: a single turn compresses to 4–8 lines, a long session to maybe 15–20. Anything past one screen has stopped being a TLDR and needs another pass.

It renders as markdown in a terminal:
- Keep `file.py:42` references intact — they're clickable, and they let the reader jump straight to the evidence.
- One level of bullets. Nesting is a sign the structure needs rethinking.
- No preamble ("Here's a summary of..."), no closing offer to elaborate. Open with the TL;DR line and stop after Next.

## Examples

**A focused debugging turn:**

```
**TL;DR** — The flaky checkout test was a shared module-level client; fixed with a per-test fixture, 50 consecutive runs pass.

**Did**
- Replaced the module-level `StripeClient` with a fixture in `tests/conftest.py:31`
- Ran `pytest tests/checkout -x --count 50` — all green

**Found**
- State leaked between tests via the shared client's `_pending` cache (`payments/client.py:88`)
- Only surfaced under `-p xdist` because ordering changed; that's why CI caught it and local didn't

**Next**
- Two other modules build clients the same way (`shipping/`, `refunds/`) — same bug is latent there
```

**A session-scale recap, organised by workstream:**

```
**TL;DR** — Auth migration is code-complete and pushed on `claude/oauth-rgqext`; the rate-limiter work is blocked on a decision about Redis vs in-process.

**Did**
- Auth: swapped session cookies for JWT across 9 handlers, added refresh endpoint, PR #212 open and CI green
- Rate limiting: prototyped a token bucket in `middleware/limit.py`, not wired up

**Found**
- The legacy `/v1/login` path had no CSRF check at all (`api/legacy.py:140`) — predates this work, worth its own fix
- Token refresh needs the clock skew allowance; without it staging fails ~1 in 20 (suspect, seen twice, not root-caused)

**Next**
- Decide Redis vs in-process for the limiter — Redis adds a dependency but survives restarts
- Fix or file the legacy CSRF gap
- PR #212 needs a reviewer
```

Note what both do: the TL;DR line answers the question, findings carry their evidence, and the caveats survive compression.
