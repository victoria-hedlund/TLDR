# Eval runs for the `tldr` skill

Run results live here; the eval *definitions* (prompts, assertions, fixtures) live with
the skill at `.claude/skills/tldr/evals/`. The split keeps run artefacts out of the skill
bundle, which otherwise grows by a full result set every iteration when packaged.

## Layout

```
iteration-1/
├── review.html                  self-contained viewer — outputs + benchmark, open in a browser
├── benchmark.json / .md         aggregated pass rates, timing, tokens, analyst notes
└── eval-<n>-<name>/
    ├── eval_metadata.json       prompt + assertions for this case
    ├── with_skill/              outputs/tldr.md, grading.json, timing.json
    └── without_skill/           same, for the no-skill baseline
```

`review.html` is the readable entry point — it renders every output side by side with its
baseline and carries the benchmark tab.

## Iteration 1 — results and caveats

Headline: skill 29/29 assertions, baseline 19/29. Executor was Sonnet, one run per
configuration, graded by the same session that wrote the skill.

**Read that number with the following in mind. It is weaker than it looks.**

1. **Fixture contamination (most important).** `debug-session.md` and `long-session.md`
   reuse the same scenarios as the two worked examples inside `SKILL.md` — the same
   `StripeClient` / `payments/client.py:88` story and the same `claude/oauth-rgqext` /
   `api/legacy.py:140` / Redis story. On those two cases the with-skill run had a worked
   answer in context and the baseline did not, so part of the gap is template copying
   rather than judgement. `trivial-last-turn.md` (streaming export) is the only clean
   comparison, and it is the one to trust.
2. **Self-grading.** The skill author wrote the assertions and graded the runs. Borderline
   calls were resolved in the skill's favour — e.g. eval-2's "organises by workstream"
   passed on implicit grouping rather than labelled sections.
3. **Nine of 29 assertions test format compliance** (headings present, no preamble, line
   ceiling). These are close to tautological: the skill mandates the scaffold, so the
   skill-run has it and the baseline does not. They inflate the delta without saying
   anything about summary quality.
4. **Three assertions cannot fail as written.** "Makes no new tool calls" is meant to guard
   the skill's most important rule — don't re-investigate — but the runs happened against
   an empty repo, so there was nothing to investigate. The guard is currently vacuous.
5. **One run per configuration**, so there is no variance data behind the 100%.

What the clean eval did show is real, and it matches the qualitative read across all three:
the skill does not make summaries more *accurate* — Sonnet's baseline summaries preserved
hedges, bad news and tradeoffs perfectly well — it makes them *consistent and scannable*.
Fixed section order, bottom line first, and `file:line` references surviving compression
(kept in 3/3 skill runs, dropped in 3/3 baselines).

## Known gaps in coverage

Nothing yet exercises: the rolling "second `/tldr` covers only what's happened since" rule,
topic-scoped requests (`/tldr the auth bug`), a session where **Next** is genuinely empty,
an aborted or failed task, or proactive triggering without the slash command.

## What iteration 2 should change

Rewrite fixtures 0 and 2 into domains that share nothing with the skill's examples; run the
anti-re-investigation case against a repo with real code to be tempted by; cut the format
assertions to two; add fixtures for the rolling and empty-Next cases; three runs per
configuration instead of one.
