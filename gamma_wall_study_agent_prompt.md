# Agent Task: Historical Gamma Wall Study for SPX 0DTE Credit Spreads

## Mission

Build a research project, around my **locally stored historical market data**, that answers one question before any trading system is built:

> **Do gamma walls (call wall / put wall) act as reliable intraday support and resistance for SPX, strongly enough to base a 0DTE SPX credit-spread strategy on?**

This engagement has exactly two outcomes I care about right now:

1. **A data-availability verdict** — is the locally stored data sufficient to compute historical gamma walls at all, and over what coverage?
2. **An empirical verdict** — when gamma walls *can* be computed, does spot SPX measurably respect those levels, better than arbitrary reference levels?

This is a **measurement / validation** study. Do **not** build the credit-spread backtester yet — that is a later phase and is explicitly out of scope. The goal now is to prove (or kill) the premise cheaply.

---

## Background you need (read before coding)

**What a gamma wall is.** For each option strike:

```
GEX(strike) = gamma * open_interest * 100 * spot_price * dealer_sign
```

- **Call Wall** = the strike (typically above spot) with the largest call gamma exposure. Theory: dealer hedging concentrates here, creating *resistance*.
- **Put Wall** = the strike (typically below spot) with the largest put gamma exposure. Theory: acts as *support*.
- **Zero Gamma / Gamma Flip** = the spot level where net GEX crosses zero. Above it dealers are typically long gamma (price-dampening); below it short gamma (price-accelerating).

The `dealer_sign` convention is an assumption and must be **explicit and configurable**. Start with the common SpotGamma-style convention (treat OI as dealer-short / a fixed call-vs-put sign); note in the report that a signed-flow dealer-positioning model (classifying trades buy/sell from the bid/ask) is a possible later refinement since tick data is available — but do NOT build that now.

**0DTE SPX specifics that you must get right:**

- 0DTE SPX options are the **SPXW** (weekly/daily) series, **PM cash-settled at the 4:00 PM ET cash close**. Expiration value = SPX settlement, not an AM print. Do not model them as AM-settled.
- **Open interest is published only once per day** (overnight, by OCC). It is therefore **frozen intraday**. The gamma wall evolves *during the day only because gamma and spot move* (time decay + price), not because OI changes. Using same-day-updated OI would be look-ahead cheating.
- Liquidity is concentrated near the money; far strikes have garbage quotes.

**The premise being tested is contrarian-sensitive:** a "bounce rate" at a wall is meaningless without a baseline. You MUST compare wall behavior against control levels (see Phase 3), or the study is worthless.

---

## Inputs to confirm (I will fill these in / you must discover)

- `DATA_ROOT`: `[FILL IN — path(s) to the locally stored tick and minute data]`
- Believed coverage: `[FILL IN — e.g. "SPX + SPXW options, 2022-present"]`
- Study window for this engagement: `[FILL IN — e.g. last 12 months of trading days]`

If any of these are blank, **discover them** in Phase 1 and report what you find. Do not assume a schema or a path.

---

## Existing assets to check and reuse (do not duplicate)

Before writing pricing or GEX code, check the workspace for code you can reuse:

- An existing **GEX calculator** (from my Tastytrade GEX dashboard) implements `GEX = gamma * OI * 100 * spot`, SpotGamma-style call/put walls (max call-GEX strike / max put-GEX strike), and a zero-gamma interpolation. Reuse its logic/conventions rather than reinventing them.
- A repo named **`Triton_Optimizer`** may be present in the workspace. If it exists, inspect it for a reusable backtest/event engine, a parameter-optimization harness, a market-data adapter/schema, and any options-pricing/greeks code. Conform to its data schemas and reuse its engine **if and only if** it is actually present and fits; otherwise note its absence and proceed standalone. Do not depend on a repo you cannot see.

Report what you found and what you chose to reuse.

---

## Phase 1 — Data Availability Audit  **(HARD GATE)**

Do this first and do not proceed to Phase 2 until it passes. Produce `DATA_AVAILABILITY_REPORT.md`.

**Inventory the local data.** For both the tick store and the minute-processed store, determine:

- Storage format(s), file layout, total size, and the most efficient way to read it (parquet/duckdb/etc.).
- Date range and which trading days are actually present (flag gaps, holidays, half-days).
- Timestamp granularity and **timezone** (normalize everything to US/Eastern; account for DST and 4:00 PM ET close).

**Check for the specific fields gamma walls require.** Build a coverage matrix (fields × date-range × completeness %):

1. **Underlying SPX spot** intraday (tick and/or minute). — Almost certainly present.
2. **SPXW 0DTE option chains** per day: the set of call/put strikes for that day's same-day expiry.
3. Per-option fields:
   - **Open interest** — *the make-or-break field.* Tick/quote data usually does NOT contain OI. Explicitly determine whether OI is present, at what cadence, and for which days. **If OI is absent, gamma walls cannot be computed from local data alone** — say so plainly and propose sourcing start-of-day OI from ThetaData (I have a subscription) as the fix.
   - **Gamma / greeks** OR **implied volatility**. If greeks are present, use them. If only IV is present, compute gamma via Black-Scholes. If neither, you must compute IV from option mid prices + spot + time-to-expiry + rate + dividend/forward — confirm the inputs needed are available.
   - Quotes (bid/ask/size) and trades (price/size) — needed later for fills and for any signed-flow work; note presence.

**Deliver the verdict.** `DATA_AVAILABILITY_REPORT.md` must end with a clear statement:

- **GO** — list exactly which fields/days support gamma wall computation, the usable date range, and any limitations; then proceed to Phase 2.
- **NO-GO / PARTIAL** — name the missing field(s) (most likely OI and/or IV inputs), the exact gap, and the minimal external fetch (e.g., ThetaData OI snapshots) that would unblock it. **Stop and surface this for my review before doing further work** — do not fabricate, interpolate, or silently work around missing OI.

---

## Phase 2 — Historical Gamma Wall Computation

Once Phase 1 is GO:

For each usable trading day, on an intraday timestamp grid (start with 1- or 5-minute bars; make it configurable):

1. Load **start-of-day OI** per strike (frozen for the session — never use later-day OI).
2. Obtain or compute **gamma per strike** at the current spot and time-to-expiry. Gamma must update through the day; do not freeze it.
3. Compute `GEX(strike)` with an explicit, configurable contract multiplier (100) and `dealer_sign` convention.
4. Derive, per timestamp: **call wall**, **put wall**, the full **net-GEX-by-strike profile**, and the **zero-gamma flip** level.
5. Persist a tidy per-day / per-timestamp table of wall levels and the underlying profile (parquet recommended).

**Assumptions to log explicitly:** risk-free rate source, dividend yield / forward handling for SPX, multiplier, dealer-sign convention, the strike universe / liquidity filter, and the timestamp grid. Sanity-check output (walls should sit at high-OI strikes; spot should be on the expected side of the flip).

---

## Phase 3 — Wall-Respect Analysis

This is the core deliverable. For the computed walls, measure whether spot respects them. Make all thresholds configurable and test sensitivity to them.

**Define events quantitatively (document the exact definitions):**

- **Touch**: spot comes within tolerance `T` of a wall level (test fixed-point and ATR-relative tolerances).
- **Bounce / respect**: after a touch, price reverses away by at least `R` within `M` minutes without breaching beyond a buffer `B`.
- **Break**: price trades through the level by more than `B` and holds.
- Record **reaction magnitude** and **max excursion beyond the level**.

**Core metrics**, computed separately for call walls (resistance) and put walls (support):

- Touch rate, bounce-given-touch rate, break rate.
- Distribution of the day's high vs the call wall, and the day's low vs the put wall (do daily extremes cluster at walls?).
- Where settlement closes relative to the walls.

**Baseline / control (mandatory — the study is invalid without it).** Repeat the same touch/bounce measurement on control levels and compare:
- random strikes,
- fixed offsets from the open,
- round-number levels.
Walls must beat these controls to be considered meaningful. Report effect sizes, sample counts, and confidence intervals — not just point estimates.

**Segment the results** by: VIX / volatility regime, wall distance from the open, wall "strength" (gamma magnitude), time of day, and which side of the gamma flip the day is on (positive vs negative net gamma).

**Bridge to 0DTE credit spreads (analytical, not a backtest):** translate findings into spread-relevant statements, e.g., "on N% of days the put wall held as support and SPX never settled more than X points below it" — quantifying how often a put credit spread placed below the put wall (or a call credit spread above the call wall) would have expired OTM. Keep this descriptive; do not build the execution/fill backtester.

**Conclude with a GO / NO-GO / CONDITIONAL recommendation** on whether gamma walls are a reliable enough base for a 0DTE SPX credit-spread strategy, with the conditions and caveats.

---

## Correctness requirements / guardrails (non-negotiable)

- **No look-ahead bias.** Use only information available at decision time. OI = the previously published (overnight) value. Never use the day's settlement, close, or any future bar when computing intraday walls or signals.
- **0DTE SPXW = PM cash-settled at the 4:00 PM ET close.** Expiry value = SPX settlement.
- **OI is frozen intraday.** Gamma/spot move; OI does not.
- Normalize all timestamps to **US/Eastern**; use an exchange calendar for holidays and half-days.
- Apply **liquidity filters**; discard zero/garbage quotes and absurd IVs.
- **Never fabricate or silently interpolate missing data** (especially OI). Flag gaps and stop if a required field is missing.
- Be **reproducible and config-driven**: parameters in a config file, assumptions logged, results regenerable from raw data.
- **Performance:** tick data is large. Use columnar storage (parquet) / duckdb, vectorize, and process day-by-day rather than loading everything into memory.

---

## Suggested stack & layout (adapt as needed)

- Python 3.11+, `pandas`/`numpy`/`scipy`, `pyarrow`/`duckdb`, `matplotlib` or `plotly`, an exchange calendar (`pandas_market_calendars`/`exchange_calendars`), a Black-Scholes/greeks lib (`py_vollib`) or the existing GEX code, `pytest`.

```
gamma_wall_study/
  config/                 # study window, thresholds, rate/div assumptions
  data_audit/             # Phase 1 inventory + coverage matrix
  walls/                  # Phase 2 GEX + wall computation (reuse existing GEX logic)
  analysis/               # Phase 3 respect metrics, controls, segmentation
  reports/
    DATA_AVAILABILITY_REPORT.md
    WALL_RESPECT_FINDINGS.md
  tests/
```

---

## Working protocol

1. **Phase 1 is a hard gate.** Produce `DATA_AVAILABILITY_REPORT.md`. If data is insufficient for gamma walls, **stop and surface it** with the specific gap and the minimal fix — do not proceed.
2. When a definition or assumption is ambiguous (tolerances, sign convention, rate/dividend, study window), **ask** rather than guessing silently — or pick a sensible default, run with it, and clearly flag the choice and its sensitivity.
3. Reuse existing assets (GEX calculator, `Triton_Optimizer` engine) where they fit; report what you reused.
4. Keep findings honest: report sample sizes, baselines, and uncertainty. A clean NO-GO is a successful outcome.

## Definition of done (this engagement)

- `DATA_AVAILABILITY_REPORT.md` with a clear GO/NO-GO/PARTIAL data verdict and coverage matrix.
- A reproducible per-day/per-timestamp gamma-wall dataset (only if Phase 1 is GO).
- `WALL_RESPECT_FINDINGS.md` with the respect metrics, control-baseline comparison, segmentation, charts, and a GO/NO-GO/CONDITIONAL recommendation on using gamma walls for 0DTE SPX credit spreads.
- The credit-spread execution backtester is explicitly **deferred** and not built in this engagement.
