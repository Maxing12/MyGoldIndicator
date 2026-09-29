# XAUUSD SMC + ICT + Volume Indicator (Pine Script v6)

`XAUUSD_SMC_ICT_Volume.pine` is a TradingView overlay indicator for XAUUSD. It links **HTF bias → premium/discount → liquidity → sweep → MSS/CHoCH → displacement → FVG/OB → retracement → entry/SL/TP** into one setup engine and grades each setup with a **Confluence Score** (not a probability).

Definitions, resolved contradictions in the brief, data flow and the repaint strategy are in [`docs/SPECIFICATION.md`](docs/SPECIFICATION.md).

## Verification status (please read)

* No TradingView/Pine compiler was available where this was written. The file was built module by module, statically reviewed, and syntax-checked with an open-source Pine parser (`pynescript`, v5 grammar) where possible. **It has not been compiled by TradingView.** Open it in the Pine Editor once and press *Add to chart*; if the editor reports a compile error, send me the message and I will fix it.
* Behaviour on live charts (signal frequency, thresholds) has not been backtested. Presets are starting points, not tuned or profitable settings.

## Install

1. Open TradingView and a **XAUUSD** chart (e.g. OANDA:XAUUSD or your broker feed).
2. Open **Pine Editor** (bottom panel).
3. **Open → New blank indicator**.
4. Delete the template and paste the full contents of `XAUUSD_SMC_ICT_Volume.pine`.
5. **Save**, then **Add to chart**.
6. Set the timeframe to **15m** (primary) or **5m** (entry refinement).
7. Open the settings (gear). Start with *Preset = Balanced*, *Volume filter = CONFIRMATION*, and set **Timezone** and session times for your market.
8. Alerts: right-click the chart → *Add alert* → Condition = the indicator. Use a named condition (e.g. "A+ LONG") for a short message, or **Any alert() function call** for the detailed text (entry, SL, TPs, RR, score). Use *Once Per Bar Close*.

## Recommended starting settings (not guaranteed profitable)

| Chart | Purpose | Suggestion |
|---|---|---|
| **XAUUSD 15m** | Setup timeframe | Preset Balanced; Sensitivity Follow Preset (major 10 / minor 5 / internal 3); Min grade A; kill zones ON; expiry 10 bars |
| **XAUUSD 5m** | Entry refinement | Preset Balanced or Conservative; Sensitivity Balanced; increase *Sweep valid for MSS* to 40 and *Bars allowed after MSS* to 4; 15M/1H/4H rows populate, the 5M row is the chart itself |
| **XAUUSD 1H** | HTF bias only | Use to read structure; turn off setups by setting *Minimum grade* to A+ or ignore them. Higher-timeframe rows (below the chart TF) show `n/a` |

Presets: **Conservative** (score A≥8 / A+≥11, RR≥2.5, sweep must be significant, MSS must be a CHoCH, HTF must be aligned, STRICT volume), **Balanced** (7/9, RR≥2.0, HTF not opposed), **Aggressive** (6/8, RR≥1.5, displacement optional, HTF ignored), **Custom** (values in group 10).

## What you see

* **Structure:** thick lines = major BOS/CHoCH, dotted thin = minor; `MSS ↑/↓` = sweep + break + displacement.
* **Liquidity:** dotted horizontal lines (EQH/EQL, PDH/PDL, PWH/PWL, session H/L); `BSL SWEEP` / `SSL SWEEP` with class Minor/Significant/Major.
* **FVG/OB boxes:** transparent; `FVG+` = strong, `FVG vol` = volume-confirmed; breakers dashed.
* **Marks:** `DISP ↑/↓`, `VOL SPIKE`, `POTENTIAL EXHAUSTION`, optional `VOL DIV`.
* **Setup:** one compact label (grade, HTF, location, chain checks, RVOL, entry, SL, TP1/2, RR, score), entry zone box, SL and TP lines. Status: PENDING → ENTERED → TP/SL, or EXPIRED / INVALIDATED.
* **Dashboard** (top right): HTF rows, bias, location, nearest BSL/SSL, PDH/PDL, sessions, ADR, volatility, RVOL, active setup, journal of the last setup. **Analytics table** (bottom left): post-signal outcomes.

## Trading workflow

1. 4H/1H bias (dashboard). 2. Premium/discount. 3. Nearest liquidity target. 4. Wait for a sweep. 5. Wait for MSS/CHoCH. 6. Look for displacement. 7. Check the FVG/OB entry zone. 8. Check RVOL/volume. 9. Wait for the retracement. 10. Read entry/SL/TP. 11. Check RR. 12. Execute only if your own rules and risk management allow it.

## Key behaviours and limits

* **Non-repainting by construction:** engine runs on closed bars only; swings are confirmed pivots (signal delay = pivot length); HTF uses the last completed HTF bar. `Show Developing Signals` (default OFF) only adds dashboard text.
* **Setup timing:** a setup is published when sweep → MSS → displacement → FVG/OB are complete; it is *ENTERED* only after price later trades to the zone midpoint. RR is measured entry-midpoint → SL and → TP2.
* **Volume** is usually tick volume; it is never a standalone or directional signal, and its score contribution is capped (default +3). With no volume data it switches itself off.
* **Analytics** are theoretical, bar-based, exclude spread/slippage, and resolve same-bar TP/SL as SL. Not guaranteed performance.
* No automatic news calendar (Pine cannot read one); only manual windows.
* Full limitation list is at the top of the `.pine` file and in the specification.
