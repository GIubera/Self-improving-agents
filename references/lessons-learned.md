# Lessons Learned — the failure catalog

Detailed write-up of failures from a real self-improving copytrade agent over ~2 months. Each entry:
what happened, why it happened, the fix, and the generalized principle. Read the entry that matches the
user's situation and pull the specifics; don't recite the whole file.

## Table of contents
1. The paper-value mirage (peak that wasn't real)
2. Single-outcome reactions (noise-chasing)
3. Domain myopia in the evaluator
4. The duplicate-list bug (blocklist vs allowlist)
5. Farmer sources passing naive quality filters
6. Pattern-mismatch sources (structurally uncopyable)
7. Filter overfitting
8. Capital lock-up and velocity blindness
9. Parameter-tuning theater
10. Cooldown false-positives on elite sources
11. Concentration risk
12. The catastrophize-and-quit reflex

---

## 1. The paper-value mirage

**What happened.** Agent equity appeared to peak well above its starting capital. Months later the
realized equity was much lower. That "peak" was mark-to-market: open positions valued at optimistic
current prices that later collapsed or were never realizable. Decisions and morale anchored to a number
that never existed in the bank.

**Why.** Open-position value is a *forecast*, not a result. In any system where actions resolve later
(trades, bets, long-running tasks), unrealized value is systematically biased toward whatever the
current quote says — and quotes are most optimistic exactly when you're most exposed.

**Fix.** Two separate numbers: `realized_pnl` (banked, immutable once resolved) and `mtm_value`
(display only). Every improvement decision — prune, tune, size, celebrate — reads realized only.

**Principle.** *An agent that optimizes a number it can't bank will optimize itself broke.* Realized is
truth; paper is a dashboard.

## 2. Single-outcome reactions

**What happened.** A source would lose once or twice and get paused/blacklisted, then a wallet would win
big right after being benched. Repeatedly. Both the agent's logic and its human operator did this.

**Why.** A source with 75% win-rate loses 25% of the time — clusters of 2-3 losses are *expected*, not
signal. Human loss-aversion (and naive thresholds) treat the most recent sting as information.

**Fix.** Decisions act on **thresholds over a minimum sample** (e.g., WR<50% over ≥10 outcomes), never
on the last 1-2 results. For urgent cases use a cooldown (timed bench) rather than a permanent kill, so
variance doesn't cause irreversible decisions.

**Principle.** Set the rule by the distribution, not the last draw. Reversible (cooldown) beats
irreversible (blacklist) when the sample is small.

## 3. Domain myopia in the evaluator

**What happened.** The agent used an LLM to score each candidate action's "edge." On sports markets it
rated clear favorites as "fair value, no edge" and rejected them — exactly the actions that specialist
sources (90%+ realized WR in that niche) were nailing. The generalist evaluator had no model of the
domain and confidently vetoed domain experts.

**Why.** A generic evaluator knows generic priors. In a vertical with private structure (team form,
matchup history, domain timing), "looks fairly priced to me" is ignorance, not insight. The evaluator's
confidence was inversely related to its competence here.

**Fix.** Let **verified specialist sources bypass the generic evaluator within their domain**, above a
size/conviction threshold. Their realized track record is better evidence than a generalist's guess.
Keep the evaluator for sources/domains without a proven specialist.

**Principle.** On home turf, a proven track record outranks a generalist model's opinion. Know when your
evaluator is out of its depth and route around it.

## 4. The duplicate-list bug

**What happened.** A source was added to the blocklist after bad performance, yet kept getting copied.
Cause: it was *also* present in the override/allowlist, and the wallet-pool builder added allowlist
entries without checking the blocklist. This happened to **multiple** sources across the run.

**Why.** Multiple membership lists with no explicit precedence. Each list was "right" locally; the
union was wrong.

**Fix.** One function builds the active set, and blocklist subtraction is the *last* step, applied
unconditionally. Add a startup assertion: `assert active_set.isdisjoint(blocklist)`.

**Principle.** When multiple lists govern membership, precedence must be explicit, centralized, and
asserted — never emergent from the order code happens to run.

## 5. Farmer sources passing naive filters

**What happened.** Discovery kept surfacing wallets with gaudy stats (96% WR, 100% WR) that were
"farmers" — they manufacture the stat by buying near-certain outcomes at $0.96-1.00 for a cent of edge,
or by gaming a leaderboard score. Copying them produced fees and near-zero/negative edge.

**Why.** Win-rate alone is trivially gameable. "Bet only on things that already happened" maximizes WR
and means nothing.

**Fix.** Multi-gate: reject WR>~92% paired with trivial average edge; reject price-at-entry in the
near-certain band; require realized P&L magnitude, not just rate; cap markets-traded (extreme counts +
tiny avg size = farming). Categorize the source's actual behavior before trusting the headline number.

**Principle.** Any single metric a source can optimize directly is a metric a bad source *will* optimize.
Gate on combinations that are hard to fake together.

## 6. Pattern-mismatch sources

**What happened.** The agent added sources whose signals it structurally could not act on in time —
e.g., live in-game sports bets, or 5/15-minute crypto markets dominated by sub-second HFT bots. By the
time the ~20-60s polling agent saw the trade, the price had moved past any edge. Weeks of "candidates"
that were always stale on arrival.

**Why.** Copyability is a *function of your own latency and access*, not of the source's quality. A
brilliant source you can't follow in time is noise to you.

**Fix.** Add a copyability gate: reject sources whose median action-to-deadline window is shorter than
your reaction latency with margin. The agent measured "how late in the window does this source act?" and
dropped sources that act inside its own lag. For a worked example of *proving* a market is uncopyable
before spending real money, see the case study's dry-run analysis.

**Principle.** Filter sources by what *you* can replicate, not by how good they look in the abstract.

## 7. Filter overfitting

**What happened.** After specific bad trades, hyper-specific skip rules were added ("skip this exact
market-name pattern", "block this date format after 13:00 UTC"). They accumulated, occasionally
contradicted each other, and targeted the last disaster rather than the next.

**Why.** Each patch was a local fix to a salient recent pain. The sum was a brittle, hard-to-reason-about
filter stack fit to history.

**Fix.** Prefer a small set of *principled* gates, each with a written rationale, over many one-off
patches. When adding a rule, ask "what general category of mistake does this prevent?" If the honest
answer is "this one specific event," don't add it.

**Principle.** Filters should encode categories of error, not memorialize individual incidents.

## 8. Capital lock-up and velocity blindness

**What happened.** The agent made "good" long-horizon bets (resolving in 60-90 days). Per-bet edge was
fine, but capital sat frozen for months while faster opportunities passed and the free balance hovered
near zero. The system was perpetually "out of funds" despite a healthy portfolio on paper.

**Why.** Optimizing per-action edge ignores the *time* the resource is committed. Two actions with equal
edge are not equal if one frees the resource in a day and the other in a quarter.

**Fix.** Track **resource velocity** = realized return per unit of resource-time committed. Cap exposure
to slow-resolving actions. Keep a reserve so fast opportunities aren't missed. Consider exiting winners
early (taking profit once a position is comfortably in the green) to recycle capital rather than waiting
for full resolution.

**Principle.** Edge-per-action is half the equation; edge-per-resource-per-time is the whole one.

## 9. Parameter-tuning theater

**What happened.** An auto-tuner adjusted thresholds every few hours, each time logging confident
reasoning. Over two months one threshold oscillated 1.2%↔2.5% endlessly. It produced motion, not
progress, and masked that the real problems were structural (bad sources, no protection).

**Why.** Tuning is the most *visible* form of "adapting" and the least *consequential* when the binding
constraint is elsewhere. A busy tuner feels like a working brain.

**Fix.** Hard clamps on every tunable. Log reasoning so *you* can spot oscillation. Before adding any
tuning rule, force the question: structural problem or threshold problem? Default to structural. Some
params do have a real optimum — but treat the tuner as a minor refinement, never as the improvement
strategy.

**Principle.** Motion is not progress. If your "self-improvement" only nudges numbers, it isn't.

## 10. Cooldown false-positives on elite sources

**What happened.** The cooldown/pruner benched one of the best, most-proven long-term sources after a
short losing streak — treating a deep-track-record source the same as a random new one.

**Why.** A flat rule ("2 losses → bench") ignores prior. For a source with hundreds of wins, two losses
is overwhelmingly variance; for a new source it might be signal.

**Fix.** Maintain an **elite/verified set** (long track record, strong realized P&L) exempt from
cooldown and aggressive pruning. Aggressive rules apply to new/mid-tier sources; proven ones get
patience. Re-derive eligibility periodically from realized stats so "elite" stays earned.

**Principle.** Apply your priors. The right reaction to a loss depends on how much you already know about
the source.

## 11. Concentration risk

**What happened.** A majority of capital ended up in a single correlated theme. Positions meant to
diversify instead all moved together; when the theme turned, everything turned at once.

**Why.** Per-action selection had no view of portfolio correlation. Each bet was individually reasonable;
collectively they were one bet.

**Fix.** Cap the share of the resource in any one theme/cluster. Detect correlation (shared tags,
underlying, timeframe) and refuse to pile on past a limit.

**Principle.** Diversification is a portfolio property; it can't be enforced one action at a time without
a global view.

## 12. The catastrophize-and-quit reflex

**What happened.** Deep into a drawdown, the tempting conclusion was "the whole approach is broken, stop
it." It wasn't broken — it lacked brakes. Adding the protection loops (stop-loss, cooldown, sizing,
reserve) recovered most of the drawdown over the following weeks.

**Why.** A deep drawdown is emotionally indistinguishable from failure, but they're different: a sound
framework without risk controls *will* draw down hard, and that's a fixable gap, not a death sentence.

**Fix.** Before quitting, ask: "is the edge structurally absent, or is the system just unprotected?" If
the winners are real and the losers are unmanaged, the answer is brakes, not burial.

**Principle.** Distinguish "no edge" from "no brakes." The first kills a project; the second is a
weekend of work.
