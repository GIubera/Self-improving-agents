# self-improving-agents

A Claude skill for designing autonomous agents that improve their own strategy over time, instead of
just running unattended and repeating the same decisions at scale.

Most "autonomous" bots only have one layer: the part that does the job each cycle. They feel hands-off
because they run without you, but they don't actually improve. This skill is about the second layer —
the one that watches results over time and changes how the agent behaves: finding new sources of
signal, dropping the ones that stop working, adjusting its own parameters, and protecting its own
resources.

## What it actually does

It's a **design and critique guide**, not a code generator. Point Claude at a long-running autonomous
system you're building, or one that's misbehaving, and it walks the structure with you: which loops you
have, which you're missing, and where the design is likely to bite.

It won't write the bot for you. What it does instead is map your system onto a small set of moving
parts and tell you which ones are absent — usually the unglamorous ones people skip.

The pattern is domain-agnostic. The worked example is a trading bot, but it maps just as cleanly to a
scraper that has to find and drop sources, a monitoring agent that adapts its thresholds, or anything
where "a source" emits "a signal" you decide whether to act on.

## What's inside

- **`SKILL.md`** — the core: the two-layer model (execution vs. improvement), the five loops a mature
  self-improving agent runs, the decision layer (EV-gating and when a multi-model "council" helps vs.
  when it just adds latency and false confidence), shadow mode for validating sources without risk,
  a set of anti-patterns, and a design checklist you can walk top to bottom.
- **`references/lessons-learned.md`** — the failure catalog: each entry is what went wrong, why, the
  fix, and the general principle behind it.
- **`references/copytrade-case-study.md`** — the whole pattern instantiated on one concrete system, so
  the abstract loops have something to stand on.

## Install

Grab `self-improving-agents.skill` and import it from Claude's Skills settings. Claude reads the
description and decides when to consult it.

## When it kicks in

You don't have to name the pattern — it's tuned to trigger on the symptom too:

- "my bot copies sources that have clearly gone bad and won't stop"
- "this scraper was great for a week, now half the sources are junk"
- "I've got auto-tuning on my agent but a month in nothing's changed — can you audit it?"
- "how do I make an agent that finds new sources and drops the dead ones on its own?"

If you're designing something meant to run by itself and get better without supervision, this is the
thing to reach for.

## A note on tone

It's not a neutral survey. It takes clear positions on what carries weight in these systems and what's
mostly motion, and it explains the reasoning behind each one rather than just asserting it. You won't
agree with all of it — that's the point. It's a distilled point of view, meant to be argued with, not
a checklist to follow blindly.

## Provenance

This skill was developed with AI assistance and manually reviewed. It is based on lessons from running
a real autonomous agent over several months, with sensitive implementation details generalized.
