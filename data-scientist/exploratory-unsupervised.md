# Exploratory Unsupervised Analysis

Discovering structure — groupings, dimensions, typologies — without a pre-specified
outcome. Covers the *why* and *when*; for Stata syntax load the `stata` skill. Draws on
James, Witten, Hastie & Tibshirani (*An Introduction to Statistical Learning*, ch. 12) and
Hennig (2015) on the nature of "true clusters."

**Stata coverage note, stated plainly:** `cluster kmeans`, `cluster kmedians`,
`cluster` hierarchical linkages, and `pca`/`screeplot` are native and solid. Gaussian
mixture models exist via `fmm:` (finite mixture models, Stata 15+) but are less
full-featured than dedicated packages. t-SNE and UMAP have **no native Stata
implementation** — if a nonlinear embedding is genuinely needed for visualization, say so
explicitly to the user rather than approximating it with PCA, and consider Stata's Python
integration (`python script`) or a one-off R/Python step rather than silently switching
tools for the whole project.

## When Unsupervised Analysis Is the Contribution

Unsupervised methods are descriptive — they find patterns, not causes. There's no single
"true" clustering (Hennig 2015): validity depends on the variables chosen, the algorithm,
and the research purpose, not some ground truth the data secretly contains. It's the right
primary tool for typology construction ("what types of districts exist?"), data reduction
(collapsing 50 correlated indicators), subpopulation discovery, and hypothesis generation.
It's a preparatory step when used for dimension reduction before regression or to motivate
subgroup analysis — but keep discovering and confirming separate: patterns found this way
should ideally be checked against held-out data or a follow-up study, not asserted as
findings on the same data that generated them.

## Cluster Analysis

| Algorithm | Best for | Stata |
|---|---|---|
| K-means | large N, roughly spherical/equal-size clusters | `cluster kmeans` |
| K-medians/PAM | robust to outliers, any distance | `cluster kmedians` |
| Hierarchical (Ward, average, complete linkage) | small-medium N, dendrogram, nested structure | `cluster wardslinkage`, `cluster dendrogram` |
| Gaussian mixture | overlapping clusters, soft assignments | `fmm: ...` (finite mixture) |
| DBSCAN/HDBSCAN, spectral | arbitrary shapes, unknown k | no native Stata command — flag to user |

Standardize variables before any distance-based method (`egen z_var = std(var)`) —
unstandardized scales let the largest-magnitude variable dominate the distance metric and
drive the "clusters." Missing data isn't handled natively by clustering commands: impute
first (`mi impute`) or use listwise deletion and state the assumption. Variable selection
is the most consequential researcher choice here — prefer theory-driven inclusion, never
include the outcome variable you'll later regress on cluster membership (that's circular),
and drop or PCA-combine variables correlated above ~0.85.

**Choosing k**: use 2–3 complementary criteria and look for agreement rather than trusting
one. The gap statistic and silhouette width are the strongest general-purpose criteria;
for `fmm`, BIC/AIC give a principled comparison across k and covariance structure; the
elbow method (`cluster stop` in Stata, plotting within-cluster SS) is supplementary only —
it's subjective. If criteria disagree, say so and show results at more than one k rather
than picking whichever looks cleanest.

**Validate on three levels**, not just one:
- *Internal*: silhouette coefficient (Stata: compute manually from `cluster generate` +
  distances, or via `clustermat`) — >0.7 strong, >0.5 reasonable, >0.25 weak structure.
- *Stability* (the one people skip, and shouldn't): bootstrap-resample and re-cluster,
  or split the sample in half and compare solutions with the Adjusted Rand Index. Internal
  indices alone can look great and still be unstable — a solution with high silhouette but
  low stability across resamples is not trustworthy.
- *External* (if a known grouping exists to compare against): Adjusted Rand Index or
  normalized mutual information.

Always use multiple random starts for K-means (25–50 minimum; Stata: `start(random(#))`
repeated) and keep the lowest within-cluster SS solution — a single run can land in a
local optimum.

**Common pitfalls**: over-interpreting clusters as natural kinds rather than one possible
partition given your choices; the "Salsa effect" — cluster profiles that are all
uniformly higher/lower (parallel), which usually means you've stratified a single
continuum rather than found distinct types (check whether profiles cross, not just
differ in level); treating hard cluster assignments as ground truth in later analysis
(see below); and reifying evocative cluster names ("resilient," "at-risk") into causally
coherent groups they were never shown to be.

**Report**: rationale, variables included/excluded and why, standardization and missing-
data handling, algorithm and distance metric, how k was chosen (all criteria values, not
just the winner), validation results (internal + stability, external if applicable),
cluster profile table (means/proportions per cluster plus overall, with n per cluster),
and sensitivity to alternative k/algorithm/variable sets.

**Profiling clusters**: report a standardized-means table (z-scores make cross-variable
comparison possible — a cluster at z=+1.5 on poverty and z=−0.8 on test scores tells a
clear story raw units don't) and cluster sizes; scrutinize any cluster under ~5% of the
sample as a possible artifact. Name clusters off their 2–3 most extreme variables, and
always show the profile table alongside any name so a reader can judge it.

## PCA for Dimension Reduction

(For PCA as an *index-construction* tool with explicit weights, see
`descriptive-analysis.md`. This section is PCA as exploration — understanding variance
structure or reducing dimensionality before clustering/regression.)

```stata
pca var1 var2 var3 var4, components(5)
screeplot                       // visual "elbow" — supplementary only, not primary
estat kmo                       // sampling adequacy before trusting the PCA at all
```

For how many components to keep: parallel analysis (retain components whose eigenvalue
exceeds the 95th percentile from randomly generated data of the same size) is the gold
standard — Stata doesn't have a built-in command, so implement via simulation or flag to
the user that this needs a manual step. The Kaiser criterion (eigenvalue > 1,
`pca, mineigen(1)`) is convenient but **systematically over-extracts components** — don't
rely on it alone. A cumulative-variance threshold (70–90%) is a reasonable practical
fallback. Run PCA on the correlation matrix (standardized variables — Stata's default) not
the covariance matrix, unless variables genuinely share a scale and you want raw variance
differences to drive extraction.

## The Classify-Analyze Problem

**The single most important pitfall here.** Treating cluster assignments (or PCA-derived
categories) as known, error-free categories in a subsequent regression or ANOVA ignores
classification uncertainty and attenuates the estimated association toward zero — often
substantially, with overlapping clusters (documented attenuation as large as 0.5→0.1 in
some applications; Lanza, Tan & Bray 2013). This affects every downstream use: cluster
membership as a regressor, ANOVA comparing clusters on an outcome, subgroup analysis by
cluster, cross-tabs of cluster with anything else.

Mitigate by: using soft/posterior probabilities (from `fmm`) as weights rather than hard
labels wherever the downstream analysis allows it; excluding or flagging ambiguous
observations (max posterior <0.7) and showing the result is robust either way; bootstrapping
the clustering *and* the downstream analysis together rather than treating the cluster
solution as fixed; and, if the typology itself is the contribution, presenting it
descriptively rather than as a formal regression predictor. Never use the same variable as
both a clustering input and the downstream outcome — that guarantees a spurious
association by construction.

## Causal Language for Unsupervised Results

These methods are descriptive; they cannot support causal claims on their own, and the
evocative feel of a "discovered" cluster or component makes it tempting to overreach:

| Don't | Do |
|---|---|
| "Students clustered as high-risk *because of* low SES" | "high-risk cluster was *characterized by* low SES" |
| "The resilient profile *caused* better outcomes" | "students *classified in* the resilient profile *tended to have* better outcomes" |
| "PCA *revealed* three underlying dimensions" | "PCA *identified* three components that *explained X% of variance*" |
| "Cluster membership *predicts* dropout" | "cluster membership was *associated with* differential dropout rates" |

Different variable sets, standardization choices, algorithms, and k all change the result
— this is inherent to the method, not a flaw to hide. Document each choice and, where
feasible, show the substantive conclusion survives reasonable alternatives.

## Further Reading

James, Witten, Hastie & Tibshirani, *An Introduction to Statistical Learning* (free,
statlearning.com) ch. 12 · Hastie, Tibshirani & Friedman, *The Elements of Statistical
Learning* ch. 14 · Hennig (2015) "What are the true clusters?" · Tibshirani, Walther &
Hastie (2001) on the gap statistic · Lanza, Tan & Bray (2013) on classify-analyze bias ·
Fraley & Raftery (2002) on model-based clustering.
