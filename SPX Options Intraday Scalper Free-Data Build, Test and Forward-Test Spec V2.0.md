# SPX Options Intraday Scalper: Free-Data Build, Test and Forward-Test Spec (v1.0, no Unusual Whales dependency)

For use with: Claude (Opus 5.5 or similar), Gemini, ChatGPT, or any capable LLM. Paste this entire document as the first message.

---

## 0. READ FIRST: mandatory notice and instructions to the AI

**This strategy is an unvalidated hypothesis. It MUST be tested (Section 10) and paper traded before any real capital is used.** Nothing here shows the rules have an edge. Intraday SPX option scalping is high risk and most such strategies lose money after costs. This is not financial advice.

Instructions to the AI:

1. Do NOT write live-trading code in your first response. First produce a **design document** (architecture, data model, diagrams, test plan, open questions) and **wait for my approval**.
2. Build in this order only: data recorder, proxy test harness, paper trading, then (only after the gates in Section 12 pass and I explicitly approve) a live adapter.
3. Never invent API endpoints, field names, data availability or results. If unsure, read the official docs or ask me. Never state backtest numbers that code did not compute.
4. Risk limits in Sections 7 and 8 are hard constraints enforced in code, never suggestions to a model.
5. Ask for anything missing (broker, accounts, hardware, constraints) in one batch before assuming it.
6. Point out any flaw or unrealistic assumption in this spec instead of silently following it.

---

## 1. Objective and scope

Automate an intraday scalping process I run manually from a thinkorswim watchlist:

- Instrument: SPX/SPXW index options, long single-leg calls or puts only (no spreads, no short premium).
- Style: quick trades (minutes), 1 to 5 contracts.
- Signals: option VWAP, volume, and Custom 1 (= VWAP x Volume, premium traded today in index points; x100 for dollars), on a ladder of strikes (multiples of 5 or 10) around the money, kept centered on the index.
- Window: 06:40 to 09:00 US Pacific (09:40 to 12:00 ET), flat by 09:15 PT. No overnight positions.

Out of scope for v1: multi-leg strategies, other underlyings, swing trades, tax or legal structuring.

---

## 2. The data reality (why this version exists)

This version does not use Unusual Whales or any paid data. The honest constraint is that **free, historical, intraday SPXW option data with per-contract volume and VWAP essentially does not exist**. Free sources give delayed quotes of today's chain, daily bars, or short windows. Intraday SPX/SPXW history is sold by vendors (priced for professionals), and some cheaper products only give coarse snapshots (for example every 15 minutes), which is too coarse for a minutes-long scalper.

So the free plan has three tracks, in this order of importance:

- **Track 1: Forward recording (the real test).** Record the live option ladder myself from my broker's data stream every day, build VWAP and Custom 1 from it, and replay those recorded days through the strategy. This is the only free way to test the actual signals. It takes weeks to accumulate enough data.
- **Track 2: Proxy backtest on free underlying data with modeled option prices.** Tests the exit logic, guardrails, costs and break-even math over years of history. It CANNOT test the VWAP, volume or Custom 1 signals, and must never be presented as validating them.
- **Track 3: Free samples and optional paid upgrade.** Use vendor free samples to validate the pipeline. Buying data is my decision, to be made only after Tracks 1 and 2 are working and only after the AI shows me what it costs and what it adds.

A forward test (recording plus paper trading) is a legitimate substitute for a historical backtest, but it needs time: expect 2 to 3 months to collect a meaningful sample.

---

## 3. Configuration (all tunable, none hard-coded)

| Parameter | Default | Notes |
| --- | --- | --- |
| timezone | America/Los\_Angeles | all schedules in Pacific |
| strike\_step | 5 | 10 allowed in Phase 3 if thin |
| min/max contracts | 1 / 5 | per trade |
| fee\_per\_contract\_per\_side | 1.12 USD | include in all P&L |
| max\_open\_positions | 1 | never a call and a put at once |
| max\_failed\_trades\_per\_day | 3 | halt the day at 3 |
| max\_daily\_loss\_pct\_equity | 2% | halt the day |
| max\_trades\_per\_day | 8 |  |
| cooldown\_after\_exit / stopout | 3 / 10 min | same contract blocked 10 min after a stop |
| max\_risk\_per\_trade\_usd | set by me | contracts x ask x 100 x stop% must not exceed |
| vwap\_edge\_pct (A) | 3% |  |
| vol\_accel | 2x |  |
| stretch\_pct (B) | 25% |  |
| quote\_stale\_sec | 3 |  |
| max\_slippage\_pct\_from\_mid | 3% | marketable limit orders only |
| mode | PAPER | LIVE needs explicit live\_enabled |

---

## 4. Architecture

Separate, testable modules:

1. **Recorder**: connects to my broker's market-data stream during market hours and stores the ladder (Section 11). Also usable as the live feed.
2. **Replay feed**: replays recorded days through the same interface as the live feed.
3. **Snapshot builder and feature engine**: time-aligned snapshots and derived metrics (Section 5). Pure functions, no lookahead.
4. **Rule engine (deterministic)**: implements Sections 6 and 7. Baseline and source of truth.
5. **LLM decision layer (optional)**: Modes 2 and 3 below.
6. **Risk harness**: validates every order against Section 8, monitors stops and trails itself on every tick, never waits on the LLM.
7. **Execution adapter**: paper first, marketable limit orders only.
8. **Logger and monitor**: append-only log of every snapshot, signal, decision, rejection, order, fill and P&L, with alerts.

**Modes (compare all three):** Mode 1 rules only (mandatory baseline); Mode 2 LLM may only confirm, veto or reduce size; Mode 3 LLM proposes the decision JSON and the harness validates it. Keep the LLM only if it beats Mode 1 on the same replayed days. Do not call an LLM inside large simulations; use the deterministic engine, and evaluate the LLM on a sampled replay of recorded snapshots.

**Hard rule:** stops, trails, time exits, halts and the flatten-at-close are executed by code. The LLM may only recommend exits that are more conservative.

---

## 5. Inputs and derived metrics

Per contract per snapshot: symbol, expiry, strike, side, bid, ask, mark, last, volume, open\_interest (prior-day), delta, theta, IV, quote\_timestamp, plus VWAP and custom1 (provided or reconstructed, see below). SPX last.

Derived: implied\_spot (strike + call\_mark - put\_mark at the nearest strike), spread\_pct = (ask-bid)/mid, vwap\_edge\_pct = mark/vwap - 1, interval\_volume and vol\_accel (vs rolling median of last 20 snapshots), custom1\_interval and custom1\_rank (among same-side ladder contracts), side\_flow\_ratio, vol\_oi\_ratio, spot\_trend (change in implied\_spot over 3 to 5 minutes).

**VWAP and Custom 1 reconstruction (important for free data):** most broker streams give cumulative day volume and last price but not an option-level VWAP. Reconstruct it as VWAP\_t = sum(last\_i x delta\_volume\_i) / sum(delta\_volume\_i), accumulated from the session open, and Custom1 = VWAP x cumulative volume. This is an approximation (last price may not equal every print; updates can be conflated). Validate it each day by comparing a few contracts against the VWAP and Custom 1 columns in my thinkorswim watchlist, and report the error. A recording that starts after the open has no valid cumulative baseline; flag that day as partial and exclude it from signal tests.

---

## 6. Entry rules and sizing

Candidates: strikes that are multiples of strike\_step within the phase's ladder (Section 7), re-centered when spot drifts one step. Liquidity gates for both setups: spread <= max($0.30, 3% of mid) and spread\_pct <= 5%; mark between $3 and $40; quote age under limit; no data fault; no event blackout.

**Setup A, momentum (all must pass):** (1) mark above VWAP by at least vwap\_edge\_pct and rising over the last 3 snapshots; (2) vol\_accel >= 2x; (3) custom1\_interval top 2 among same-side contracts and side\_flow\_ratio >= 1.5; (4) spot\_trend in the trade direction (if opposed, no trade).

**Setup B, VWAP snap-back (counter-trend, all must pass):** (1) Custom1 top 2 on its side and >= 1.5x the median same-side Custom1; (2) mark <= VWAP x (1 - stretch\_pct); (3) turn confirmed: higher lows over 3 snapshots, mark above its 5-snapshot average, interval volume >= 1.5x median, spot\_trend in trade direction for 2+ consecutive minutes (a contract still making new lows is not a setup); (4) target = mark + 0.6 x (VWAP - mark) and (target - ask)/ask >= 1.5 x active stop %; (5) max 3 contracts; 2 failed B trades halt B for the day. Near expiry a far-below-VWAP mark can simply be decay, so B must prove itself separately.

**Conflicts:** call and put both qualifying, or A and B opposite: no trade. A and B cannot fire on the same contract. No re-entry in the same contract within 10 minutes of a stop-out.

**Sizing (1 to 5):** start at 1; add 1 each for vol\_accel >= 3x, Custom1 rank #1 and rising, strong underlying confirmation, spread < 2% of mid. Cap per phase table. After any failed trade today, cap at 1. Harness check: contracts x ask x 100 x active stop % <= max\_risk\_per\_trade\_usd.

---

## 7. Exits and time-of-day tightening (harness-enforced; Pacific time)

Values only get stricter through the day.

| Phase | Window | Ladder | abs(delta) | Max contracts | Hard stop | Trail arms at | Trail distance | Time stop |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 06:40-07:30 | +/-4 strikes | 0.25-0.55 | 5 | 25% | +15% | 5% off peak | 6 min |
| 2 | 07:30-08:15 | +/-2 strikes | 0.35-0.55 | 4 | 20% | +12% | 4% off peak | 4 min |
| 3 | 08:15-09:00 | ATM +/-1 | 0.40-0.55 | 3 | 15% | +10% | 4% off peak | 3 min |

No new entries after 09:00; flat by 09:15.

- Hard stop: exit when mark <= entry\_fill x (1 - stop%).
- Trail: once mark >= entry\_fill x (1 + arm%), stop = peak\_mark x (1 - trail%), ratcheting up only. Peak is the highest option mark since entry.
- Time stop: exit if gain is under +5% after the time-stop minutes.
- Noise floor: effective stop % = max(phase stop, 4 x spread\_pct). If that exceeds 25%, or spread\_pct exceeds 5%, do not enter. Test max(trail%, 2 x spread) against fixed trails.
- Early exits the model may recommend: VWAP loss (A), volume dry-up, underlying reversal, event blackout, new low after entry (B), target hit (B).
- Never widen or remove a stop, average down, add to a loser, or hold past the flat time.
- Stops trigger on mark; fills happen at the bid minus slippage. Simulations must model this gap.

---

## 8. Risk guardrails (hard limits in the risk harness)

1. Failed trade = closed with net P&L < 0 after fees. **3 failed trades: halt for the day.**
2. Halt for the day if day P&L <= -max\_daily\_loss, or trades\_today >= max\_trades.
3. One position at a time; no averaging down, no pyramiding.
4. Cooldowns: 3 min after any exit, 10 min after a stop-out (same contract).
5. Entry window and flat time per Section 7; no overnight positions.
6. Event blackouts: no entries within 10 minutes of scheduled macro releases or 30 minutes before an FOMC decision; calendar checked daily.
7. Data faults mean no-trade (and exit if in a position): stale quote, crossed market, mark outside bid/ask, missing fields, spread over 2x its 5-minute median, feed gap over 5 s.
8. Marketable limit orders only; cancel unfilled after 3 s and re-evaluate; never chase.
9. Reconcile positions and orders with the broker every 30 s; any mismatch: flatten and halt.
10. Kill switch, manual and automatic (any unhandled exception cancels, flattens, halts).
11. Idempotent order IDs; no duplicates on retry.
12. LIVE only if live\_enabled is set by me and the Section 12 gates are marked passed.
13. Check before going live: margin account, pattern-day-trader rules and minimum equity if applicable, options approval level. Ask me; do not assume.
14. Daily report: trades, reasons, rejections, slippage vs model, P&L after fees.

---

## 9. LLM runtime prompt and output schema (Modes 2 and 3)

System prompt for the model called each cycle (fill in config values):

```
ROLE: You are the signal engine for a rules-based SPX/SPXW intraday options scalper.
You do NOT place orders. Each cycle you receive a snapshot and account/position state
and return exactly ONE JSON decision. A harness validates it against hard risk limits,
places orders and enforces stops. If it rejects a decision, accept it; never try to work
around a limit. Default is NO_TRADE; a zero-trade day is a good day. Capital
preservation beats activity.
INPUT: timestamp (Pacific), mode, SPX and implied spot, ladder rows with history (bid,
ask, mark, last, vwap, volume, open_interest, custom1, delta, theta), account state
(equity, day_pnl, trades_today, failed_trades_today, open position with entry_fill,
peak_mark, stop_level, trail_armed), event_flags, config, current phase.
RULES: apply Setups A and B, sizing, phase table, exits and guardrails exactly as in
the specification (values in config). Strikes that are multiples of strike_step only.
Long single-leg calls or puts only. One position at a time. Conflicting signals:
NO_TRADE. In a position you may only recommend EXIT or HOLD, and only more
conservatively than the harness rules. Never widen or remove a stop, average down, add
to a loser, or re-enter a contract within its cooldown. If data is stale, inconsistent
or incomplete: NO_TRADE (EXIT if in a position). If unsure: NO_TRADE.
OUTPUT: JSON only, no prose.
```

```json
{
  "decision": "NO_TRADE | ENTER | EXIT | HOLD | HALT_DAY",
  "setup": "A | B | null",
  "phase": 1,
  "contract": "<symbol or null>",
  "side": "CALL | PUT | null",
  "contracts": 1,
  "limit_price": 0.0,
  "target_price": null,
  "conviction": 1,
  "signals": {"vwap_edge_pct": 0.0, "vol_accel": 0.0, "custom1_rank": 0,
              "underlying_confirms": true, "spread_pct": 0.0},
  "watchlist_changes": {"add": [], "remove": []},
  "reason": "25 words max",
  "invalidation": "what would prove this wrong"
}
```

The harness schema-validates every reply; a malformed reply or timeout is NO\_TRADE (or HOLD with code-level stops active if already in a position).

---

## 10. Validation plan (REQUIRED before any live use)

**Goal: find out whether the rules have positive expectancy after realistic costs. The honest answer may be no.**

### Track 1: forward recording and replay (tests the real signals)

- Record every trading day from before the open to the close. Minimum 60 trading days before judging; more is better. The AI states how many trades that is likely to yield per setup, and says plainly if Setup B will not reach a statistically useful count.
- Replay recorded days through the deterministic engine exactly as live (same guardrails, latency, fills). Run paper trading in parallel on live data.
- Fills: buy at ask, sell at bid, plus slippage; stops trigger on mark and fill at bid minus slippage; fees $1.12 per contract per side. Compare paper fills against the replay fill model each week.
- No lookahead: features at time t use data at or before t only; open interest is the prior-day value.
- Split recorded days chronologically: tune on the first part, evaluate on the later untouched part. Never tune on the evaluation days.

### Track 2: proxy backtest on free underlying data (tests mechanics, not the signals)

- Data: free 1-minute index or SPY bars, daily VIX and, where available, VIX1D, a risk-free rate series. The AI must check each free source's current history limits and rate limits and tell me what depth it can really get; free intraday history is often limited to weeks or a couple of years.
- Price modeled options with Black-Scholes: spot, strike, fractional-day time to expiry, IV from VIX or VIX1D plus a configurable skew and intraday IV adjustment, dividend yield and rate as parameters. Model bid/ask spreads per phase as configurable (for example $0.20 to $0.50) and stress at 2x and 3x.
- What it can test: the stop, trail, time-stop and phase mechanics; guardrail behavior (halts, cooldowns, flat time); fee drag; and **break-even analysis** (the hit rate and win/loss ratio needed to clear costs under each phase's stop and trail).
- **Null-hypothesis test:** run the exit rules on random-walk entries (no edge) to measure the expected loss per trade just from spread, fees and exits. Any real signal must beat that hurdle.
- Proxy signals: surrogates built from the underlying (spot momentum and spot stretch from its own VWAP with volume acceleration). Label them A-proxy and B-proxy. They are **not** the strategy and say nothing about Custom 1.
- Limits to state in every report: modeled option prices ignore real IV changes, skew and microstructure, so absolute P&L from Track 2 is unreliable. Use it for go or no-go on mechanics and for sizing the cost hurdle.

### Track 3: free samples and optional paid upgrade

- Use vendor free samples (several sell SPX data and offer sample files) to test the loaders, symbol normalization and VWAP reconstruction. Treat samples as pipeline tests, not as evidence of edge.
- Optional: list paid historical intraday SPXW sources with price, history depth and fields (bid/ask, volume, trades vs snapshots), and recommend one only if it clearly adds value. Do not buy anything; ask me.

### Reporting (for every track; Setup A, Setup B and combined, separately)

Trades, win rate, average win and loss, expectancy per trade and per contract after fees, profit factor, max drawdown, worst day, percent of days halted, longest losing streak; breakdowns by phase, call vs put, 0DTE vs 1DTE, weekday, volatility regime; sensitivity of vwap\_edge\_pct, vol\_accel, stretch\_pct, stops, trails, ladder width (a result that only works at one setting is overfit); **ablation** (remove each rule one at a time); **baselines** (random entries with the same exits; ATM entries at fixed times); bootstrap confidence intervals on expectancy; Monte Carlo reshuffle for drawdown; 2x and 3x cost stress; the number of parameter combinations tried and the multiple-testing risk. Deliver code, config, data manifest, reproduction commands, and a plain-language summary of what does NOT work.

---

## 11. Instructions for the AI: free data setup

Do these in order and ask before assuming.

1. **Inventory what I have.** Ask which brokers I have access to and whether I can create a developer API app. I have a Schwab account, so first check Schwab's developer API: its streamer offers a Level 1 options service (LEVELONE\_OPTIONS) delivering changes only, with fields that, according to the documentation, include bid, ask, last, mark, volume, open interest and greeks. Read the official documentation and confirm that SPX/SPXW index options, the fields I need, and the strike ladder subscription limits are supported. Note that tokens are short-lived (access about 30 minutes) and the refresh token expires after about 7 days, so the recorder needs automatic refresh and an alert when manual re-authorization is due. Verify all of this in the current docs; do not rely on this paragraph.
2. **Secrets.** Credentials go in environment variables or a local secrets file, never in chat or in code, and never in the repository.
3. **Fallbacks if the broker API is unavailable or lacks fields:** say so and propose alternatives, for example manual end-of-day exports from thinkorswim, or a paid data feed, each with cost and limits, and wait for my decision. Do not scrape sources in ways that violate their terms.
4. **Build the recorder.** Runs on a machine that stays on from before 06:30 PT to after 13:00 PT. Subscribes to the SPX index and to a ladder of SPXW strikes (multiples of 5) for the nearest expiries, re-centering as spot moves. Because the stream sends changes only, keep the last known state and write a full snapshot every 1 to 5 seconds. Also log raw change events. Store as Parquet partitioned by date, with a manifest (row counts, gaps, start and end times) and an automatic data-quality report per day.
5. **Reconstruct VWAP and Custom 1** per Section 5, validate against my thinkorswim columns on spot-checked days, and report the error. Flag partial days.
6. **Normalize symbols** across formats (OCC style such as SPXW261103C07550000 versus thinkorswim's .SPXW261103C7550) into one internal key: root, expiry, side, strike.
7. **Timezones.** Store UTC and exchange time; convert to Pacific for the schedule; handle daylight saving explicitly.
8. **Event calendar** (macro releases, FOMC) from a free source, with a manual override file.
9. **Daily health check** that alerts me when the recorder stops, the stream gaps, the token needs re-auth, or disk is low. Estimate storage per day and per month before I start.
10. **Pilot first:** one week of recording, show me the data and the quality report, and get my approval before building the replay and strategy engines on top.

---

## 12. Promotion gates (suggested; tell me if you disagree)

1. **Mechanics gate (Track 2):** the exit and guardrail logic passes unit tests (including proof that a stop cannot be widened and a fourth failed trade cannot be placed), and the cost and break-even analysis shows a plausible path to positive expectancy.
2. **Signal gate (Track 1):** at least 60 recorded trading days and at least 300 simulated trades per setup that I intend to trade (a setup with fewer stays in paper), positive expectancy after fees and 2x cost stress on the untouched evaluation days, with a bootstrap interval that excludes zero, profit factor of at least 1.3, stable under plus or minus 20% parameter changes, and better than the random-entry baseline.
3. **Paper gate:** at least 100 paper trades on live data, realized slippage and fill rates within tolerance of the model, zero guardrail breaches, no unhandled errors.
4. **Live gate (my explicit written approval):** 1 contract only, a small loss limit and a daily manual review for several weeks; scale up only after a reviewed record.

Failing a gate means stop and report, not tune until it passes.

---

## 13. Deliverables and build order

1. Design document with architecture, data schema, sequence diagrams, position and stop/trail state machine, failure modes, test plan and open questions. **Stop and wait for approval.**
2. Recorder pilot and data-quality report. **Wait for approval.**
3. Replay engine, rule engine, and Track 2 proxy harness with results. **Wait for approval.**
4. Paper trading harness with logger and monitor, running alongside continued recording.
5. Live adapter, disabled by default, behind the Section 12 gates.

Stack: Python, Parquet and DuckDB for data, pytest for every rule and guardrail, one configuration file.

---

## 14. Model-specific notes

- **Claude, ChatGPT, Gemini:** use code execution if available; otherwise produce runnable code and exact run instructions. Do not state results you did not compute.
- Any model that proposes loosening a risk limit, changing stop logic, or skipping the testing plan must say so explicitly, explain why, and wait for my approval.

---

*End of specification. This must be tested and paper traded before any live use, and it is not financial advice.*
