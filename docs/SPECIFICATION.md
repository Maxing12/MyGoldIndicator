# XAUUSD SMC + ICT + Volume Indicator: Technical Specification (Phase 1)

Target: TradingView, Pine Script v6, overlay indicator, XAUUSD, intraday (1H bias, 15M setup, 5M refinement, optional 4H context).

This file fixes the definitions, contradictions, architecture, data flow and object-management strategy *before* code. `XAUUSD_SMC_ICT_Volume.pine` implements it.

---

## 1. Contradictions / ambiguities found in the brief and how they are resolved

| # | Issue | Resolution |
|---|-------|-----------|
| 1 | Score table sums to 15 (incl. 2 volume rows) but §36 says volume is a separate capped +3, and examples show "/14". | Non-volume factors total **14**. Volume contributes a separate capped sub-score (default cap **3**). Max score = 14 + volume cap (14 when volume is OFF or unavailable). The two volume rows of the table are replaced by the §36 volume sub-score. Dashboard shows `score/max`. |
| 2 | "Retracing 5M" on a 15M chart: lower-timeframe data cannot be requested reliably with `request.security` without `request.security_lower_tf` arrays. | Rows for a timeframe **lower than the chart** show `n/a`. Run the script on 5M to see 15M/1H/4H plus the native 5M structure. |
| 3 | "No look-ahead" vs pivot-based swings that need right-hand bars. | All swings are **confirmed pivots** (`ta.pivothigh/low(len,len)`); a swing of length L is only known L bars after it forms. Structure breaks are evaluated on **closed bars only**. Developing information is shown only as a text row in the dashboard when `Show Developing Signals` is ON (default OFF); it never creates objects. |
| 4 | Objects (arrays, drawings) are **not rolled back** on realtime ticks in Pine. | The whole state-mutating engine runs only when `barstate.isconfirmed`. Historical and realtime therefore execute identically, once per closed bar. |
| 5 | Signal timing vs entry: "A+ LONG" with an *entry* zone implies the signal precedes the fill. | A setup is **published when structure + zone are complete** (status `PENDING`), and becomes `ENTERED` only when price later retraces to the zone reference (CE). Expiry (default 10 bars) applies to `PENDING`. |
| 6 | Historical analytics need future bars to know outcomes. | Signal generation never reads bars after the signal. Outcomes are resolved bar by bar *after* the signal (`Post-Signal Outcome Analysis`), separately counted and labelled. Same-bar TP/SL ambiguity is resolved pessimistically (SL first). |
| 7 | "Minor/major/internal" swings, and MSS being stricter than CHoCH. | Major = trend / dealing range / significant liquidity. **Minor = execution structure** (BOS/CHoCH/MSS that drive setups). Internal = small pivots (optional display, volume divergence). MSS = minor-structure break **plus** a prior sweep **plus** displacement (see §2). |
| 8 | "Sessions/timezone: do not hard-code broker timezone" vs daily/weekly levels. | Sessions/kill zones use an IANA timezone input (default `America/New_York`). PDH/PDL/PW/PM use TradingView's `timeframe.change("D"/"W"/"M")` i.e. the symbol's own daily boundary (for most FX/gold feeds 17:00 New York), which is how the chart's daily candle is defined. |
| 9 | Economic-calendar news is not available in Pine. | Only a **manual** news-risk window (two user-entered time ranges). Nothing is predicted. |
| 10 | XAUUSD volume is usually **tick volume**. | Labelled "Volume (tick)". Never used as a standalone or directional signal; capped contribution; can be OFF. If the feed has no volume, all volume logic switches itself off and the dashboard says `N/A`. |
| 11 | "Probability" wording. | Only "Confluence Score"; grades A+/A/B are technical-alignment labels, not probabilities. |
| 12 | Compile/test each phase. | No TradingView compiler is available in the authoring environment. The code was written phase-by-phase in module order and statically reviewed, but **must be compiled once in the Pine Editor** (see README "Verification status"). |

---

## 2. Definitions (mathematical, no discretion)

Let `L` = pivot length, `ATR` = `ta.atr(atrLen)` of the *previous* bar (`ATR[1]`) so a bar cannot inflate its own threshold.

* **Swing High (length L)**: bar `b` whose `high` is strictly greater than the highs of the `L` bars on each side (`ta.pivothigh(high, L, L)`). Confirmed at bar `b+L`. **Swing Low** is symmetric.
* **Major structure**: swings with `L = majorLen` (Balanced 10). **Minor structure**: `L = minorLen` (5), used for BOS/CHoCH/MSS and setups. **Internal**: `L = internalLen` (3).
* **Trend state** (per structure level): `+1` after the last break was upward, `-1` after downward, `0` before any break.
* **BOS ↑**: a confirmed-pivot high that is still unbroken is exceeded by `close` (or by `high` if wick mode) while trend ≠ `-1`. **BOS ↓** symmetric. Each pivot can be broken once.
* **CHoCH ↑**: same break while trend = `-1` (break *against* the prevailing structure). **CHoCH ↓** symmetric.
* **Liquidity (pool)**: an un-taken price level where stops are assumed to rest: confirmed swing highs/lows (major weight 2, minor weight 1), equal highs/lows (weight 2), previous day/week/month high/low (weight 3), completed Asian/London/NY session high/low (weight 2).
* **Equal High / Equal Low**: a new minor pivot within `tolerance` of an existing un-taken pool of the same side. Tolerance = `k·ATR`, or `k%` of price, or `k` ticks (user choice). The pool is merged and upgraded to `EQH`/`EQL`.
* **BSL / SSL**: buy-side liquidity = pools above price (highs); sell-side = pools below (lows).
* **Liquidity sweep (raid)**: on a closed bar `high > pool + pen·ATR` (BSL) and the bar **closes back below** the pool (SSL mirrored). Optional relaxed mode accepts an upper-wick share ≥ 50% of the range without close-back. Class: Minor (weight 1), Significant (2), Major (3) = weight of the most important pool taken. A pool that is exceeded *without* rejection is simply removed (a breakout, not a sweep).
* **Displacement (bullish)**: body ≥ `dispBody·ATR`, range ≥ `1.2·avg range(20)`, body/range ≥ 0.6, close in the top 25% of the range, `close > close[1]`. Levels: **1 Normal**; **2 Volume-confirmed** (RVOL ≥ high threshold); **3 Strong** (body ≥ 1.6·ATR, close in top 15%, and volume-confirmed when volume is active). Bearish is the mirror.
* **MSS ↑ (Market Structure Shift)**: a minor-structure break upward that occurs while a **SSL sweep is armed** (occurred ≤ `sweepWindow` bars earlier) **and** a bullish displacement occurred within the last `dispWindow` bars (including the break bar). Conservative preset additionally requires the break to be a CHoCH. MSS ↓ mirrors with a BSL sweep and bearish displacement. Breaks without those conditions stay plain BOS/CHoCH.
* **FVG**: bullish when `low[0] > high[2]` (bearish `high[0] < low[2]`) and gap ≥ `fvgMin·ATR`. Stores top, bottom, CE (midpoint), direction, bar, state, strength.
  * States: **Fresh** (untouched), **Partial** (touched, mitigation threshold not reached), **Mitigated** (threshold by method: first touch / 50% / full fill; default 50%), **Invalidated** (close beyond the far side).
  * Strength: Weak / **Volume Confirmed** (middle-candle RVOL ≥ high threshold) / **Strong** (≥ 3 of: displacement, structure break, volume, strong close, with volume confirmed).
* **Order Block (bullish)**: the last *meaningful* bearish candle (body/range ≥ 0.3, range ≥ 0.3·ATR) within `obLookback` bars before a bullish displacement candle that also (a) broke minor/major structure or (b) created an FVG. Zone = that candle's high–low. Strength 1–5 = displacement + structure break + FVG + volume + strong displacement; must reach the strength filter. Bearish is the mirror.
  * States: **Fresh**, **Tested** (touched), **Mitigated** (threshold by method: first touch / 50% / full / close-through), **Invalidated** (close through the far side).
* **Breaker Block**: an Order Block that becomes *Invalidated* becomes a zone of the opposite direction (bullish OB failed → bearish breaker). Display/alert only; not used as setup entry zone.
* **Dealing range**: latest confirmed major swing high and low. **Premium** = above 50%, **Equilibrium** = 50% ± tolerance, **Discount** = below. For a *setup*, the range is `[min(major low, sweep extreme), max(major high, leg high)]`.
* **OTE**: retracement of the setup *leg* (sweep extreme → post-MSS extreme) between 62% and 79% (configurable). Counted when the entry zone overlaps the band.
* **RVOL** = `volume / SMA(volume, N)[1]` (average of prior bars). **Volume spike**: `volume > SMA[1] · spikeMult` (default 1.5). **Volume confirmation**: RVOL ≥ high threshold (default 1.5). Extreme: RVOL ≥ 3.
* **Potential exhaustion**: RVOL ≥ extreme, range ≥ 2·ATR, rejection wick ≥ 50% of range, on a sweep bar. Label only; never flips bias.
* **Setup**: a record created once a liquidity sweep → MSS/CHoCH → displacement → FVG/OB chain is complete, the score/grade thresholds, RR filter and risk-width limits pass. Has entry zone, SL, TP1–3, RR, expiry, invalidation.
* **Invalidation** (before entry): close beyond SL (long: below) or beyond the entry-zone far side, or replaced by an opposite setup. **Expiry**: `N` bars without a fill (default 10).
* **A Setup** / **A+ Setup**: score ≥ `minA` / ≥ `minA+` (Balanced 7 / 9), all hard gates passed. Under STRICT volume mode, A/A+ additionally require RVOL ≥ high threshold on the displacement; otherwise the grade is capped at B. These are **technical-alignment grades, not probabilities.**

### Confluence score (max 14 + volume cap)

| Factor | Points |
|---|---|
| HTF alignment (1H aligned = 2; 1H neutral + 4H aligned = 1) | 0–2 |
| Liquidity sweep (minor 1, significant/major 2) | 0–2 |
| Structure (MSS = 2, CHoCH/BOS-with-sweep = 1) | 0–2 |
| Displacement (normal 1, ≥ volume-confirmed 2) | 0–2 |
| FVG present in the chain | 1 |
| Order Block present in the chain | 1 |
| Entry in discount (long) / premium (short) of the dealing range | 1 |
| Entry zone overlaps OTE | 1 |
| Kill zone (or London/NY session if kill zones are off) | 1 |
| ADR used below limit (default 85%) | 1 |
| **Volume sub-score** (cap default 3): RVOL > 1, spike, volume-confirmed displacement, volume-confirmed FVG, sweep with RVOL ≥ high | 0–3 |

---

## 3. Architecture (single file, ordered modules)

```
 1 declaration      2 constants/types   3 inputs (groups)   4 presets
 5 utilities        6 sessions          7 volatility/ADR    8 volume/RVOL
 9 swings+structure (f_structure x3: major/minor/internal)   10-12 BOS/CHoCH/MSS
13-15 liquidity pools, EQH/EQL, sweeps  16-17 FVG + mitigation   18 displacement
19-21 order blocks, mitigation, breakers   22-23 dealing range, OTE
24 HTF bias (request.security)   25-28 confluence, setup, entry/SL/TP, expiry
29 dashboard   30 plotting   31 alerts   32 analytics   33 object management
```

### Data flow (per bar)

```
top level (every tick):  series maths only: ATR, RVOL, displacement, pivots/structure events,
                         sessions, D/W/M levels, HTF trends (request.security, confirmed)
                                   │
                    if barstate.isconfirmed and enabled  ──►  ENGINE (mutates arrays/objects)
                                   │
   pools: sweeps → new pivots/levels/sessions
   FVG: mitigate → create        OB: mitigate/invalidate/breakers → create
   structure drawing → sweep arm → MSS classification
   setups: advance open setup (entry / TP / SL / expiry) → build new setup from candidate
                                   │
                     barstate.islast ──► dashboard, analytics table, dealing-range lines
                     alertcondition()/alert() ◄── event flags set by the engine
```

### Repaint / look-ahead strategy

* Pivots need `L` right-hand bars; engine acts only on confirmed pivots, documented delay = `L` bars.
* HTF: `request.security(tf, f_trend()[1], lookahead_on)` returns the **previous, completed** HTF bar's value, identical in history and realtime (one-HTF-bar lag by design).
* Realtime: state only mutates on closed bars. Plot shapes are gated with `barstate.isconfirmed`.
* Daily/weekly/monthly levels only publish *completed* previous periods.
* Analytics: outcome resolution starts on the bar **after** the signal.

### Object-management strategy

* Pools ≤ 40 (oldest removed, line/label deleted with it), FVG ≤ 10, OB ≤ 8, structure lines/labels ≤ 30, marks ≤ 30, setup visuals ≤ 5 (oldest visuals deleted). Taken/mitigated/expired objects are deleted immediately.
* All loops are bounded by those caps; no unbounded history scans (OB search ≤ `obLookback` bars, TP search over ≤ 40 pools).
* Slot-based session RVOL uses a fixed 1440-element array updated once per closed bar.

### Known limitations (not faked)

* No automatic news calendar; manual windows only.
* Lower-timeframe rows unavailable on higher-timeframe charts.
* Sweep detection is same-candle close-back (optionally wick-ratio); multi-candle reclaim sweeps are not tracked.
* Breaker blocks are informational, not used as entry zones.
* One active setup at a time (a pending setup may be replaced by an opposite one).
* Volume is usually tick volume; no buy/sell split exists in the data.
