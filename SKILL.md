---
name: self-improving-agents
description: >-
  Design and critique self-improving autonomous agents: source discovery/pruning, parameter tuning,
  resource protection. Use when a bot/agent should "get better on its own" or keeps repeating
  mistakes.
---

# Self-Improving Agents

## What this is

A **self-improving agent** is a long-running autonomous system that changes its own behavior based on
its own results — not a model that retrains, but an *operational* loop: it finds new sources of signal,
measures how each one actually performs, drops the bad ones, tunes its own knobs, and protects its
resources when things go wrong. The human sets the frame; the agent runs the treadmill.

This guide exists because the gap between "a bot that runs" and "a bot that improves" is where almost
all the pain lives. A bot that runs will happily repeat the same losing behavior forever. The patterns
here come from a real system that went through a difficult cycle — an optimistic-looking peak, then a
deep drawdown, then a slow disciplined recovery — and the lessons are written from what actually broke,
not from theory.

**Your job when this skill triggers:** help the user *design or critique* the self-improvement layer of
their agent. Produce a blueprint and a review, not a code dump. Walk the five core loops, surface the
anti-patterns they're about to hit, and end with the design checklist. Adapt every example to their
domain — the running example here is a copytrade trading bot, but the pattern is domain-agnostic
(swap "wallet/source" and "trade/action" for the user's equivalents).

When the user's system is concrete enough, read `references/copytrade-case-study.md` for a fully
worked example, and `references/lessons-learned.md` for the detailed failure catalog — pull specifics
from there rather than reinventing them.

## The mental model: separate the agent from its improvement layer

The single most useful framing: an autonomous agent has **two distinct layers**, and people conflate
them.

1. **The execution layer** — does the actual job each cycle (copies a trade, scrapes a page, sends a
   message). Fast, frequent, stateless-ish.
2. **The improvement layer** — observes the execution layer's *results over time* and changes how it
   behaves. Slower, periodic, stateful. This is what makes it "self-improving."

Most bots only have layer 1. They feel autonomous because they run unattended, but they don't improve —
they just accumulate the same mistakes at scale. The whole point of this skill is layer 2.

Keep them as separate concerns. The execution layer reads config and source-lists; the improvement
layer *writes* config and source-lists on its own cadence. They communicate through **state files**
(or a small DB), not through shared in-memory cleverness — because the improvement layer needs to
survive restarts and run on its own schedule.

## Domain translation

The pattern is domain-agnostic. Before applying anything below, translate four words into the user's
world — everything else follows from these:

| Generic term | Copytrade bot | News scraper | Monitoring agent | LLM router |
|--------------|---------------|--------------|------------------|------------|
| **source** | wallet you copy | feed / site | sensor / check | model / provider |
| **action** | place a trade | ingest an article | fire an alert | route a call |
| **realized outcome** | resolved P&L | article kept / used | true vs. false alert | answer quality / cost |
| **resource** | capital | API quota / bandwidth | pager budget / attention | tokens / spend |

If you can name those four for the user's system, the loops, the decision layer, and the anti-patterns
all map straight over. When in doubt, anchor on **realized outcome** — it's the one people most often
get wrong (see the paper-value anti-pattern).

## The decision layer: how to act on a signal (EV-gating and the evaluator council)

Before the improvement loops matter at all, the execution layer faces a per-signal question: *a source
just did something — do I act on it?* Copying blindly ("the source acted, so I act") is how you inherit
every one of the source's bad decisions plus your own latency. Two patterns turn blind mimicry into a
bet with an actual edge. Both are load-bearing, and both bit the reference system, so design them with
eyes open.

**EV-gating (the foundation).** Don't act on a signal unless the *expected value* clears a bar.
Estimate a probability of success and a payoff, and compare against the cost of acting:

```
edge = your_estimated_probability − price_implied_probability
act only if edge ≥ threshold   (and size by conviction, see dynamic sizing)
```

In the reference system the market price *was* the crowd's implied probability, so the bot only copied
when its own estimate beat the price by a margin. The generalization: an action is worth taking only
when your information says the odds are better than what you're paying. "Someone I respect did it" is
not an edge — it's a candidate to *evaluate*.

**The evaluator council (estimating the probability).** Where does `your_estimated_probability` come
from? One option that *sounds* great: poll several independent evaluators (the reference system used two
different LLMs — DeepSeek and Claude Haiku — each returning PROB / EDGE / COPY independently) and act
only on consensus. The intuition is sound — independent estimators cancel each other's random errors,
so agreement is a stronger signal than any single model.

**But a council is a sharp tool, and here's what it actually cost** (this is the honest part most
write-ups skip):

| What a council buys you | What it costs you (paid in real losses) |
|-------------------------|------------------------------------------|
| Independent errors cancel → fewer false positives | **Correlated *blind spots* don't cancel.** Two generalist models equally ignorant of a vertical agree on a *wrong* answer — and agreement masquerades as confidence. |
| Consensus = higher conviction | **Latency.** Each evaluator adds seconds. The reference council added 30-60s/signal — a direct reason fast markets (sub-minute crypto) were structurally uncopyable. |
| Edge is explicit and auditable | **Cost.** N API calls per signal, every signal, forever. |
| | **Phantom edge.** Asked to produce a number, a model *will* — sometimes inventing a 20% edge from nothing (the reference system copied a couple of these and lost). |

The killer lesson: **a council of equally-ignorant evaluators does not produce wisdom — it launders
ignorance into false confidence.** Consensus only helps when the evaluators have *real and independent*
information about the domain. On sports favorites, both LLMs said "fairly priced, no edge" and vetoed
exactly the trades the specialist sources nailed — because neither had a model of team form. Two
confident wrong answers agreeing is worse than one, because it *feels* trustworthy.

**This is the origin of the elite-bypass.** Because the council was domain-blind, the reference system
let *verified specialist sources skip the evaluator entirely* in their domain, above a conviction/size
threshold — their realized track record is better evidence than two generalists guessing. So the design
rule is:

- Use **EV-gating always** — never act without an edge estimate, however crude.
- Use a **council when your evaluators have genuine, independent signal** on the domain (and you can
  afford the latency/cost). Demand consensus to cut false positives.
- **Bypass the council for proven specialists on their home turf** — track record outranks a
  generalist panel. (See anti-pattern "domain myopia" below — the council *is* the myopic evaluator.)
- Watch for **phantom edges**: clamp implausibly large estimated edges and treat them as suspect, not
  as green lights.

## The five core loops

A mature self-improving agent runs these as independent periodic jobs (a daemon thread, cron, or a
scheduler), each on its **own cadence**. The cadences matter: discovery is expensive and the world
changes slowly, so it runs rarely; protection is cheap and urgent, so it runs often.

| Loop | Typical cadence | What it does | Failure it prevents |
|------|-----------------|--------------|---------------------|
| **Self-monitoring** | every few hours | snapshots metrics, logs a recap, nudges params | flying blind |
| **Source discovery** | daily | finds new candidate sources, quality-gates them | strategy goes stale |
| **Source pruning** | every few hours | scores active sources, drops the bad ones | bleeding on dead weight |
| **Parameter tuning** | every few hours | adjusts thresholds within guardrails | over/under-trading |
| **Resource protection** | every cycle / ~30 min | stop-loss, cooldowns, sizing, reserve | catastrophic drawdown |

Don't build all five on day one. Build execution + monitoring first so you can *see*. Add pruning next
(it has the highest ROI — killing a bad source stops ongoing losses). Discovery and tuning come after
you trust your metrics. Protection should arrive the first time a single bad stretch scares you — which
will be sooner than you think.

---

## Loop 1 — Self-monitoring (build this first)

You can't improve what you can't see. Before any auto-anything, the agent must record what it did and
how it turned out, on a schedule, in a form you (and it) can read later.

**Design it to capture:**
- per-cycle counters (actions taken, skipped, and *why* skipped — the skip reasons are gold)
- per-source attribution (which source triggered each action — you'll need this for pruning)
- realized outcomes once they're known (not just "I acted" but "the action won/lost")

**The critical distinction: realized vs. mark-to-market.** This burned the reference system badly. The
agent's equity *looked* like it peaked far above reality because open positions were marked at optimistic
current prices. That gain was never real — it was unrealized paper value that evaporated. **Track realized
results as the source of truth for all improvement decisions.** Paper/MTM value is fine for a dashboard,
but never tune, prune, or celebrate based on it. An agent that optimizes against unrealized gains is
optimizing against a number it can't bank.

Output of this loop is just a log line + a state file. That's enough. The point is that every other
loop reads from here.

## Loop 2 — Source discovery (with quality gates)

The agent periodically goes looking for new sources of signal (new wallets to copy, new sites to scrape,
new feeds). The danger isn't finding too few — it's **finding plausible garbage and trusting it.**

**Always gate discovered sources before they go live.** The reference system kept adding "high
win-rate" sources that turned out to be farmers — sources whose impressive stats came from a strategy
that looks great on paper but is uncopyable or fake (e.g., a wallet with 96% win-rate that achieves it
by betting on already-certain outcomes for pennies). Naive metrics actively select *for* these.

Quality gates that earned their place:
- **Reject extreme win-rates with trivial edge** — 96% WR on near-certain outcomes is farming, not skill.
- **Reject pattern mismatch** — a "source" whose actions you structurally *cannot* replicate (too fast,
  wrong timing, different access) is worse than useless. The reference system wasted weeks on sources
  whose signals were always stale by the time it saw them.
- **Require a minimum track record** AND meaningful realized P&L, not just rate.
- **Categorize, don't just score** — tag *what kind* of source it is, because a source can be great in
  one category and garbage in another (see the "domain myopia" lesson below).

Write discovered sources to a file the execution layer reads (e.g., `dynamic_sources.json`), separate
from your hand-curated list, so you can always tell what the agent chose vs. what you chose.

## Loop 3 — Source pruning (highest ROI loop)

The mirror of discovery: measure every active source's *realized* performance and **cut the
persistent losers automatically.** This is usually the single most profitable loop, because a bad
source bleeds you every cycle until something stops it — and "you noticing manually" is too slow.

**The decision rule needs two thresholds, not one:**
- a **minimum sample** before judging (e.g., ≥10 realized outcomes) — so you don't kill a good source
  on a 2-loss unlucky streak
- a **performance floor** (e.g., realized win-rate < 50%) — below which it's cut

**Crucial refinement learned the hard way: exempt your verified elite sources from auto-pruning.** A
source with a long, deep track record (hundreds of outcomes, strong realized P&L) that hits 2-3 losses
is experiencing *variance*, not decline. The reference system's pruner benched one of its best
long-term sources on a short losing streak — a false positive that cost real opportunity. Maintain an
"elite/verified" set that the pruner skips. New and mid-tier sources get pruned aggressively; proven
ones get the benefit of the doubt. (See the cooldown lesson for the same principle applied faster.)

**Prefer graduated states over a binary kill switch.** On/off is brittle — a source flickers in and out
on every wobble (annoying "flapping") and a single bad week deletes a good source for good. Model a
source's standing as a small **state machine** driven by its rolling realized score:

```
DISCOVERED → (shadow trial) → ACTIVE ⇄ OBSERVATION → SUSPENDED → ARCHIVED
                                  ↑__________________________|  (recovers)
```

- **ACTIVE** — score above the high bar; full participation / full size.
- **OBSERVATION** — score slipped below the bar; keep acting but at reduced size and flag outputs as
  low-confidence. A warning, not a death sentence.
- **SUSPENDED** — score under the low bar for *N consecutive* checks (persistence requirement kills
  flapping); stop acting but **keep tracking in shadow** (see below) so it can earn its way back.
- **ARCHIVED** — suspended a long time; out of the active loop but kept in history.

Graduation requires *persistence* (multiple checks under threshold), not a single reading — this is the
"thresholds over samples" discipline made structural. Sizing scales with state, so degradation reduces
exposure *gradually* instead of all-at-once.

**Add a human-review queue for the few high-stakes ambiguous calls.** Full autonomy is the goal, but the
fastest path there is to let the agent decide everything *clear-cut* on its own and escalate only the
genuinely ambiguous, high-impact decisions to a tiny weekly queue: "a long-trusted source is
collapsing — confirm before I suspend it?", "a borderline candidate wants promotion?". Clear wins and
clear losers: automatic, zero notifications. This keeps the human's involvement at minutes-per-week
(reviewing edge cases) instead of babysitting, and it shrinks over time as you trust the thresholds.
The point isn't to keep a human in the loop forever — it's to *earn* full autonomy decision by decision.

## Loop 4 — Parameter auto-tuning (the seductive trap)

The agent adjusts its own thresholds (how selective to be, how big to act, how far ahead to look).
This *feels* like the essence of self-improvement, but it's the **most overrated and most dangerous**
loop. Read this section as a warning as much as a how-to.

**Why it's dangerous:** parameter tuning gives the illusion of improvement while changing nothing
fundamental. The reference system's tuner spent two months oscillating one threshold between 1.2% and
2.5% and back, writing confident reasoning each time about why this nudge would help. It never helped.
The bot's real problems were structural (bad sources, no capital protection, wrong market types) and
**no amount of threshold-fiddling touches a structural problem.** A tuner that's "always active" can
fool you into thinking the system is adapting when it's just twitching.

If you build it anyway (sometimes you should — some params genuinely have a learnable optimum):
- **Hard guardrails (min/max clamps).** The tuner must never be able to set a parameter to a value
  that risks the whole system. The reference tuner once quietly raised max-bet-size 3x; a clamp would
  have caught it.
- **Tune on realized outcomes, never on paper P&L** (same trap as Loop 1).
- **Make it write its reasoning** to a log. Not because the reasoning is always right — it often
  wasn't — but because reading a month of "reasoning" is how *you* notice the tuner is just oscillating.
- **Prefer structural fixes.** When tempted to add a tuning rule, first ask: "is the problem really
  this threshold, or is it a bad source / missing guardrail / wrong market?" Usually the latter.

## Loop 5 — Resource & capital protection (build this earlier than you want to)

The loops above make the agent *better*; this one keeps it *alive*. Every autonomous agent that spends
a finite resource (money, API quota, reputation, compute) needs automatic brakes, because the failure
mode of "run unattended" is "run unattended straight off a cliff."

Protection mechanisms that proved their worth:

- **Stop-loss.** Exit a position/commitment automatically once it's down past a threshold *and* has had
  time to be wrong (e.g., −50% after 7+ days). The time condition matters — don't panic-sell normal
  short-term swings. Recovering 50% of a doomed position beats riding it to zero.
- **Cooldown after a streak.** When a source produces N consecutive losses, bench it for a fixed window
  (e.g., 48h) instead of waiting for the slow pruner. This catches a source having a *bad day* faster
  than the 10-sample pruner can. Same elite-exemption applies: verified sources don't get cooled on a
  short streak.
- **Dynamic sizing.** Scale each action's size by the source's recent realized win-rate — half size for
  shaky sources, normal for solid, a modest bump for proven. You stay exposed to your best, less to
  your worst, automatically.
- **Reserve floor.** Never deploy 100% of the resource. Keeping a reserve (e.g., 20-30%) means a great
  opportunity doesn't find you fully locked up — which, in the reference system, was a chronic problem:
  capital perpetually tied up in slow-resolving commitments with nothing free to act on fresh signals.
- **Avoid concentration.** Cap how much of the resource sits in correlated bets. The reference system
  once had the majority of its capital in one theme, so positions that should have diversified instead
  all moved together.

---

## Shadow mode: simulate before (and instead of) committing

A technique that pays for itself across discovery, pruning, and debugging: **run a source in
simulation — record what *would* have happened if you'd acted, without spending the resource.** Cheap,
risk-free, and it answers questions live trading can't.

Three jobs it does:
1. **Vetting new sources** before they go live — a discovered source spends a trial period in shadow;
   you promote it to ACTIVE only once its *simulated realized* performance clears the bar. Garbage
   that looks good on headline stats gets caught on paper, before it touches the real resource.
2. **Earning suspended sources back** — a SUSPENDED source keeps trading in shadow, so it can prove a
   recovery without risk.
3. **Separating "bad source" from "bad execution" — the diagnostic most people miss.** If a source is
   *profitable in shadow* but your *real copies lose*, the source isn't the problem — **your execution
   is** (latency, slippage, worse fills because you act after them). These are completely different
   bugs with different fixes, and without shadow you can't tell them apart. The reference system's BTC
   sub-minute experiment was killed exactly this way: a dry-run proved the top wallets' edges were gone
   by the time its ~20-60s latency could copy them — uncopyable on paper, before risking a cent.

The general rule: **before you trust a new source category, simulate copying it at your real latency.**
If you can't make money in a no-cost simulation, you certainly won't with fees and slippage.

---

## The anti-patterns (read these aloud to the user)

These are the mistakes that actually happened. They're worth more than the patterns above because
everyone rebuilds the patterns and everyone repeats these. Full detail in
`references/lessons-learned.md`; the headline set:

1. **Optimizing against paper value.** Covered above and worth repeating: the "peak" that isn't real
   will drive terrible decisions. Realized-only for all improvement logic.

2. **Reacting to single outcomes.** A 75-90% win-rate source *will* lose 1 in 4-10 times. Pausing or
   ripping out a source after one or two losses is noise-chasing. The discipline is to act on
   *thresholds over samples*, not on the sting of the last result. (The human running the reference
   system caught the agent — and the agent's operator — doing this repeatedly.)

3. **Domain myopia in evaluation.** If part of your agent judges quality with an LLM or a generic
   model, beware: it can be confidently wrong in verticals it doesn't understand. The reference
   system's AI-eval rated sports favorites as "no edge / fair value" because it had no model of the
   domain, and rejected exactly the trades the specialist sources got right. Fix: let *verified
   specialist sources bypass the generic evaluator* in their domain. Track record beats a generalist's
   guess on home turf.

4. **The duplicate-list bug.** A source on both a blocklist and an allowlist — and the allowlist wins.
   This silently re-enabled sources that had been explicitly killed, more than once. Whenever you have
   multiple membership lists, enforce precedence explicitly (blocklist always wins) and assert it.

5. **Filter overfitting.** After a few bad trades it's tempting to add a hyper-specific rule ("skip
   markets matching this exact pattern"). These rules accrete, contradict each other, and fit the last
   disaster instead of the next one. Prefer a few principled gates with explained reasoning over a
   museum of one-off patches.

6. **Capital lock-up / velocity blindness.** An agent optimizing per-action profit can still fail if
   every action ties up the resource for months. The reference system made "good" geo bets that locked
   capital for 60-90 days while faster opportunities passed. Track *resource velocity* (return per unit
   time the resource is committed), not just per-action edge.

7. **"Self-improvement" that's just parameter twitching.** Covered in Loop 4. If your improvement
   layer only ever nudges thresholds, it isn't improving — it's fidgeting. Real improvement adds/removes
   sources and changes structure.

8. **Catastrophizing into a full stop.** The inverse failure: after a drawdown, ripping the whole
   system out. A drawdown in a sound framework is a tuning signal, not a death sentence. The reference
   system recovered most of a deep drawdown by adding protection loops, *not* by quitting. Distinguish
   "this framework is broken" from "this framework needs brakes."

## Design checklist

Use this to design a new self-improving agent or to audit an existing one. Walk it with the user and
flag every box they can't check.

**Foundations**
- [ ] Execution layer and improvement layer are cleanly separated, communicating via state files/DB.
- [ ] Every action is logged with its triggering source and its skip-reason (if skipped).
- [ ] Outcomes are recorded as **realized**, and realized is the source of truth for all improvement.
- [ ] Paper/MTM value, if shown at all, is never used for tuning/pruning decisions.

**Decision layer**
- [ ] Actions pass an **EV/edge gate** — you never act without an edge estimate vs. the implied price.
- [ ] If you use an evaluator council, its members have *real, independent* domain signal (not two
      generalists agreeing). You can afford its latency/cost.
- [ ] Verified specialists **bypass** the council on their home turf.
- [ ] Implausibly large estimated edges are clamped/flagged as suspect (phantom-edge guard).

**Improvement loops**
- [ ] Self-monitoring runs on a schedule and writes a human-readable recap.
- [ ] Discovery quality-gates candidates (anti-farmer, anti-pattern-mismatch, min track record).
- [ ] Pruning uses *minimum-sample AND performance-floor*, and exempts a verified-elite set.
- [ ] Sources move through **graduated states** (active/observation/suspended/archived) with a
      persistence requirement, not a binary on/off that flaps.
- [ ] A **human-review queue** catches the few high-stakes ambiguous calls; clear cases are automatic.
- [ ] **Shadow mode** vets new sources and diagnoses bad-source vs. bad-execution before real spend.
- [ ] Tuning (if present) has hard min/max clamps and logs its reasoning. You've asked "is this
      structural?" before trusting it.

**Protection**
- [ ] Automatic stop-loss with a time condition.
- [ ] Cooldown after a loss streak (elite-exempt).
- [ ] Dynamic sizing tied to recent realized performance.
- [ ] A reserve floor so the agent is never 100% committed.
- [ ] Concentration cap across correlated actions.

**Hygiene**
- [ ] List precedence is explicit and asserted (blocklist beats allowlist).
- [ ] Restart-safety: improvement state survives process restarts (it lives in files/DB, not memory).
- [ ] Resource velocity is tracked, not just per-action edge.
- [ ] There's a clear line between "needs brakes" and "is structurally broken" — and a plan for each.

## How to run a session with this skill

1. **Identify the two layers** in the user's system. If they only have execution, that's the headline.
2. **Map their domain onto the five loops.** What's a "source"? What's an "action"? What's "realized"?
3. **Check the decision layer.** Do they act on an EV/edge gate, or copy blindly? If they use an
   evaluator council, is it giving real independent signal or laundering ignorance into false
   confidence? Should specialists bypass it?
4. **Find which loops are missing.** Almost always: monitoring is thin, pruning is manual, protection
   is absent. Prioritize in that order.
5. **Surface the anti-patterns they're closest to.** Use the list above; pull specifics from
   `references/lessons-learned.md`.
6. **Walk the checklist** and hand them the unchecked boxes as a prioritized to-do.
7. If they want a concrete reference implementation to copy structure from, point them at
   `references/copytrade-case-study.md`.

Keep it grounded and specific to *their* system. The worst version of this skill is a generic lecture;
the best version is "here are the three loops you're missing and the two anti-patterns you're already
living in."
