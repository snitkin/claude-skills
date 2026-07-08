# Statistical Modeling Methodology

Model selection, assumption checking, robust inference, and interpretation — the *why* and
*when*. For Stata syntax (`regress`, `reghdfe`, `logit`, `poisson`, `margins`), load the
`stata` skill. Draws on Angrist & Pischke (*Mostly Harmless Econometrics*), Cunningham
(*Causal Inference: The Mixtape*), and Wooldridge (*Econometric Analysis of Cross Section
and Panel Data*) — cite these directly rather than this file.

Related: causal identification strategies → `causal-inference.md`; causal vs. correlational
language → `research-questions.md`; regression as a purely descriptive tool →
`descriptive-analysis.md`.

## Model Selection by Outcome Type

| Outcome | Primary model | Stata command | Note |
|---|---|---|---|
| Continuous | OLS | `regress`, `reghdfe` | check heteroskedasticity, functional form |
| Binary | Logit / LPM | `logit`, `regress` | LPM gives directly interpretable AME; mind [0,1] boundary |
| Count | Poisson | `poisson`, `ppmlhdfe` | Poisson QMLE is consistent for the mean even if variance is misspecified (Wooldridge 2010) |
| Unordered categorical | Multinomial logit | `mlogit` | requires IIA |
| Ordered categorical | Ordered logit/probit | `ologit`, `oprobit` | requires proportional odds |
| Panel | FE / RE | `xtreg, fe`, `reghdfe`, `xtreg, re` | choose on research design, not just Hausman |

**Start simple.** Begin with OLS/LPM even for a binary outcome — coefficients are directly
interpretable as probability changes, and with robust SEs inference is reliable away from
the [0,1] boundary. Add complexity (logit, Poisson, splines) only when diagnostics or the
question demand it — e.g., marginal effects near the boundary matter, or the count
structure itself is of interest. Let theory and diagnostics drive specification; running
many specifications and reporting the best-looking one is specification searching (Leamer
1983), not model selection.

**FE vs. RE vs. CRE:** FE absorbs all time-invariant unobserved heterogeneity — the safer
default whenever unobserved unit characteristics might correlate with the regressor of
interest. RE is more efficient but biased if that independence assumption fails. Correlated
random effects (Mundlak 1978: `xtreg, re` plus group-mean controls) nests FE inside RE, and
lets you test the RE assumption while still recovering effects of time-invariant variables.

## Regression as CEF Approximation

Regression approximates the conditional expectation function E(Y|X), not a literal model
of reality (Angrist & Pischke ch. 3). You don't need to believe the world is linear — you
need the linear approximation to be adequate in the region you care about. Any Y
decomposes as E(Y|X) + ε, with ε mean-zero and orthogonal to any function of X; that
orthogonality is what makes coefficients interpretable as approximations to the CEF.

**"Controlling for" mechanically** (Frisch-Waugh-Lovell): the coefficient on X1 in a
multivariate regression equals the coefficient from regressing Y on the *residual* of X1
after partialling out the other covariates. Controlling for X2 isolates variation in X1
orthogonal to X2 — which is also why adding highly correlated controls inflates standard
errors (little residual variation left), and why fixed effects work (they partial out group
means, leaving within-group variation).

**Bad controls:** never control for a post-treatment variable — one the treatment itself
affects. Classic case: estimating a gender wage gap while controlling for occupation, when
gender affects occupational sorting. The test: could the treatment plausibly affect the
control? If yes, it likely absorbs part of the effect you're trying to measure. When in
doubt, show results with and without the questionable control.

**Saturated models:** with a full set of group dummies, the regression *is* the CEF — the
fitted values are just group means, no functional-form approximation needed. This is why FE
and treatment×group interactions let you estimate heterogeneous effects without worrying
about functional form.

## Assumption Checking

| Assumption | Diagnostic | If violated |
|---|---|---|
| Linearity | `rvfplot`, `ovtest` (RESET) | polynomials, splines, transform |
| Independence | time series → autocorrelation; spatial → Moran's I | cluster SEs, or model the structure |
| Homoscedasticity | `estat hettest`, `rvfplot` fan shape | `vce(robust)` |
| Normality of residuals | `qnorm`, `swilk` | irrelevant for N > ~50 (CLT); bootstrap for small N |
| No perfect multicollinearity | `estat vif` (>10 is a concern) | drop redundant vars, build an index, or accept wider SEs |
| Influential outliers | `predict cooksd, cooksd`; leverage | winsorize, robust regression, report both ways |

Read `rvfplot` first — a fan shape means heteroskedasticity (use robust SEs); a curve means
nonlinearity (add flexibility); clusters of residuals suggest a missing group effect
(consider FE or clustering). Only check normality if N is small; the CLT makes it moot
otherwise. Run diagnostics in that order — linearity/heteroskedasticity, then
multicollinearity, then influence, then normality/independence if the data structure calls
for it — and fix what's found before interpreting coefficients.

**Missing data:** listwise deletion (Stata's default) is defensible when missingness is
plausibly unrelated to the outcome conditional on covariates and drops under ~5–10% of the
sample. Always report full-sample N vs. analytic-sample N, and compare means of key
variables across the two (`mean var if missing_flag == 0` vs. `== 1`). If they diverge, say
so — don't silently drop and move on. Beyond that threshold, consider multiple imputation
(`mi impute`); for MCAR/MAR/MNAR framing see `descriptive-analysis.md`.

**Sample size heuristics** (rules of thumb, not hard cutoffs): OLS wants roughly 10–20
observations per regressor; logistic regression wants ~10 events per variable (below that,
`logit, firth` via the `firthlogit` package instead of plain MLE); interactions
multiplicatively increase the N required to detect anything. When N is marginal, prefer
`vce(hc3)` over `hc1`, keep the specification simple, and lead with confidence intervals
rather than p-values — a null result may just reflect low power.

**Multiple testing:** a single pre-specified primary outcome needs no correction. Multiple
outcomes or subgroups need one (Benjamini-Hochberg for exploratory work; Bonferroni is valid
but conservative). Robustness checks of the *same* hypothesis are not multiple tests — the
coefficient should be qualitatively stable across them, not independently significant in
each.

## Standard Errors

| Type | When | Stata |
|---|---|---|
| Classical | rarely — requires iid errors | `regress y x` |
| Robust (HC1) | default for cross-sectional data | `regress y x, vce(robust)` |
| HC2/HC3 | small samples (N<50) | `vce(hc2)` / `vce(hc3)` |
| Cluster-robust | correlated within groups, ≥50 clusters | `vce(cluster groupvar)` |
| Few-cluster correction | 20–49 clusters | `reghdfe ..., vce(cluster g)` (uses CRV3-style small-sample adjustment) |
| Wild cluster bootstrap | <20 clusters | `boottest` package |
| HAC / Newey-West | time series, serial correlation | `newey y x, lag(#)` |

**Choosing the cluster level is a design question, not a p-hacking knob** (Abadie, Athey,
Imbens & Wooldridge 2023): cluster where treatment is assigned (state-level policy → cluster
by state) or where a common shock hits a group (students in the same school). When in doubt,
cluster at the higher level — too coarse is conservative, too fine is anti-conservative, and
conservative is the safer error. Below ~50 clusters, standard `vce(cluster)` under-rejects
too rarely — don't trust it; move down the table toward wild cluster bootstrap. For
multi-way clustering (e.g., school and cohort simultaneously), use `reghdfe` with
`vce(cluster school cohort)` (Cameron-Gelbach-Miller 2011 multiway formula).

Estimate the model once, then compare SEs under different assumptions (`estat vce`, or
re-display with different `vce()` options) rather than re-running the regression — the
coefficients don't change, only the inference does, and showing several side by side reveals
whether "significance" depends on the SE assumption.

## Interpreting Coefficients

| Specification | β means |
|---|---|
| Y ~ X | 1-unit ΔX → β-unit ΔY |
| log(Y) ~ X | 1-unit ΔX → ≈(β×100)% ΔY *for small β* |
| log(Y) ~ log(X) | 1% ΔX → β% ΔY (elasticity) |
| Y ~ log(X) | 1% ΔX → β/100-unit ΔY |
| logit | ΔX → β log-odds; use `margins, dydx(x)` for probabilities |
| Poisson | ΔX → multiplies E(count) by exp(β) |

The log-linear "times 100 percent" shortcut only works for \|β\| < ~0.1 — otherwise use the
exact (exp(β)−1)×100. This matters most for dummy variables (e.g., a treatment indicator in
a log-wage regression), where β is often large enough that the approximation is wrong
(Kennedy 1981; Halvorsen & Palmquist 1980 formula for exact dummy-variable percentage
effects in log models).

**Marginal effects for nonlinear models:** compute average marginal effects (`margins,
dydx(*)`), not effects at the mean (`margins, atmeans`). AME averages the effect over the
actual covariate distribution, avoiding a nonsensical "average individual" (e.g., averaging
over a 0/1 gender variable) and matching the policy-relevant quantity better. For
interactions in nonlinear models, the interaction coefficient itself is not the interaction
effect on the outcome scale — compute it via `margins` (Ai & Norton 2003 show the sign can
even differ from the raw coefficient's sign).

**Report effect sizes, not just p-values.** "A 1-SD increase in per-pupil spending is
associated with a 0.08-SD increase in test scores" says more than "p < 0.05." Cohen's
d ≈ 0.2/0.5/0.8 are conventional small/medium/large benchmarks, but Cohen himself warned
against applying them mechanically — a 0.1 SD effect can be highly policy-relevant at scale
in education or public economics. Lead with confidence intervals; they show magnitude and
precision in one display, which a significance star does not.

## Robustness Checks

A result surviving only one specification is fragile; one surviving several reasonable
alternatives is credible. Vary the choices a skeptical referee would question, and report
all of them — never only the specifications that "work":

- **Controls**: short (minimal) → long (extended) regression, added progressively. Large
  coefficient movement when adding controls could mean you removed omitted-variable bias
  (good) or introduced a bad control (bad) — the pattern of movement is informative, not
  just the endpoint.
- **Functional form**: linear vs. log vs. polynomial/spline. Conclusions that only hold
  under one functional form are fragile.
- **Variable definitions**: alternative cutoffs or time windows (e.g., poverty at 100% vs.
  150% FPL).
- **Subsamples**: demographic groups, geography, early vs. late period. A result
  concentrated in one subgroup is informative (real heterogeneity) but should be flagged,
  not hidden — especially if that subgroup was chosen after seeing the data.
- **Alternative estimators**: OLS vs. robust regression (outlier sensitivity), OLS vs.
  quantile regression (`qreg`, effects across the distribution), FE vs. Mundlak CRE.

**Sensitivity to omitted variables** — "how much confounding would it take to overturn
this?":
- **Oster (2019)**: compares how the coefficient *and* R² move as controls are added
  (`psacalc` package). Large R² movement with small coefficient movement → unobservables
  are unlikely to overturn the result. Requires specifying R_max (Oster suggests
  min(1, 1.3×R² of the fully controlled model) — a calibrated, not derived, choice).
- **Cinelli & Hazlett (2020)**: partial-R² sensitivity analysis and a single robustness
  value summarizing how strong a confounder would need to be (`sensemakr` — R package, no
  native Stata port; run via `rsource`/RStata bridge if needed, or report qualitatively).
- Using both is standard practice; they decompose sensitivity differently and are
  complementary rather than redundant.

Present robustness in a specification table (`esttab`/`estout`, columns = specifications,
rows = coefficient/SE) — baseline, preferred, extended controls, alternative functional
form, alternative sample. If the coefficient of interest flips sign or vanishes in any
defensible specification, that instability is the headline finding for that check — state
it plainly rather than burying it.
