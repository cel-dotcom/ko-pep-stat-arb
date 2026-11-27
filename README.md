# Statistical Arbitrage Study: KO/PEP and a 25-Pair Walk-Forward Scan
 
A first quant project: build a pairs-trading strategy, test it honestly, and report what happened, including the failures.
 
**Headline result:** a textbook pair (Coca-Cola / PepsiCo) passed a cointegration test in-sample and failed out of sample. A walk-forward strategy with a cointegration gate, applied unchanged to 25 pairs of US large caps, showed no detectable edge (holdout portfolio Sharpe 0.24, return 0.5% over five years, statistically indistinguishable from zero).
 
---
 
## 1. Hypothesis
 
If two stocks are cointegrated, their spread mean-reverts. Trading deviations of the spread (long the cheap leg, short the expensive leg) should earn a return after costs.
 
## 2. Data and tools
 
- Daily adjusted close prices from `yfinance`, 2015-01-01 to 2025-12-31
- Python: pandas, numpy, statsmodels, scipy, matplotlib; developed in Jupyter
- Costs: flat 10 bps per position change (no borrow fees or bid-ask modeling)
- Look-ahead control: positions decided at the close of day *t* earn the return of day *t+1*
## 3. Phase 1: KO/PEP with fixed parameters
 
**Setup.** Formation period 2015-2021, trading period 2022-2025. Hedge ratio from OLS on price levels, spread z-scored on a 60-day rolling window, enter at |z| > 2, exit at z = 0, stop at |z| > 4.
 
| Measure | Formation (2015-2021) | Trading (2022-2025) |
|---|---|---|
| Correlation | 0.96 | n/a |
| Cointegration p-value (Engle-Granger) | **0.024** | **0.19** |
| Hedge ratio (beta) | 0.29 | fixed |
| Mean-reversion half-life | 43.6 trading days | n/a |
 
**Backtest (trading period, net of costs):** Sharpe -0.72, total return -16.3%, max drawdown -20.4%, 13 trades. Gross return was -14.1%, so costs explain only about 2 points; the signal was the problem.
 
**Diagnosis.**
 
- Two long trades (shorting the spread against an upward drift, held 92 and 108 days) account for most of the loss. The other 11 trades were roughly flat in total.
- Every trade held under about 30 days was a winner. Every large loser was held 70+ days.
- The raw spread stayed near its formation mean until mid-2024, then climbed from roughly 10 to 25-33 and did not return. This is a regime shift, not noise.
- The |z| > 4 stop never protected the position, because a 60-day rolling z-score re-centers as the spread drifts. A rolling z-score makes a drifting spread look stationary.
**Lesson:** a significant in-sample cointegration p-value does not guarantee the relationship persists, and a rolling z-score chart can look healthy while the strategy is losing money.
 
## 4. Phase 2: walk-forward strategy with a cointegration gate
 
Changes made in response to the Phase 1 diagnosis:
 
- Every 21 days, re-estimate beta, spread mean/std, cointegration p-value and half-life on a trailing window, using only past data
- Log prices instead of price levels
- Trade only when the gate passes: p < 0.05 and 0 < half-life < 120 days; exit if the gate closes
- Each open trade keeps the parameters it was entered with
- Time stop at 2x the half-life measured at entry
- Entry |z| > 2, stop |z| > 4
**KO/PEP diagnostics (2-year window, 108 rolling windows):**
 
- Only 2% of windows passed p < 0.05 (minimum p = 0.016). A 5% test would pass about 5% of the time by chance alone, so there is no evidence of a stable relationship.
- Half-life was acceptable in 93% of windows. Speed of reversion was not the problem; stability was.
- The rolling half-life jumped from roughly 50 days to 250+ between mid-2024 and early 2025, which is when the Phase 1 trades lost money.
- The strategy never traded KO/PEP.
## 5. Phase 3: 25-pair scan
 
**Pair selection (rule fixed before looking at results).** Three sectors (staples, financials, energy). Within each sector, take the most correlated pairs ranked on 2015-2017 returns only (quotas 9 / 8 / 8). Selection never uses data the strategy trades on.
 
**Frozen rules.** Lookback 756 days (3 years), rebalance 21, p < 0.05, entry 2, stop 4, time stop 2x half-life, 10 bps costs.
 
**Honesty note on the one tuned choice.** The move from a 2-year to a 3-year lookback was made after seeing that the 2-year gate rarely opened on KO/PEP and five other pairs. It was chosen for a stated reason (more test power) and then frozen, but it was not pre-registered before any results were seen.
 
**Split.** 2018-2020 development, 2021-2025 holdout, evaluated once.
 
**Results.**
 
| Measure | Value |
|---|---|
| Pairs that traded in holdout | 10 of 25 |
| Pairs with 5+ trades | 3 (CL/KMB, MO/PM, USB/PNC) |
| Median / mean Sharpe (traded pairs) | 0.10 / 0.01 |
| Best / worst Sharpe | 0.80 / -0.78 |
| Positive Sharpe | 6 of 10 (sign test p = 0.75) |
| Equal-weight portfolio Sharpe, dev (2018-2020) | -0.15 |
| Equal-weight portfolio Sharpe, holdout (2021+) | 0.24 |
| Portfolio return / max drawdown, holdout | 0.5% / -1.0% |
 
**Interpretation.**
 
- The portfolio t-statistic is roughly 0.24 x sqrt(5) = 0.5, far below the ~2 needed for evidence of an edge.
- Only 3 pairs produced enough trades to evaluate individually, and they split one good, one flat, one bad.
- The best-looking pairs (CVX/COP, CVX/OXY, COP/OXY) share stocks and the same oil-price shocks, so they behave like one observation rather than three.
- KO/PEP, the most correlated staples pair, never traded.
- Time in market was 2-9% per pair, which is about what false positives from a 5% test would produce. A direct check is described below.
## 5b. 2026 out-of-sample check (run once, parameters frozen)
 
Data through 2026-11-25 (227 trading days), never used by any earlier code. Same 25 pairs, same frozen rules, scored on 2026 only.
 
| Measure | Value |
|---|---|
| Pairs that traded | 6 of 25 (13 trades in total) |
| Pairs with positive Sharpe | 0 of 6 |
| Median Sharpe (traded pairs) | -1.35 |
| Equal-weight portfolio Sharpe | -2.15 |
| Portfolio return / max drawdown | -1.62% / -1.99% |
 
The six pairs that traded were PG/CL, PEP/PM, PG/KMB, EOG/OXY, CVX/COP and CVX/OXY. They fall into two clusters (staples and oil producers), so they are closer to two observations than six. Most of the loss came from a sharp drop around late January to early February, with smaller losses in October-November. The oil cluster (CVX/COP, CVX/OXY) had the best holdout Sharpe ratios in 2021-2025 and lost money in 2026, so the holdout result did not persist. With 13 trades concentrated in two clusters, this is too little data for statistical inference; it is consistent with no edge and gives no evidence of one.
 
## 6. Gate pass-rate check (selected vs random pairs)
 
The test compares the gate pass rate (share of rolling 3-year windows with p < 0.05 and a half-life between 0 and 120 days) on the 25 selected within-sector pairs against 60 random cross-sector pairs with no economic link (fixed seed 0, set before looking at results).
 
| Group | Mean pass rate | Median | Never passed |
|---|---|---|---|
| Selected within-sector pairs | 9.5% | 6.2% | 6 of 25 |
| Random cross-sector pairs | 9.2% | 6.2% | 8 of 60 |
 
Mann-Whitney p (selected > random): **0.601**
 
**Interpretation.** The gate passes about as often on pairs with an economic link as on unrelated pairs. It does not discriminate between the two groups, so the trades it allowed are largely noise trades, which is consistent with the near-zero holdout Sharpe. Both pass rates are above the 5% a well-calibrated test would give by chance. Likely reasons are heavily overlapping windows (one lucky stretch counts in many consecutive windows) and shared market-wide trends affecting many pairs at once, so the absolute rate should not be over-read. Caveat: with 25 and 60 pairs, the check can only detect a large difference, so it does not prove the pairs have zero cointegration signal, only that this gate did not find a measurable one.
 
## 7. Limitations
 
- **Survivorship bias.** Tickers are today's large caps; delisted or collapsed stocks are absent.
- **Simplified costs.** Flat 10 bps, no borrow fees, no bid-ask modeling, execution at the signal-day close.
- **Overlapping windows.** Rolling tests share most of their data, so pass rates and p-values are indicative, not exact.
- **Non-independent pairs.** Pairs sharing tickers or sector shocks reduce the effective sample size below 25.
- **Small trade counts.** Most pairs had 0-4 trades in the holdout, so individual Sharpe ratios carry little weight.
- **Trade counting quirk.** A trade straddling the 2021 split can be undercounted (PG/KMB shows a loss with 0 trades).
- **Low test power.** Engle-Granger on 2-3 years of data has limited power; the gate protects against breakdowns but starves the strategy of trades.
- **Held-out data is used up.** The 2021-2025 holdout cannot be reused for tuning.
## 8. Conclusion
 
With disciplined methodology (formation/trading split, walk-forward estimation, a pre-specified pair-selection rule, a held-out period, and multiple-testing guardrails), the standard pairs-trading strategy on US large caps with daily data produced no detectable edge. This is a common outcome: the trade is heavily arbitraged, relationships break, and short-window cointegration tests are noisy. The most useful findings were methodological: a rolling z-score hides regime shifts, a significant in-sample p-value does not generalize, and a gate that protects capital also removes most of the sample.
 
## 9. Next steps
 
- Evaluate the frozen rules once on 2026 data (never used by any code above)
- Forward paper-trade with the frozen rules
- A new project phase on fresh data, with the design fixed in advance: ETF pairs, a Kalman-filter hedge ratio, or intraday data
## 10. Reproducing
 
Run notebook cells in order and use Restart Kernel and Run All before trusting any output. Results saved to `pair_scan_results.csv`.
