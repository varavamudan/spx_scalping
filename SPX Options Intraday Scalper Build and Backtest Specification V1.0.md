# SPX Options Intraday Scalper: Build and Backtest Specification (v1.0)

For use with: Claude (Opus 5.5 or similar), Gemini, ChatGPT, Unusual Whales AI, or any capable LLM. Paste this entire document as the first message.

---

## 0. READ FIRST: mandatory notice and instructions to the AI

**This strategy is an unvalidated hypothesis. It MUST be backtested, then paper traded, before any real capital is used.** Nothing in this document shows the rules have an edge. Intraday SPX option scalping is high risk, and most such strategies lose money after costs. This is not financial advice.

Instructions to the AI reading this:

1. Do NOT write live-trading code in your first response. First produce a **design document** (architecture, data model, diagrams, test plan, open questions) and **wait for my approval**.
2. Build in this order only: data acquisition, backtest engine, paper trading, then (only after the gates in Section 12 pass and I explicitly approve) a live-trading adapter.
3. Never invent API endpoints, field names, data availability, or results. If you are unsure, say so, read the official docs, or ask me. Never fabricate backtest numbers; report only what code actually computed.
4. Treat all risk limits in Sections 7 and 8 as hard constraints enforced in code, not as suggestions to the model.
5. Ask me for anything missing (broker, data access, capital, constraints) before assuming it, in one batch of questions.
6. Point out any flaw, risk, or unrealistic assumption you find in this spec instead of silently going along with it.

---

## 1. Objective and scope

Automate an intraday SPX/SPXW options scalping process I currently run manually from a thinkorswim watchlist:

- Instrument: SPX / SPXW index options, long single-leg calls or puts only (no spreads, no short premium, no naked shorts).
- Style: quick entries and exits (minutes), 1 to 5 contracts per trade.
- Primary signals: option VWAP, volume, and "Custom 1" (= VWAP x Volume, i.e. premium traded today in index points; multiply by 100 for dollars).
- Watchlist: a ladder of strikes a few levels above and below the money, kept centered on the current index level, on the nearest expiry (0DTE or 1DTE) plus optionally the next few expiries for context.
- Trading window: 06:40 to 09:00 US Pacific (09:40 to 12:00 ET), flat by 09:15 PT. No overnight positions.

Out of scope for v1: multi-leg strategies, other underlyings, swing trades, and any tax or legal structuring.

---

## 2. Configuration (all tunable, none hard-coded)

| Parameter | Default | Notes |
| --- | --- | --- |
| timezone | America/Los\_Angeles | all schedules in Pacific |
| strike\_step | 5 | use 10 in Phase 3 if liquidity is thin; only multiples of 5 or 10 |
| min\_contracts / max\_contracts | 1 / 5 | per trade |
| fee\_per\_contract\_per\_side | 1.12 USD | include in every P&L |
| max\_open\_positions | 1 | never hold a call and a put at once |
| max\_failed\_trades\_per\_day | 3 | day halts at 3 failed trades |
| max\_daily\_loss\_pct\_equity | 2% | day halts at this loss |
| max\_trades\_per\_day | 8 |  |
| cooldown\_after\_exit | 3 min |  |
| cooldown\_after\_stopout | 10 min | same contract blocked for 10 min |
| max\_risk\_per\_trade\_usd | set by user | contracts x ask x 100 x stop% must not exceed this |
| vwap\_edge\_pct (Setup A) | 3% | mark above its VWAP by at least this |
| vol\_accel | 2x | interval volume vs rolling median |
| stretch\_pct (Setup B) | 25% | mark at least this far below VWAP |
| quote\_stale\_sec | 3 | entry blocked; in position, exit |
| max\_slippage\_pct\_from\_mid | 3% | marketable limit orders only |
| mode | PAPER | LIVE requires explicit live\_enabled flag |

---

## 3. Architecture

Components (build each as a separate, testable module):

1. **Data feed**: live quotes and trades for the SPX index and the SPXW option ladder (broker API or data vendor). In backtest, a replay feed with the same interface.
2. **Snapshot builder**: assembles a time-aligned snapshot every N seconds (default 5 s live; 1 min minimum for backtest) with the fields in Section 4 plus a short rolling history.
3. **Feature engine**: computes derived metrics (Section 4). Pure functions, no lookahead.
4. **Rule engine (deterministic)**: implements Sections 5 to 7 exactly. This is the baseline and the source of truth.
5. **LLM decision layer (optional)**: see operating modes below.
6. **Risk harness**: validates every proposed order against Section 8, places orders, and monitors stops and trails itself on every tick. It never waits on the LLM.
7. **Execution adapter**: broker interface (paper first). Marketable limit orders only.
8. **Logger and monitor**: log every snapshot, signal, decision, rejection, order, fill and P&L, with timestamps, to an append-only store. Alerts on halts, data faults, and kill switch.

**Operating modes (backtest all three and compare):**

- **Mode 1, rules only**: the deterministic rule engine trades alone. This is the mandatory baseline.
- **Mode 2, LLM veto**: the rule engine proposes; the LLM may only confirm, veto, or reduce size. It can never create a trade or enlarge size.
- **Mode 3, LLM proposes**: the LLM returns the decision JSON (Section 9); the harness validates it.

The LLM layer is only justified if it beats Mode 1 out of sample after its own cost and latency. If it does not, drop it. Do not call an LLM inside a full multi-year backtest loop; use the deterministic engine for the backtest, and evaluate the LLM on a sampled replay set of saved snapshots.

**Hard rule:** stops, trails, time exits, halts and flatten-at-close are executed by the harness in code. The LLM may only recommend exits that are more conservative than the code-level rules.

---

## 4. Inputs and derived metrics

Per contract, per snapshot: symbol, expiry, strike, side, bid, ask, mark, last, vwap, volume, open\_interest (prior-day, point-in-time), custom1, delta, theta, net\_change, quote\_timestamp.

SPX: last, bid/ask if available.

Derived:

- **implied\_spot**: from put-call parity at the nearest strike K: K + (call\_mark - put\_mark). Used to cross-check the index level.
- **spread\_pct** = (ask - bid) / mid.
- **vwap\_edge\_pct** = mark / vwap - 1.
- **interval\_volume**: volume now minus volume at the previous snapshot; **vol\_accel** = interval\_volume / rolling median interval volume (last 20 snapshots, same contract).
- **custom1\_interval**: change in Custom1 over the interval; **custom1\_rank**: rank among same-side ladder contracts.
- **side\_flow\_ratio**: same-side interval Custom1 / opposite-side interval Custom1.
- **vol\_oi\_ratio**: volume / open\_interest (fresh activity indicator).
- **spot\_trend**: change in implied\_spot over the last 3 to 5 minutes.

VWAP and Custom1 are cumulative from the session open. In backtest, reconstruct them from trade-level or minute-level data with no lookahead (see Section 10).

---

## 5. Entry rules

Candidate set: strikes that are multiples of strike\_step, within a ladder around implied\_spot, width and delta band set by time-of-day phase (Section 7). Re-center the ladder whenever spot drifts one strike step from center.

Common liquidity gates (both setups): spread <= max($0.30, 3% of mid) and spread\_pct <= 5%; mark between $3 and $40; quote age under quote\_stale\_sec; no data faults; not inside an event blackout.

### Setup A: Momentum

All must pass:

1. mark above VWAP by at least vwap\_edge\_pct and rising over the last 3 snapshots.
2. vol\_accel >= 2x.
3. custom1\_interval ranks top 2 among same-side contracts and side\_flow\_ratio >= 1.5.
4. Underlying confirms: spot\_trend is in the trade direction. If it opposes the trade, no trade.

### Setup B: VWAP snap-back (counter-trend, higher risk)

Idea: heavy premium has been paid into a contract, the mark has fallen far below its VWAP, and it is now turning back up after confirmation. All must pass:

1. Crowding: Custom1 is top 2 on its side and >= 1.5x the median Custom1 of same-side ladder contracts.
2. Stretch: mark <= VWAP x (1 - stretch\_pct).
3. Turn confirmation: higher lows over the last 3 snapshots; mark above its 5-snapshot average; interval volume >= 1.5x median; spot\_trend in trade direction for at least 2 consecutive minutes. A contract still making new lows is not a setup.
4. Reward vs risk: target = mark + 0.6 x (VWAP - mark); require (target - ask) / ask >= 1.5 x active stop %.
5. Max 3 contracts. Two failed B trades halt Setup B for the day.

Note: near expiry, a far-below-VWAP mark can simply mean decay. Check in the backtest that Setup B has positive expectancy by itself.

### Conflicts

- Call-side and put-side both qualifying, or A and B in opposite directions: no trade.
- Setup A and B cannot fire on the same contract.
- Do not re-enter the same contract within 10 minutes of a stop-out.

---

## 6. Position sizing (1 to 5 contracts)

Start at 1. Add 1 for each of: vol\_accel >= 3x; Custom1 rank #1 and rising; strong underlying confirmation; spread < 2% of mid. Cap at 5 (Setup B cap 3, phase caps below). After any failed trade today, cap at 1 contract. Harness check: contracts x ask x 100 x active stop % <= max\_risk\_per\_trade\_usd.

---

## 7. Exits and time-of-day tightening (harness-enforced)

All times Pacific. Values only get stricter through the day, never looser.

| Phase | Window | Ladder | abs(delta) | Max contracts | Hard stop | Trail arms at | Trail distance | Time stop |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 06:40-07:30 | +/-4 strikes | 0.25-0.55 | 5 | 25% | +15% | 5% off peak | 6 min |
| 2 | 07:30-08:15 | +/-2 strikes | 0.35-0.55 | 4 | 20% | +12% | 4% off peak | 4 min |
| 3 | 08:15-09:00 | ATM +/-1 | 0.40-0.55 | 3 | 15% | +10% | 4% off peak | 3 min |

No new entries after 09:00. All positions flat by 09:15.

**Definitions**

- Hard stop: exit when mark <= entry\_fill x (1 - stop%).
- Trailing stop: once mark >= entry\_fill x (1 + arm%), the trail is armed. Stop = peak\_mark x (1 - trail%). It only ratchets up, never down. "Peak" is the highest option mark since entry.
- Time stop: exit if gain is under +5% after the time-stop minutes.
- Noise floor: effective stop % = max(phase stop, 4 x spread\_pct). If that exceeds 25%, or spread\_pct exceeds 5%, do not enter. A trail tighter than about 2 x the spread will stop itself out on quote noise; the backtest should compare max(trail%, 2 x spread) against fixed trails.
- Early exits the model may recommend: VWAP loss (Setup A), volume dry-up, underlying reversal, event blackout, new low after entry (Setup B), target hit (Setup B).
- Never: widen or remove a stop, average down, add to a loser, or hold through the flat time.
- Stops trigger on mark; fills happen at the bid minus slippage. Backtest must model this gap.

---

## 8. Risk guardrails (hard limits in the risk harness)

1. Failed trade = closed with net P&L < 0 after fees. **3 failed trades: halt for the day.**
2. Halt for the day if day P&L <= -max\_daily\_loss, or trades\_today >= max\_trades.
3. One position at a time; no averaging down; no pyramiding.
4. Cooldowns: 3 min after any exit, 10 min after a stop-out (same contract).
5. Entry window and flat time as in Section 7. No overnight positions.
6. Event blackouts: no entries within 10 minutes of scheduled macro releases and 30 minutes before an FOMC decision. Calendar provided by a data source, checked daily.
7. Data faults trigger no-trade (and exit if in a position): stale quote, crossed market, mark outside bid/ask, missing fields, spread over 2x its 5-minute median, feed disconnect over 5 s.
8. Orders: marketable limit only, max slippage per config. No market orders. Cancel unfilled orders after 3 s and re-evaluate; never chase.
9. Position and order reconciliation with the broker every 30 s. Any mismatch: flatten and halt.
10. Kill switch: a manual flag and an automatic one (any unhandled exception) that cancels working orders, flattens, and halts.
11. Idempotent order IDs; no duplicate orders on retry.
12. LIVE only if live\_enabled is set by me, the account is verified, and the backtest and paper gates in Section 12 are marked passed.
13. Account constraints to check before going live: margin account, pattern day trader rules and minimum equity if applicable, broker limits on options approvals. Ask me; do not assume.
14. Daily report: trades, reasons, rejections, slippage vs model, P&L after fees.

---

## 9. LLM runtime prompt and output schema (Modes 2 and 3)

Use this as the system prompt for the model that is called each cycle. Fill in the config values.

```
ROLE: You are the signal engine for a rules-based SPX/SPXW intraday options scalper.
You do NOT place orders. Each cycle you receive a market snapshot and account/position
state and return exactly ONE JSON decision. A separate harness validates your decision
against hard risk limits, places orders, and enforces stops. If the harness rejects a
decision, accept it. Never try to work around a limit. Default is NO_TRADE; a zero-trade
day is a good day if nothing qualifies. Capital preservation beats activity.

INPUT: timestamp (Pacific), mode, SPX and implied spot, ladder rows with history
(bid, ask, mark, last, vwap, volume, open_interest, custom1, delta, theta, net_change),
account (equity, day_pnl, trades_today, failed_trades_today, open_position with
entry_fill, peak_mark, stop_level, trail_armed), event_flags, config, current phase.

RULES: apply Setup A and Setup B, sizing, phase table, exits and guardrails exactly as
defined in the specification sections 5 to 8 (values supplied in config).
Only strikes that are multiples of strike_step. Long single-leg calls or puts only.
One position at a time. Conflicting signals: NO_TRADE. In a position you may only
recommend EXIT or HOLD, and exits only more conservatively than the harness rules.
You may never widen or remove a stop, average down, add to a loser, or re-enter the
same contract within the cooldown.
If data looks stale, inconsistent, or incomplete: NO_TRADE (EXIT if in a position).
If you are unsure: NO_TRADE.

OUTPUT: JSON only, no prose.
```

Output schema:

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
  "signals": {
    "vwap_edge_pct": 0.0,
    "vol_accel": 0.0,
    "custom1_rank": 0,
    "underlying_confirms": true,
    "spread_pct": 0.0
  },
  "watchlist_changes": { "add": [], "remove": [] },
  "reason": "25 words max",
  "invalidation": "what would prove this wrong"
}
```

The harness must schema-validate the output, reject anything malformed, and treat a timeout or malformed reply as NO\_TRADE (and as "HOLD with code-level stops active" if already in a position).

---

## 10. Backtest specification (REQUIRED before anything else)

**Goal: find out whether the rules have a positive expectancy after realistic costs, out of sample. The honest result may be that they do not.**

**Data and period**

- Target 3 to 5 years of intraday SPX and SPXW data. Daily (Mon-Fri) SPXW expirations only exist since roughly 2022, so verify what is actually available and say so. Short history means regime risk (one market environment); report this limitation.
- Needed per contract per time step: bid, ask, last, volume, trade-level prints (preferred) or minute bars, open interest (point-in-time), plus the SPX index level and an event calendar. Include expired and delisted contracts (no survivorship bias).
- Fidelity tiers: A) quotes and trades at tick level (best); B) 1-minute bars with bid/ask; C) trades only (add conservative slippage and disclose). Say which tier you have; results from Tier C must be labeled lower confidence.

**Engine rules**

- No lookahead: features at time t use only data at or before t. VWAP and Custom1 are rebuilt cumulatively from the session open. Open interest is the prior-day value.
- Signal at time t, order at t plus latency (default 2 s live-equivalent; for 1-minute bars, fill at the next bar's open or worse).
- Fills: buy at ask, sell at bid, plus slippage (default 1 tick or per config). Stops trigger on mark but fill at bid minus slippage; model gaps through stops. Apply the $1.12 per contract per side.
- Apply every guardrail in Section 8 exactly as live (halts, cooldowns, blackouts, flat time).

**Reporting (separate for Setup A and Setup B, and combined)**

- Trades, win rate, average win and loss, expectancy per trade and per contract after fees, profit factor, total and per-day P&L, max drawdown, Sharpe/Sortino, worst day, percent of days halted, longest losing streak.
- Breakdowns: by phase, call vs put, 0DTE vs 1DTE, weekday, volatility regime (VIX bucket), and event days.
- Sensitivity heatmaps for vwap\_edge\_pct, vol\_accel, stretch\_pct, stop %, trail %, ladder width. A strategy that only works at one exact setting is overfit.
- **Ablation study**: remove or alter each rule one at a time (VWAP filter, volume accel, Custom1 rank, underlying confirmation, time-of-day tightening, trail) and report the effect.
- **Baselines**: random entries with the same exits and sizing; simple ATM buy at fixed times with the same exits. The strategy must beat these.
- **Validation**: walk-forward (e.g. train 12 months, test 3 months, roll forward) and a final untouched holdout period. Report confidence intervals (bootstrap) on expectancy. Monte Carlo reshuffle of trade order for drawdown. State the number of parameter combinations tried and discuss multiple-testing risk.
- Slippage stress test: re-run with 2x and 3x assumed slippage and fees; a strategy that dies at 2x costs is fragile.
- Deliver all code, config, data manifest, the exact commands to reproduce, and a plain-language summary that includes what does NOT work.

**LLM evaluation (Modes 2 and 3)**: replay a stratified sample of saved snapshots; compare against Mode 1 on the same samples; report agreement, vetoed trades, and the P&L difference of the vetoed trades.

---

## 11. Instructions for the AI: getting historical data from Unusual Whales for the backtest

I have an Unusual Whales account and want it to be the primary history source. Do the following in order. Do not guess endpoints; read the official documentation.

1. **Confirm access.** Ask me which Unusual Whales plan I have and whether I have API access. The web app subscription and the API may be separate products. If I do not have API access, tell me what to purchase or request (API plan, or any historical data download product) before continuing.
2. **Read the current docs and OpenAPI spec** at https://api.unusualwhales.com/docs (and any linked spec) and list the endpoints that cover: option contract listings for SPX/SPXW including expired contracts; intraday (minute-level or tick) data per contract; historical trades or flow; open interest and volume history; greeks; index price history. For each, record the historical depth, time resolution, rate limits and pagination, and whether bid/ask quotes are included. Report any field you need but they lack.
3. **Secrets.** I will store the token in an environment variable (for example UW\_API\_TOKEN). Never ask me to paste it into chat or hard-code it. Read it from the environment.
4. **Fidelity decision.** Tell me which data tier (Section 10) UW can provide. If UW lacks the depth or quotes (for example no bid/ask or under 3 years of intraday history), propose alternatives and costs before buying anything: for example Databento (OPRA), ThetaData, Polygon, CBOE DataShop, or OptionMetrics. Free sources can be used for the SPX index level and the event calendar.
5. **Build a resumable downloader.** Rate-limit aware with retries and backoff; idempotent; progress and resume state; caches raw responses; writes partitioned Parquet files by trade date, expiry and root (SPXW or SPX); stores a manifest with row counts, checksums and download time. Start with a one-week pilot, show me the schema and size estimates, then scale up.
6. **Normalize symbols.** Option IDs come in different formats (OCC style such as SPXW261103C07550000 versus thinkorswim's .SPXW261103C7550). Map them to one internal key: root, expiry, side, strike.
7. **Reconstruct and validate the signals.** Rebuild each contract's cumulative VWAP and Custom1 (= VWAP x volume) from the downloaded trades, with no lookahead. Compare against the provider's own volume and VWAP fields where available, and against my thinkorswim snapshots if I supply some. Report discrepancies.
8. **Data quality report** before any backtest: missing minutes, zero-volume gaps, crossed or locked quotes, outlier prints, early-close days, holidays, missing expirations, and expired contracts that are absent. Flag days to exclude, with reasons.
9. **Timezone.** Store timestamps in UTC with the original exchange time; convert to Pacific for the strategy schedule; handle daylight saving transitions explicitly.
10. **Hand off** to the backtest engine only after I approve the data quality report.

---

## 12. Promotion gates (suggested thresholds, tell me if you disagree)

1. **Backtest gate**: out-of-sample positive expectancy after fees and 2x slippage stress, with a bootstrap confidence interval that excludes zero; at least 300 out-of-sample trades per setup; profit factor of at least 1.3; max drawdown within my stated tolerance; results stable under plus or minus 20% parameter changes; beats the random-entry baseline.
2. **Paper gate**: at least 4 to 8 weeks and 100 or more paper trades via the broker's paper environment or live quotes; realized slippage and fill rates within tolerance of the backtest model; zero guardrail breaches; no unhandled errors.
3. **Live gate** (my explicit written approval required): start with 1 contract only, a small loss limit, and manual daily review for several weeks. Scale up only after a reviewed track record.

Failing a gate means stop and report, not tune until it passes.

---

## 13. Deliverables and build order

1. Design document: architecture, data schema, sequence diagrams (signal to order to exit), state machine for position and stop/trail logic, failure modes and handling, test plan, and your open questions. **Stop and wait for approval.**
2. Data acquisition (Section 11) and the data quality report. **Wait for approval.**
3. Backtest engine and results (Section 10). **Wait for approval.**
4. Paper trading harness with the full logger and monitor.
5. Live adapter, disabled by default, behind the gates in Section 12.

Stack preferences: Python, Parquet/DuckDB for data, pytest with unit tests for every rule and guardrail (including tests that prove a stop cannot be widened and a fourth failed trade cannot be placed). Keep configuration in one file.

---

## 14. Model-specific notes

- **Claude, ChatGPT, Gemini**: can follow this spec for design, code and analysis. Use their code execution tools if available for the backtest; otherwise produce runnable code and tell me exactly how to run it. Do not state backtest results you did not compute.
- **Unusual Whales AI**: use it for exploring flow and data availability and for checking Section 11. It will not place orders; the execution harness does that.
- If any model proposes loosening a risk limit, changing the stop logic, or skipping the backtest, it must say so explicitly and explain why, and wait for my approval.

---

*End of specification. Reminder: this must be backtested and paper traded before any live use, and it is not financial advice.*
