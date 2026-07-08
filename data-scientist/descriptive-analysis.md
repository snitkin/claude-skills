# Rigorous Descriptive Analysis

Systematically characterizing patterns, distributions, and associations — the *why* and
*when*, not implementation syntax (load `stata` for that). `eda-checklist.md` asks "what
does this data look like?"; this file asks "what does it tell us?"

## When Description Is the Right Approach

Description isn't a lesser cousin of causal inference — it's the right endpoint for
prevalence ("what fraction of districts offer full-day pre-K?"), trend ("how has
enrollment changed?"), disparity ("how large is the Black-White gap at each percentile?"),
and benchmarking questions. Every regression, even a causal one, is fundamentally
estimating a conditional expectation function E[Y|X] — that's a descriptive object; whether
it's *causal* depends on design, not technique. Rigorous description with proper
subgroups, correct weights, and honest uncertainty is more valuable than a shaky causal
design — don't let "just descriptive" become an excuse to be sloppy, and don't reach for a
causal framing a straightforward description doesn't need. See `causal-inference.md` and
`research-questions.md` for the causal/correlational line.

## Summary Statistics

Match the statistic to the shape. If mean and median diverge by more than ~20%, the
distribution is skewed — lead with the median and report percentiles, not just mean/SD.
For right-skewed, strictly positive variables (income, spending, enrollment), consider a
log transform: differences in logs approximate percent differences for small gaps (0.10
log points ≈ 10%), but use the exact exp(β)−1 formula once gaps exceed ~0.20. Zeros break
logs — use `ihs = ln(x + sqrt(x^2+1))` (inverse hyperbolic sine) rather than `log(x+1)`
when zeros are substantively meaningful, not just a nuisance.

```stata
summarize var, detail          // mean, sd, skewness, percentiles in one call
tabstat var, stats(mean median sd p10 p90) by(group)
gen ihs_income = ln(income + sqrt(income^2 + 1))   // handles zeros/negatives
```

Percentile tables (every 5th or 10th percentile) describe a distribution far better than
mean/SD alone, and are standard for wage/wealth work. When building summary tables:
group related variables, round consistently, always report N (it can vary across rows due
to missingness), and consider group columns with a difference column when comparing
groups (`estpost tabstat ..., by(group)` + `esttab`).

## Subgroups and Stratification

Label subgroup cuts as **a priori** (pre-specified, most credible) vs. **exploratory**
(data-driven, valuable but carries multiple-comparison risk) vs. **post hoc** (explaining
a surprise after seeing it — treat with real skepticism). If comparing many subgroups,
either correct for multiple comparisons (Benjamini-Hochberg for exploratory work) or
reframe as "how much does the estimate vary across groups" rather than testing each cut
individually — and report every comparison attempted, not just the significant ones.

Watch for the **ecological fallacy**: a district-level correlation between poverty and
graduation rates is a statement about districts, not students — don't restate it as
"poor students graduate less." Always state the unit of analysis explicitly.

Minimum cell sizes: ~30 for means/proportions, ~100 for percentile estimates within a
cell; collapse categories or flag small cells rather than presenting them as comparable to
large-cell estimates. Federal education data (NCES) has hard suppression rules (cells <3,
sometimes <5 or <10) — check the source documentation, not just your own judgment.

**Education note**: FRPL eligibility has been an unreliable poverty proxy since the
Community Eligibility Provision (2014–15) let high-poverty schools report 100% FRPL for
every student. Prefer SAIPE estimates or direct certification counts where available.

## Distributional Analysis

Two groups with equal means can have very different distributions — whenever the question
involves inequality, heterogeneity, or "did the whole distribution shift or just the
tails," go beyond the mean.

```stata
kdensity var, bwidth(#)                 // kernel density; same bandwidth across groups being compared
qqplot var1 var2                        // compare shapes across two distributions
ksmirnov var, by(group)                 // formal distributional-equality test (large-N: even trivial gaps are "significant")
xtile pctile10 = var, nq(10)            // decile assignment
qreg var x1 x2, quantile(0.1)           // conditional quantile — does X matter more at the bottom of Y's distribution?
```

Report quantile ratios (90/10 overall inequality, 90/50 upper tail, 50/10 lower tail)
alongside the mean — they show *where* in the distribution a gap lives, which the mean
alone can't.

## Cross-Tabs and Rates

```stata
tabulate rowvar colvar, row     // % of colvar within each level of rowvar — pick direction deliberately
tabulate rowvar colvar, chi2 V  // chi-squared + Cramer's V
```

Large administrative datasets make chi-squared "significant" for trivial associations —
always report Cramer's V (or an odds ratio for 2×2 tables) alongside the p-value; V < 0.1
is rarely substantively meaningful regardless of significance. **Always report the
denominator** for any rate ("more suspensions in district A than B" is meaningless without
enrollment counts). Check for **Simpson's paradox** whenever an aggregate trend surprises
you: compute it within subgroups before trusting the aggregate — a rising state average
can hide a decline in every subgroup if the population composition shifted (this is the
shift-share decomposition, below, applied informally).

## Trend and Time-Series Description

```stata
tsset panel_id time
tssmooth ma trend_ma = outcome, window(6 1 5)     // moving average
gen growth = (outcome - L.outcome)/L.outcome       // simple growth
gen log_growth = D.ln_outcome                      // log-difference: additive over time, symmetric gains/losses
```

Log-differences are usually preferred in economics because they chain additively
(growth 1→3 = growth 1→2 + growth 2→3). For seasonality, year-over-year comparisons are
often simpler and more interpretable than formal decomposition. For unknown structural
breaks, `estat sbknown`/`estat sbsingle` (Stata's structural break tests) beat eyeballing
the series. Index a series to a representative (non-outlier) base year = 100 before
comparing multiple series on different scales.

## Decomposition Methods

| Question | Method | Stata |
|---|---|---|
| Why do two groups have different average outcomes? | Oaxaca-Blinder | `oaxaca` |
| Why do two populations have different aggregate rates? | Kitagawa | manual (rate = Σ group-share × group-rate) |
| Which controls explain the most of a regression gap? | Gelbach | `b1x2` (SSC) |
| Is an aggregate change within-group or compositional? | Shift-share | manual |

Oaxaca-Blinder splits a mean gap into an explained (composition, X) and unexplained
(returns to X, β) component — the unexplained piece is often loosely called
"discrimination" in wage-gap work, but it really just captures everything the included
covariates don't explain. It depends on which group's coefficients serve as the reference
(the index-number problem); report both directions or use a pooled reference and note the
sensitivity. Gelbach's decomposition is the order-invariant alternative to adding controls
one at a time and eyeballing coefficient movement — use it whenever the question is "which
control matters most for closing this gap."

## Index Construction

Normalize before combining (z-score for roughly symmetric inputs, rank-based for skewed
ones), and align directionality first — mixing "higher = better" and "higher = worse"
components silently breaks an index. `pca` in Stata gives first-component loadings if
using PCA-derived weights, but remember the first PC captures *variance*, not necessarily
*importance* — a data-driven weight isn't automatically the theoretically correct one.
Report the variance explained (if <40%, a single index may not represent the construct
well), Cronbach's alpha (`alpha var1 var2 var3`, >0.7 conventional but mechanically
increases with more components), and always show the individual components alongside the
composite so a reader can see what drives any observation's score.

## Weighted Analysis

| Weight type | Purpose | Stata |
|---|---|---|
| Survey/sampling | correct unequal selection probability | `svyset`, then `svy:` prefix |
| Population | scale to population totals | `[pweight = wt]` |
| Frequency | each row represents N identical units | `[fweight = n]` |
| Analytic/precision | inverse-variance weighting | `[aweight = wt]` |
| IPW | correct for non-random selection/attrition | `teffects ipw`, manual `[pweight]` |

Common, consequential errors: applying survey weights to a census file (CCD, IPEDS
universe files are censuses — unweighted is correct there); filtering the data *before*
computing a subgroup estimate from survey data instead of using `svy, subpop()` — filtering
first can drop entire strata/PSUs and understates the resulting standard errors; reporting
a weighted mean with unweighted SEs. If replicate weights are provided (ACS PUMS, CPS
ASEC, MEPS), prefer them over Taylor linearization (`svyset ..., vce(brr)` or
`vce(jackknife)`) — see the survey documentation for the specific replicate-weight
variables.

## Inequality Measurement

| Measure | Range | Property | Stata |
|---|---|---|---|
| Gini | 0–1 | sensitive to the middle; doesn't decompose by subgroup | `ineqdeco`, `sumdist` |
| Theil T / L | 0–∞ | exactly decomposes into between + within group | `ineqdeco` |
| Percentile ratios (90/10, 90/50, 50/10) | — | shows *where* inequality lives | `xtile` + manual ratio |

The Gini can't tell you whether inequality comes from a rich upper tail or a poor lower
tail; percentile ratios can. Theil's exact decomposability answers "how much of national
inequality is between states vs. within states" — a question the Gini can't cleanly
answer.

## Correlation and Association

`pwcorr` (Pearson, linear) vs. `spearman` (rank, monotonic-not-necessarily-linear) — a
much higher Spearman than Pearson signals a real but nonlinear relationship. Partial
correlation (`pcorr`) removes a third variable's linear effect from both X and Y — this is
the Frisch-Waugh-Lovell logic applied to correlation. Always report a confidence interval
alongside r; "r=0.3, 95% CI [0.05,0.52]" and "r=0.3, CI [0.28,0.32]" are very different
claims collapsed into the same point estimate. Correlation is not causation — use "is
associated with," not "affects" or "drives," unless the identification strategy in
`causal-inference.md` earns the causal verb.

## Missing Data

| Mechanism | Depends on | Safe handling |
|---|---|---|
| MCAR | nothing observed or unobserved | listwise deletion unbiased, just less efficient |
| MAR | observed variables only | imputation using observed predictors (`mi impute`) |
| MNAR | the missing value itself | no clean fix; bound the bias, don't pretend it's solved |

No test distinguishes MAR from MNAR (you can't observe what's missing). Gather evidence
instead: compare observed characteristics of complete vs. incomplete cases
(`ttest x, by(missing_flag)`), check whether variables are missing together
(`misstable patterns`), and lean on domain knowledge about *why* something would be
missing. Report, for every analysis: missingness rate per variable used, whether patterns
co-occur, the assumed mechanism and why, how it was handled, and how much conclusions
would shift under an alternative assumption. Above ~15% missingness on a key variable,
treat results as assumption-sensitive and say so; above ~30%, that variable generally
shouldn't be a primary outcome or covariate.

## Sample Description

State explicitly: target population, sampling frame, sample, and analytic sample after
exclusions — each transition can introduce selection. Compare sample characteristics to a
known external benchmark when one exists (Census/ACS for demographics, NCES/IPEDS universe
counts for institutions) — systematic gaps don't necessarily invalidate the analysis, but
must be disclosed. Report a sample-flow table showing N lost at each exclusion step:

```
Full dataset:                  N = 100,000
Excluding territories:         N = 98,500
Requiring enrollment data:     N = 95,200
Requiring poverty data:        N = 91,800  ← final analytic sample
```

For longitudinal data, watch survivorship bias: apparent improvement among "stayers" can
simply reflect attrition of worse-performing units. Ask explicitly who dropped out and
whether that could produce the pattern you're seeing.

## Further Reading

Angrist & Pischke, *Mostly Harmless Econometrics*; Cunningham, *Causal Inference: The
Mixtape*; Fortin, Lemieux & Firpo (2011) "Decomposition Methods in Economics" (*Handbook
of Labor Economics*); Oaxaca (1973), Blinder (1973), Gelbach (2016) for decompositions;
Solon, Haider & Wooldridge (2015) "What Are We Weighting For?"; Little & Rubin, *Statistical
Analysis with Missing Data*; Cowell, *Measuring Inequality*.
