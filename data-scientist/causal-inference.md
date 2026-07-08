# Causal Inference Methodology

When and why to use each causal design, the assumptions behind it, and how to judge
whether a causal claim is credible. For Stata implementation, load the `stata` skill.
Draws on Cunningham (*Causal Inference: The Mixtape*), Huntington-Klein (*The Effect*),
Angrist & Pischke, and Sant'Anna's DiD lecture series — cite these directly, not this file.

## The Fundamental Problem

We only ever observe one potential outcome per unit — a district either got the funding
increase or didn't (Holland 1986). The counterfactual is unobservable by logical
necessity, not a data limitation. Every causal method is a different strategy for
approximating that missing counterfactual: randomization (statistical equivalence by
design), regression/matching (condition on observables), IV (isolate exogenous variation),
RD (exploit a known cutoff), DiD (exploit parallel trends), FE (exploit within-unit
variation), synthetic control (weighted combination of donors).

**Potential outcomes notation:** Y_i(1), Y_i(0) are outcomes under treatment/control;
D_i is the treatment indicator; observed Y_i = D_i·Y_i(1) + (1−D_i)·Y_i(0). Key estimands:
ATE = E[Y(1)−Y(0)], ATT = E[Y(1)−Y(0)|D=1], LATE = effect for IV compliers, CATE =
effect conditional on X. The naive comparison E[Y|D=1]−E[Y|D=0] equals ATE only when
selection bias and heterogeneous-effect bias are both zero — which randomization
guarantees and nothing else does.

**DAGs**, as a complementary framework: chains (A→B→C, B mediates), forks (A←B→C, B
confounds — condition on it), colliders (A→B←C, conditioning on B *creates* spurious
association — do not condition on it). The back-door criterion: close all non-causal paths
from treatment to outcome by conditioning on the right variables, while leaving the causal
path open. For graphs beyond ~6 nodes, use dagitty.net to find a valid adjustment set. A
DAG determines what a method needs to condition on; potential outcomes define what
parameter you're estimating.

**Identification hierarchy** (internal validity only, not generalizability): RCT >
quasi-experimental (IV/RD/DiD) > observational with controls (relies on conditional
independence — all confounders observed) > naive comparison.

## Method Selection

| Design | Use when | Key assumptions | Estimates | Main threat | Stata |
|---|---|---|---|---|---|
| RCT | can randomize | random assignment, SUTVA, no attrition | ATE | attrition, non-compliance | `regress`, `ttest` |
| Regression/matching | selection on observables plausible | conditional independence (CIA) | ATE if CIA holds | unobserved confounders | `regress`, `teffects` |
| Matching / IPW | CIA + need nonparametric robustness | CIA + common support | ATT / ATE | overlap violations | `teffects psmatch`, `teffects ipw` |
| Fixed effects / panel | repeated obs, time-invariant confounders | strict exogeneity | within-unit ATE | time-varying confounders | `xtreg, fe`, `reghdfe` |
| Instrumental variables | endogenous treatment, valid instrument | relevance, exclusion, monotonicity | LATE (compliers) | weak instrument, exclusion violation | `ivregress 2sls`, `ivreghdfe` |
| Regression discontinuity | treatment assigned by a score cutoff | continuity at cutoff, no manipulation | LATE at cutoff | sorting, bandwidth sensitivity | `rdrobust`, `rddensity` |
| Difference-in-differences | policy hits some units, not others, over time | parallel trends, no anticipation | ATT | parallel-trends violation | `csdid`, `did_imputation`, `xtdidregress` |
| Synthetic control | few treated units, long pre-period | weighted donors match pre-trend | ATT for treated unit | poor donor fit | `synth`, `synth_runner` |

**The "furious five" progression** (Angrist & Pischke): RCT → regression → IV → RD → DiD —
each solves the previous method's limitation but introduces its own assumption. Pick the
method that matches the actual source of variation in your data; don't pick a method and
hope the assumptions hold.

### Instrumental Variables

First-stage strength is the first check: F > 10 is the classic Staiger-Stock threshold,
modern practice prefers F > 100 or weak-instrument-robust inference (`ivreghdfe` reports
the Kleibergen-Paap F for clustered/robust SEs — don't rely on the naive F). Below F=10,
stop and reconsider the instrument; 2SLS is biased toward OLS and CIs are unreliable.
Consider `liml` as less biased under weak instruments. The exclusion restriction (the
instrument affects Y *only* through treatment) cannot be tested — it needs a theoretical
defense. With multiple instruments, `estat overid` (Sargan/Hansen J) tests whether
instruments agree with each other, not whether any one is valid. Monotonicity (no
"defiers") is a substantive, context-specific assumption for the LATE interpretation.

### Regression Discontinuity

Sharp RD: treatment is a deterministic function of the running variable crossing a cutoff
(estimates ATE at the cutoff). Fuzzy RD: the cutoff shifts the *probability* of treatment
— essentially IV with the cutoff as instrument (estimates LATE for compliers at the
cutoff). Use `rdrobust` for bias-corrected, robust CIs with data-driven bandwidth
(Calonico-Cattaneo-Titiunik); report results at 0.5×, 1×, 2× the optimal bandwidth to show
robustness. Prefer local linear (order-1 polynomial) over higher-order polynomials —
higher orders overfit and extrapolate erratically near the cutoff (Gelman & Imbens 2019).
Always run `rddensity` (McCrary-style manipulation test) first: if the running variable's
density is discontinuous at the cutoff, units are sorting around it and the design is
compromised — this is a hard stop on the whole design, not a caveat to footnote. Standard
RD plot: `rdplot`, binned means with fitted curves either side of the cutoff.

### Matching and IPW

Both are selection-on-observables strategies assuming CIA: conditional on X, treatment is
independent of potential outcomes. `teffects psmatch` (nearest-neighbor on the propensity
score), `teffects ipw`, `teffects aipw` (doubly robust — consistent if *either* the outcome
or the propensity model is right). After matching, check balance
(`tebalance summarize`/`pstest`): standardized differences should be <0.1 (ideally <0.05);
if balance fails, the matching failed — don't proceed to outcomes. Check common support
(`teffects overlap`): propensity scores near 0/1 mean treatment is nearly deterministic for
those units; trim and report how many observations were dropped. Matching cannot fix
unobserved confounding — pair it with sensitivity analysis (Rosenbaum bounds via
`rbounds`, or Oster 2019).

### Synthetic Control

For few treated units (often one) with a long pre-period: construct a weighted combination
of donor units (`synth` or `synth_runner`) that closely tracks the treated unit's
pre-treatment trajectory; the post-period gap is the effect. Restrict the donor pool to
plausible comparators — irrelevant donors introduce interpolation bias. Pre-treatment fit
quality is the credibility check: poor fit means an unreliable counterfactual, full stop.
Inference is via placebo tests, not t-tests: run the same procedure on every untreated
donor (in-space placebo) and compare the treated unit's gap to that distribution; also try
in-time placebos (fake treatment date pre-period) to confirm no spurious "effect" appears.

## Modern Difference-in-Differences

**Avoid plain two-way fixed effects (`xtreg, fe` / `reghdfe` with a 0/1 treatment dummy)
for staggered adoption** — where units start treatment at different times. TWFE implicitly
uses already-treated units as controls for later-treated units, and some group-time
effects get *negative* weights, so the aggregate can have the wrong sign even when every
underlying effect is positive (Goodman-Bacon 2021; de Chaisemartin & D'Haultfoeuille 2020).
Event-study versions of TWFE inherit the same contamination (Sun & Abraham 2021) — a clean
pre-trend plot from a TWFE event study is not reliable evidence of parallel trends.

Modern estimators separate three steps TWFE conflates: identify each group-time ATT(g,t),
estimate it (outcome regression, IPW, or doubly-robust — DR is generally preferred),
then aggregate transparently into an overall ATT, event-time profile, or group-specific
effects. Report the disaggregated ATT(g,t) alongside any aggregate — aggregation always
discards the heterogeneity, which is often the substantively interesting part.

| Estimator | Stata | Best for |
|---|---|---|
| Callaway-Sant'Anna | `csdid` | general-purpose default; flexible aggregation, doubly robust |
| Gardner two-stage | `did_imputation` (or `did2s` port) | simple imputation approach, stronger homogeneity assumption |
| Sun-Abraham | `eventstudyinteract` | event studies specifically; corrects TWFE contamination |
| Wooldridge ETWFE | `xthdidregress`/manual | regression-based, familiar framework |
| de Chaisemartin-D'Haultfoeuille | `did_multiplegt` | non-absorbing (reversible) treatment |

Start with `csdid` as the default. Pre-trend tests (joint test on pre-period event-study
coefficients) are necessary but not sufficient evidence for parallel trends — failing to
reject may just reflect low power (Roth 2022). Roth & Sant'Anna (2023) show parallel
trends can hold in levels but fail in logs (or vice versa) — check both scales, don't
assume the functional form is innocuous. For honest inference under a possible parallel
trends violation, use the Rambachan & Roth (2023) sensitivity framework (`honestdid`
package) rather than assuming the assumption holds exactly.

## Causal vs. Descriptive — Know Which One You're Doing

Descriptive questions (prevalence, trend, disparity, distribution — "how many," "how has
X changed," "does Z differ across groups") don't need a causal design; see
`descriptive-analysis.md`. The failure mode to avoid: using associational methods but
writing action-oriented conclusions that imply causation ("the program should be
expanded," from a cross-sectional correlation). Either commit to a causal design or state
plainly that the finding is suggestive, not conclusive. Exploring how an *association*
varies across subgroups is descriptive; estimating heterogeneous *treatment effects*
requires a causal design within each subgroup. Know which parameter your design actually
identifies: random-variation designs typically estimate ATE, designs specific to the
treated group estimate ATT, IV estimates LATE for compliers only, RD estimates a local
effect at the cutoff — none of these is automatically "the" population effect.

## Threats to Validity

| Design | Check |
|---|---|
| RCT | balance tables, attrition analysis, ITT vs. LATE |
| Regression/matching | sensitivity analysis (Oster 2019), robustness to specification |
| IV | first-stage F, exclusion restriction argument, monotonicity defense |
| RD | `rddensity` manipulation test, bandwidth/polynomial sensitivity |
| DiD | pre-trend test + Rambachan-Roth sensitivity, event-study plot, composition checks |
| Synthetic control | pre-treatment fit quality, donor restrictions, placebo tests |

External validity needs an explicit argument, not an assumption — LATE and local RD
effects are inherently local to compliers/the cutoff; site- and time-specific effects may
not transfer. On measurement: classical measurement error attenuates coefficients toward
zero (a known, signed bias); non-classical error biases in an unknown direction; check
whether the variable actually measures the intended construct.

## Sensitivity and Partial Identification

When point identification needs assumptions too strong to be fully credible, report
sensitivity bounds rather than asserting a point estimate is exact — this is a baseline
expectation for observational causal claims, not an optional extra:

- **Oster (2019)** — how much selection on unobservables would it take to explain away the
  result, given how much the coefficient and R² move as controls are added (`psacalc`).
- **Cinelli & Hazlett (2020)** — partial-R² sensitivity contours and a robustness value
  summarizing the confounding strength needed to overturn the result.
- **Rambachan & Roth (2023)** — "Honest DiD": bounds the DiD estimate under specified
  degrees of parallel-trends violation rather than assuming it holds exactly.
- **Rosenbaum bounds** (`rbounds`) — for matching estimators, the confounder strength
  (Γ) at which the conclusion would flip.

Reach for these whenever assumptions are debatable (nearly always), stakes are high, or a
referee/policymaker will reasonably question identification.

## Further Reading

Cunningham, *Causal Inference: The Mixtape* (mixtape.scunning.com) · Huntington-Klein,
*The Effect* (theeffectbook.net) · Angrist & Pischke, *Mostly Harmless Econometrics* and
*Mastering 'Metrics* · Sant'Anna, DiD lecture series (psantanna.com/did-resources) ·
Baker, Callaway, Cunningham, Goodman-Bacon & Sant'Anna, "Difference-in-Differences
Designs: A Practitioner's Guide" (*JEL*, forthcoming) · Cattaneo, Idrobo & Titiunik, *A
Practical Introduction to Regression Discontinuity Designs*. Key papers cited inline
above by author-year; look up full citations as needed rather than duplicating a
bibliography here.
