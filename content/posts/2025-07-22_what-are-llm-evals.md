+++
title = "What LLM Evals Actually Are (and Why 'Looks Good to Me' Isn't a Strategy)"

[taxonomies]
tags = ["AI", "Machine Learning"]
+++

A lot of teams building on top of LLMs ship a prompt, look at a handful of
outputs, decide it "seems good," and move on. That works right up until a
model update, a prompt tweak, or an edge case quietly breaks something
nobody's testing for. Evals are the fix — the same instinct that gives you
unit tests and regression tests for regular code, applied to something
that doesn't produce the same output twice.

<!-- more -->

## Why this is harder than normal testing

Traditional tests check for exact or near-exact output: given this input,
expect this output. LLM output is non-deterministic and open-ended — the
same prompt can produce meaningfully different (but equally valid)
phrasing every time. `assertEqual` doesn't work here. Evals are the set of
techniques for answering "is this output good?" when "good" doesn't mean
"identical to a fixed string."

## The main flavors of evals

**Golden datasets with reference answers.** Curate a set of representative
inputs with known-good expected outputs (or acceptable ranges of outputs),
and run them against the model whenever the prompt, model version, or
pipeline changes. This is the closest thing to a regression test suite,
and it's the first thing worth building — even 20-30 well-chosen examples
that cover your known edge cases catches a surprising number of
regressions before they reach users.

**Rubric-based scoring.** Rather than one "correct" answer, define a
rubric — a checklist of properties an acceptable answer must have (did it
cite a source, did it refuse when it should have, did it stay under a
length limit, did it avoid a specific failure mode you've seen before).
Score against the rubric, either by a human or by another model.

**LLM-as-judge.** Use a second (often more capable) model to evaluate the
first model's output against criteria you specify. This scales far better
than human review, but it comes with its own failure modes — judge models
have their own biases, tend to prefer longer or more confident-sounding
answers regardless of correctness, and need their own validation against a
smaller set of human-labeled examples to confirm they're actually judging
the thing you care about.

**Human review, sampled.** Even with automated evals in place, periodic
human review of a random sample of real production outputs is worth
keeping. Automated evals only catch what you thought to test for; human
review catches the failure modes you didn't anticipate.

## What to actually measure

Good eval criteria are usually specific to the task, but a few categories
show up almost everywhere:

- **Task success** — did it actually do the thing (answer the question,
  extract the right fields, follow the instruction)?
- **Groundedness** — for anything RAG-adjacent, did the answer stay
  consistent with the retrieved context, or did it hallucinate details not
  present in the source material?
- **Format compliance** — if you need JSON, valid JSON matching a schema;
  if you need a specific structure, does it match?
- **Safety/refusal behavior** — does it refuse when it should, and *not*
  refuse when it shouldn't (over-refusal is a real, measurable failure
  mode, not just a nice-to-have)?

## Where evals fit in the workflow

The useful pattern is treating evals like CI: run the golden dataset
automatically whenever you change a prompt, swap a model, or update a
retrieval pipeline, and treat a drop in eval scores the same way you'd
treat a failing test — something to investigate before shipping, not
something to notice after users complain. Combined with periodic sampled
human review of live traffic (to catch what the golden dataset doesn't
cover yet, and to grow the dataset over time), this turns "does the prompt
still work" from a vibe into a number you can track.

The teams that get burned hardest by LLM regressions are usually the ones
with no evals at all — where the first signal that something broke is a
support ticket, not a failing check.
