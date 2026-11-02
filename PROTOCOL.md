# Pre-Analysis Protocol

**Project:** An independent validation of the ERC portfolio model
**Author:** Héctor Ornelas
**Registered:** 2026-10-16
**Status:** I committed this file before I downloaded any data. Its commit timestamp can be checked against the timestamps of the data and the results.

This document exists so that the decisions that shape the results are on the record before the results exist.

Any analysis involves choices that are defensible on their own. Made after seeing the numbers, the same choices stop being analysis and become selection. Once the work is finished nobody can tell the two apart from the outside, including me, so writing the choices down first is what makes the difference checkable.

A validation report is a claim about someone else's rigor, so my own method has to survive the same test. The easiest way to fail it is to decide something after seeing the result that decision affects: to pick the estimation window that makes the model look better, or to report the one configuration out of thirty-six that worked. That does not take bad intentions. It only takes exploring until the numbers look reasonable.

So I fix three things in advance: the sample split, the hypotheses, and the rubric I use to grade findings. Those are the three a later choice could bend. Dozens of smaller decisions will come up while I work, and I am not claiming those were made today. I am claiming these three did not move.

If I change something here, I do not edit this document. I add a row to the deviations log in §13 with the date and the reason, and I report the change as a decision I made after registering this protocol.

---

## 1. Model under validation

Maillard, S., Roncalli, T. and Teïletche, J. (2009). *On the properties of equally-weighted risk contributions portfolios.* Working paper, May 2009 version, 23 pp. SSRN 1271972. Later published as *The Properties of Equally Weighted Risk Contribution Portfolios*, Journal of Portfolio Management 36(4), 60–70, 2010.

I use the working paper and not the journal version, because the journal version drops Appendices A.1 to A.4 and those appendices hold the proofs that my replication checks rely on. A reader who tries to verify those checks against the journal version will not find them.

I chose this model for two reasons. The first is that it belongs to the family of portfolio rules that use only the covariance matrix and never estimate expected returns, which is the sensible response to how much harder a mean return is to estimate than a volatility. The second is that the paper proves several closed-form results and prints one numerical example that can be reproduced exactly, so I can show that my implementation is correct before I criticize anything. A finding produced by broken code is worse than no finding, and this was the cheapest way to close that risk.

What I validate is the published method and not any particular program. I write my own implementation from the description in the paper, and I do not read or reuse an existing one, because if I had copied a working implementation then matching it would prove nothing. A defect in my code is mine and not the model's, and I say which one I mean whenever the distinction matters.

A model is not fit or unfit on its own, it is fit for a use, so I have to name one. I assume the model has been submitted for approval at an asset manager, for a long-only, unlevered, multi-asset allocation rebalanced monthly and with no expected-return inputs, and I write the report the way an independent validation function would write it, following the structure of the Federal Reserve's SR 11-7.

---

## 2. Universe

Nine US-listed ETFs, chosen to approximate the asset classes of the global diversified example in §4.2 of the source paper.

| Ticker | Exposure | Stands in for |
|--------|------------------------------|------------------------------------|
| SPY | US equity | MSCI World, US share |
| EFA | Developed ex-US equity | MSCI World, ex-US share |
| EEM | Emerging market equity | MSCI EM Europe, MSCI AC Asia ex JP |
| IEF | US Treasuries, 7–10y | JP Morgan Govt Bond US |
| TLT | US Treasuries, 20y+ | JP Morgan Govt Bond US, long end |
| LQD | US investment grade credit | — |
| HYG | US high yield credit | ML US High Yield Master II |
| GLD | Gold | — |
| DBC | Broad commodities | S&P GSCI |

I fix this list here and I do not revise it after seeing any result. It has to be fixed, because adding or dropping one asset can change the conclusion and nothing would stop me from doing that until the model looked the way I expect.

I use a multi-asset universe on purpose. The three examples in the paper behave very differently from each other, and the difference comes from the inputs rather than from the model. With US sector indices, where the volatilities cluster near 19% and the correlations are high, the paper reports that ERC and equal weight are nearly the same portfolio. With agricultural commodities, where the volatilities differ widely and the correlations are low, the weights collapse to roughly inverse volatility. Only the global diversified case is heterogeneous in both, and it is the only one where estimating the full covariance matrix has to earn its place, so that is the case worth validating.

This universe is similar to the paper's and it is not the same one. Euro government bonds and emerging market sovereign debt are in theirs and not in mine, and investment grade credit and gold are in mine and not in theirs, so I do not present my results as a replication of their backtests.

HYG started trading on 2007-04-04, and I allow one month after the launch before using its prices, which is also a reasonable margin for the thin trading of a new fund. No sample can start before 2007-05-01, and every other fund here has history back to 2006 or earlier, so this is what sets the start date in §3.

---

## 3. Data

Daily total-return series for the nine funds, from 2007-05-01 to 2025-12-31.

I use total returns and not price returns, because four of the nine funds distribute meaningful income. Price-only series would leave the volatilities almost unchanged, so the ERC weights would barely move, but every performance comparison would be distorted. I use simple returns rather than log returns, because a portfolio's return is a weighted sum of the simple returns of its holdings and log returns do not add up that way across assets.

**Sources.** Stooq through pandas-datareader is the primary source and yfinance is the check. I download both, compare them, and report the size of the disagreement in the appendix. I fix the tie-break here rather than later: Stooq stands unless I find a specific and documented defect in a series, and I do not switch sources because one of them produces a result I prefer. Where the two disagree by more than 10 basis points on a daily return I investigate that date and record what I found.

**Alignment.** I keep only the dates on which all nine funds traded, and I report how many observations that removes. A day missing from one series is not filled in.

**Reproducibility.** The price files are committed in `data/`, so anyone who clones this repository runs the analysis against the exact series I used. Both vendors revise adjusted prices over time, and a fresh download a year from now can differ in the last decimals, which would leave a reader unable to tell a mistake in my work from a change in the data underneath it. `download_data.py` records where every series comes from and can rebuild the files, and the notebooks read the saved copies rather than downloading anything. The data enters the repository after this protocol is committed, and the commit dates show it. Any stochastic procedure uses seed 42, and I expect only one, the bootstrap used to test H2.

**Survivorship.** All nine funds still exist in 2026 and I picked them knowing that, so a fund that had closed would never have entered this list. The effect is small for broad index ETFs, but it is still a selection I made with information that the 2007 investor did not have.

---

## 4. Sample split

| Sample | Period | Use |
|-------------|-------------------------|------------------------------------------|
| Development | 2007-05-01 → 2016-12-31 | Implementation, process verification, conceptual soundness, the full sensitivity grid |
| Holdout | 2017-01-01 → 2025-12-31 | Run once, after the development work is frozen |

The 252-day estimation window consumes the first year, so the development backtest starts around May 2008 even though the data starts a year earlier, and I report the first rebalance date rather than the data start date.

The first holdout rebalance estimates its covariance from 2016 returns, which are development-period data. That is correct and worth stating, because an investor in January 2017 had exactly that history, and estimating from the past is not look-ahead.

I run the holdout once, with one configuration, and I commit the output as a frozen CSV. Everything exploratory happens in the development sample, including all thirty-six configurations of the sensitivity grid.

There is one case where I will run the holdout again, and I want it written down before it happens. If the code turns out to be broken, I fix it and rerun, and then §13 carries both runs and the report carries both sets of numbers. If the code is fine and I simply do not like the answer, the answer stands.

---

## 5. Backtest protocol

This is the base case, and §7 varies it.

Following the source paper: rebalance on the last trading day of each month, estimate the covariance matrix from a rolling window of 252 daily returns, long-only and fully invested with `w >= 0` and `sum(w) = 1`, no leverage and no expected-return inputs.

**Timing.** I estimate the covariance from returns up to and including the rebalance date, and the new weights take effect at the next trading day's close. Computing and trading on the same close would be a one-day look-ahead, and it is the easiest one to introduce without noticing.

**Drift.** Between rebalances I hold shares and not weights, so the weights move with prices. Turnover is the distance between the drifted weights on the rebalance date and the new target, and not the distance between consecutive targets.

**Solver failure.** If a solver does not converge on a given date I carry the previous weights forward, log the date, and report how many dates that happened on. I decide this now so that I am not improvising a fallback once the backtest is running.

**Risk-free rate.** The Fed funds effective rate from FRED, which is what the paper uses. That is a third source beyond the two in §3 and the same download script pulls it. Annualization uses 252 trading days.

---

## 6. Transaction costs

Rebalancing means trading and trading costs money, so this section fixes how much and how it is charged. The base case is 5 basis points per unit of notional traded, with sensitivity at 1 basis point and at 15. At each rebalance the cost is that rate times the sum of the absolute weight changes, so a sale and the purchase it funds each pay.

Five basis points is on the high side for SPY or IEF and about right for DBC or EEM. I still charge every fund the same, because to do it properly I would need to know what each fund's spread actually was on each day since 2007, and that is not published anywhere. Guessing one number per fund would only look more precise than it is, so I let the three rates cover the range instead.

The rate is also constant across time, and that is the part I am least comfortable with. Spreads widened sharply in late 2008 and in March 2020, and those are the same periods when a volatility spike pushes the model to trade more, so both effects land at once and a flat rate understates costs precisely when they hurt. I treat this as a finding rather than a footnote.

Every strategy pays, benchmarks included. Equal weight and 60/40 rebalance monthly as well, and charging the model but not its benchmarks would settle the comparison before the data does.

Market impact, taxes and securities lending income are out of scope. Costs are proportional to turnover, so the results do not depend on portfolio size, which is an assumption of its own and one that fails for a large enough allocation.

The source paper omits transaction costs and says so, so adding them closes a gap the authors flagged rather than raising an objection I invented.

---

## 7. Sensitivity grid

| Dimension | Values |
|----------------------|-------------------------------------------------|
| Estimation window | 63 / 126 / 252 / 504 trading days |
| Covariance estimator | Sample / EWMA (lambda 0.94) / Ledoit-Wolf |
| Rebalancing | Monthly / quarterly / 5% weight-drift threshold |

Thirty-six configurations, all of them run in the development sample.

The grid is not a search for the best configuration. Finding the best one is the job of the person who builds the model, and doing it here would turn me into the person I am supposed to be checking. What the grid measures is how much the results depend on choices the paper leaves open, so the finding is the spread across the thirty-six and not the maximum, and the appendix carries the full grid while the body reports the range and the dispersion rather than a winner.

**Which configuration goes to the holdout.** The base case in §5, which is 252 days, sample covariance and monthly rebalancing. I pick it because it is the paper's own protocol, and I pick it now so that I cannot pick it later for having won. If another configuration dominates in development I report that it did, and it still does not go to the holdout, because promoting the development winner is exactly the selection this document exists to prevent.

The 63-day window is in the grid on purpose even though nobody would run it. With nine assets there are forty-five distinct parameters to estimate, and sixty-three observations give fewer than two observations per parameter, so that is where estimation error should show up first. Lambda 0.94 is the RiskMetrics convention for daily data, obtained by minimizing average squared forecast error across a large set of series in the 1996 Technical Document, and I take it unchanged rather than fitting it, because the decay rate is a dial I could turn until the results improved. It was calibrated in the mid-1990s on a mostly FX and equity sample, so it is not guaranteed to be optimal for a multi-asset ETF universe, and I record that as an assumption rather than adjusting it.

---

## 8. Benchmarks

Four of them, and each one answers a different question.

**Inverse volatility**, with weights proportional to the reciprocal of each asset's volatility. It uses only the individual volatilities and ignores every correlation, where ERC uses the whole covariance matrix, so the gap between the two measures what the correlations are worth. Equation (3) of the paper sharpens this, because under constant correlation the two are the same portfolio. This is the diagnostic benchmark rather than a rival.

**Equal weight.** It uses nothing at all, no volatilities, no correlations and no estimation, so it is the zero-information floor. Together with the previous one it splits the value of the model in two: the step from equal weight to inverse volatility is what the volatilities buy, and the step from inverse volatility to ERC is what the correlations buy.

**60/40 in SPY and IEF**, rebalanced monthly. This is not an information benchmark, it is what an allocator plausibly holds today, so it answers whether anyone would switch.

**Minimum variance, long-only.** The opposite end from equal weight. Appendix A.3 proves that the volatility of the ERC portfolio sits between these two, so I compute all three at every rebalance and check that the ordering holds, which makes these two a running correctness check and not only competitors.

Every benchmark runs on the same universe, the same dates, the same rebalancing schedule, the same cost model and the same estimation window where one applies, because a benchmark given easier terms would not be a benchmark.

**What counts as beating.** I fix this now so that I cannot pick the flattering metric later. The primary comparison is the Sharpe ratio after costs on the holdout, and beside it, carrying equal weight in the conclusion, is the dispersion of realized risk contributions, because that tests the model's own claim rather than a performance claim. Maximum drawdown and turnover are secondary, and they shape the conditions attached to the use decision rather than the decision itself.

---

## 9. Metrics

Three families, and the third is what makes this a validation rather than a backtest.

**Performance.** Annualized return, volatility, Sharpe ratio, maximum drawdown and Calmar ratio, plus the 1% Value-at-Risk at one day, one week and one month, as the paper reports it.

**Implementation.** Annualized turnover and cost drag in basis points per year, at each of the three cost rates.

**The model's own claim.** The ERC portfolio does not promise high returns and never claimed to, it promises that every asset contributes the same share of risk, so judging it only on Sharpe would be judging it against a claim it did not make. At each rebalance the weights are set so that the ex-ante contributions are equal, and over the month that follows I recompute the contributions using that month's realized covariance and measure how far they drifted, using the Gini index of the realized contribution shares, where zero means perfectly balanced. I use Gini and Herfindahl because Appendix A.4 of the paper uses them, so my numbers sit beside theirs, and I compute both on weights and on risk contributions as the paper does.

**Sub-periods.** 2018, 2020 and 2022, fixed here. 2018 is a volatility shock without a recession, 2020 is a fast crash followed by a fast recovery, and 2022 is the one that matters most, because equities and bonds fell together and that is the condition the whole diversification argument rests on. I name the three now, because choosing sub-periods after seeing results is the same selection problem as choosing a configuration.

**Testing H2.** Sharpe differences are noisy, so I use the Ledoit-Wolf robust Sharpe ratio test with a stationary bootstrap instead of comparing the two numbers by eye. Nine years of holdout gives low power against small differences, and I say so in advance, because failing to reject will not be evidence that the two are equal, only that this sample cannot separate them.

---

## 10. Process verification

The three backtests in the paper use proprietary index data and cannot be reproduced, so I do not attempt them. Verification rests instead on the one published figure I can reproduce exactly and on the results proved in the appendices, which is the stronger of the two options anyway, because a table can be matched by coincidence and a set of mathematical identities cannot.

| # | Check | Source | Type | Tolerance |
|---|-------|--------|------|-----------|
| 0 | Four assets with volatilities of 10/20/30/40% and constant correlation give weights of 48/24/16/12% | §4.1, p. 9 | unit | within printed rounding |
| 1 | Constant correlation gives weights proportional to the reciprocal of volatility | eq. (3) | unit | 1e-8 |
| 2 | Equal volatilities give weights proportional to the reciprocal of the weighted average correlation | eq. (4) | unit | 1e-8 |
| 3 | At the solution, weights are proportional to the reciprocal of beta | eq. (5) | unit | 1e-8 |
| 4 | Two assets: the closed form, which does not depend on the correlation | §3.1 | unit | 1e-8 |
| 5 | The volatility ordering, minimum variance ≤ ERC ≤ equal weight | Appendix A.3 | runtime | exact ordering |
| 6 | Both solvers agree, and the risk contributions are equal at the solution | §3.3, eqs. (6) and (7) | unit | 1e-8 and 1e-10 |
| 7 | Scale invariance and permutation invariance | definition | unit | 1e-10 |

Check 0 is the only one against a published figure, and the paper prints it to two significant figures, so I require agreement within rounding rather than to machine precision.

Check 5 is not a unit test. Appendix A.3 proves the ordering in general, so it has to hold at every rebalance date, and I assert it at every rebalance rather than verifying it once on an example. A window where it fails is either a bug or a finding, and I investigate it before continuing.

**Two implementations.** I write the non-convex formulation of equation (6) and the convex log-barrier formulation of equation (7) separately, and they should agree. The authors prefer (6) for numerical convenience even though only (7) guarantees a unique solution when the covariance matrix is positive definite, so any case where the two disagree is a conceptual soundness finding and gets reported rather than discarded.

**What a failure means.** A failing check stops the work, and I do not move on to the sensitivity grid in §7 until all of them pass, because a finding produced by broken code is worse than no finding at all. If a check cannot be made to pass I report that and say plainly what it implies for everything downstream.

---

## 11. Hypotheses

I write these before running anything. Some of them will turn out to be false, and that is the point, because a pre-registration where everything is confirmed reads like a pre-registration written afterwards.

**H1.** The realized risk contributions out of sample will show dispersion materially above zero, which means the portfolio is balanced before the month and not after it. This is the model's central claim and I expect it to be the first finding.

**H2.** ERC will not beat inverse volatility by a distinguishable margin after transaction costs. If this holds, the finding is that estimating the full covariance matrix does not pay for itself.

**H3.** The sensitivity of the weights to the choice of covariance estimator will be larger than their sensitivity to the length of the estimation window.

**H4.** In 2022 the ERC portfolio will show a worse drawdown than 60/40, because the low volatility of bonds induces a concentration in fixed income and that year equities and bonds fell together.

**H5.** The contributions to Expected Shortfall will be systematically skewed toward the fat-tailed assets, HYG, EEM and DBC, despite the portfolio being balanced in variance.

---

## 12. Severity rubric

I fix this before seeing any result, so that findings are not graded to fit whatever story the data ends up telling.

| Severity | Definition |
|----------|------------|
| **High** | The finding materially affects whether the model is fit for its stated use. Either the central claim fails, or a reasonable implementation choice changes the conclusion. It requires remediation before use. |
| **Medium** | The finding affects the results measurably but does not change the use decision. It requires disclosure, monitoring, or a documented constraint on use. |
| **Low** | The finding is theoretically valid but immaterial at realistic parameter values. It is recorded in the appendix rather than in the findings table. |

A finding enters the main findings table only at High or Medium, and the table is capped at seven rows. A validator who flags everything stops being read.

---

## 13. Deviations log

Any departure from this protocol is appended below with the date, what changed and why. It is empty at registration.

| Date | Deviation | Reason |
|------|-----------|--------|
| — | — | — |

---

## 14. Limitations known at registration

I state these here so that they are not presented later as discoveries.

- **Independence is partial.** The same person implements the model and validates it. This is the central limitation of the exercise and the report restates it rather than working around it.
- I have earlier personal work on risk parity. The implementation here is written from scratch from the paper, and the earlier work is disclosed and not reused.
- The backtests in the source paper are not reproducible, because their data is proprietary. The ETF universe is a proxy and it introduces tracking error, a shorter history and replication costs that the original indices do not have.
- No production implementation, model inventory, governance or control review is in scope.
- This is a personal project and not an institutional validation. It carries no regulatory authority and does not claim any.
