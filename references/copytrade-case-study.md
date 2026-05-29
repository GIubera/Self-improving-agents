# Case Study — a self-improving copytrade bot on prediction markets

A concrete instantiation of the five-loop pattern, from a real Polymarket copytrade agent. Use this to
show the user how the abstract loops map to a working system, and to borrow structure. The domain is
trading, but read it as a template: **source = wallet to copy, action = trade, realized = market
resolution payout.**

## System shape

- **Execution layer**: every ~20s, poll a set of monitored wallets via API, aggregate their recent
  buys into signals, filter, and place a scaled-down copy of qualifying trades. Reads `bot_config.json`
  (params) and a wallet pool (curated + auto-discovered) every cycle.
- **Improvement layer**: a daemon thread running five jobs on independent cadences, writing back to the
  config and source lists the execution layer reads.

The two layers share nothing but files: `bot_config.json`, `dynamic_wallets.json` (auto-discovered),
`wallet_performance.json` (realized stats), `wallet_cooldowns.json`, `auto_blacklist.json`. This is what
makes the improvement layer restart-safe and independently schedulable.

## The five loops, concretely

### Monitoring (every 3h)
Logs a recap: copied / skipped / skip-reasons / errors / no-funds counts. Skip-reasons turned out to be
the most valuable telemetry — "why didn't we act" revealed the duplicate-list bug, the farmer floods,
and the stale-signal problem long before P&L did.

### Discovery (every 24h)
Runs a scanner over thousands of top traders, filters for the agent's niches, applies anti-farmer gates,
and writes the survivors to `dynamic_wallets.json`. Gates that mattered: reject WR>92% + trivial edge,
reject avg-size too small, require realized P&L band ($30k–$1M lifetime), categorize by sport/geo/etc.
Auto-discovered wallets are kept in a *separate* file from the hand-curated override list so the agent's
picks are always distinguishable from the human's.

### Pruning (every 6h)
Reads `wallet_performance.json`, computes realized WR per source, and writes sources with **≥10 outcomes
and WR<50%** to `auto_blacklist.json`. Exempts an `ELITE_GEO_SET` of verified long-track-record wallets.
The boot-time wallet-pool builder subtracts `auto_blacklist.json` *last*, unconditionally — the fix for
the duplicate-list bug.

### Tuning (every 3h)
Adjusts `MIN_EDGE_PCT`, `MIN_ORIGINAL_TRADE`, `MAX_BET_USDC`, `MAX_DAYS_TO_RESOLVE` within clamps, and
writes a `reasoning` string into the config each time. **This loop was the least useful** — months of
oscillation. Its main value in hindsight was the reasoning log, which is how the oscillation became
visible. If rebuilding, this loop would be demoted to rare, small adjustments with tighter clamps.

### Protection (every cycle + every 30 min)
- stop-loss: sell positions ≤50% of entry after 7+ days open
- cooldown: 48h bench for wallets with a recent loss streak (elite-exempt)
- dynamic sizing: 0.5× / 1.0× / 1.3× by recent realized WR
- (recommended additions the system was missing early: reserve floor, concentration cap)

## What each anti-pattern looked like here

- **Paper mirage**: equity "peaked" on paper (MTM) far above the realized value. Now realized-only drives decisions.
- **Single-outcome reactions**: a top LoL wallet went 1W-3L in a day and got paused; it was variance.
  Fixed by threshold-over-sample + elite exemption.
- **Domain myopia**: the AI evaluator rejected sports favorites the specialists nailed. Fixed with an
  elite-bypass-AI path for proven specialist wallets above a size threshold.
- **Duplicate-list bug**: `wallet-A` and `wallet-B` sat on blocklist *and* override; kept trading.
  Fixed with last-step unconditional blocklist subtraction + disjoint assertion.
- **Farmer**: `wallet-C` showed 87% WR but it was NO-on-near-certain-IPO farming. Blacklisted; gate added.
- **Pattern-mismatch**: BTC 5/15-minute markets — a dry-run analysis proved the top wallets' edges were
  gone by the time a 20-60s-latency poller could copy them (HFT territory). Conclusion: structurally
  uncopyable; don't spend real money. Good example of *proving* uncopyability before committing.
- **Capital lock-up**: geo bets resolving in 60-90 days kept the balance near zero. Lesson logged as
  "track resource velocity"; partial fix was selling winners early to recycle capital.

## Dry-run before real money (a transferable technique)

Before trusting a risky new source category, the agent ran a **dry-run analysis**: pull historical
resolved markets, reconstruct each candidate wallet's trades, compute realized WR/P&L, and — critically —
measure *entry timing vs. the action deadline* to test copyability under the agent's real latency. This
killed the BTC-15m idea on paper, saving real capital. Any user adding a new source type should do this:
simulate the copy with your actual lag before believing the source's headline stats.

## Sequencing that worked (and didn't)

What the system did: built execution first, bolted on loops reactively as pain arrived — monitoring,
then discovery, then pruning, then tuning, then (too late) protection.

What it *should* have done: execution → monitoring → **protection** → pruning → discovery → (maybe)
tuning. Protection arriving last is why a normal bad stretch became a deep drawdown. If you advise a user,
push protection much earlier than instinct suggests — it's cheap and it's the difference between a
drawdown and a blowup.
