# Geospatial Operations Guide

Practical decisions and pitfalls for executing spatial operations — more granular than
`geospatial-analysis.md`'s conceptual overview. Read that file first for when/why; this
file for how to avoid the specific ways spatial operations go silently wrong.

**Scope note:** geometry-level operations — spatial joins, polygon overlay, geometry
validity repair, interpolation/kriging, zonal statistics on rasters — are GIS operations
that **Stata does not natively perform**. If a task needs these, say so plainly and route
to geopandas/PySAL (Python), `sf`/`terra` (R), or QGIS rather than forcing it through
Stata. What Stata *does* handle natively — spatial weights construction and spatial
regression — is covered here and in `geospatial-analysis.md`.

## Spatial Joins (if working outside Stata)

Choosing the predicate matters: `intersects` is the permissive default; `within` is
conventional for point-in-polygon assignment (a point strictly inside, avoiding boundary
ambiguity); `nearest` for "closest feature" questions. Spatial joins are inherently
many-to-many in a way tabular joins aren't — a point on a shared boundary can match
multiple polygons, overlapping source polygons can multiply matches. After every spatial
join: compare row counts before/after, check for unexpected duplication of the left table,
check for nulls in joined columns (unmatched rows), and spot-check a handful of matches on
an actual map, not just in a table. **Both layers must share a CRS before joining** — a
mismatch doesn't error, it silently assigns points to the wrong polygon or produces zero
matches.

## Geometry Validity (if working outside Stata)

Invalid geometries (self-intersecting "bowtie" polygons, unclosed rings, duplicate
vertices) are the most common cause of silent failures in overlay and join operations —
often no error, just wrong or empty output. Check validity before any overlay, join with
`contains`/`within`, or transformation, especially after reading Shapefiles (prone to this).
Repair with `make_valid` where available (more predictable than the classic "buffer by
zero" trick, which can merge components that should stay separate) — then re-validate
after repair, since some fixes introduce new problems.

## Spatial Weights Construction

This is the one piece of "geospatial operations" squarely inside Stata's native
toolkit (`spmatrix`, `spatwmat`), and it deserves as much care as any other modeling
choice — every Moran's I, LISA map, and spatial regression result depends on it.

| Type | Logic | Best for |
|---|---|---|
| Queen contiguity | shares an edge or vertex | regular polygons (tracts, grid cells) |
| Rook contiguity | shares an edge only | when diagonal adjacency isn't meaningful |
| K-nearest neighbors | k closest centroids | points, or guaranteeing every unit has neighbors |
| Distance band | all units within threshold d | when a specific proximity range is substantively meaningful |

Most applications row-standardize (each row sums to 1, so the spatial lag is a weighted
average of neighbors). Distance-based weights (KNN, distance band) require a **projected**
CRS — computing "distance" in lat/lon degrees is meaningless since a degree of longitude
isn't a constant length. Contiguity weights are less CRS-sensitive but project anyway as
good practice. Because the weights choice affects every downstream statistic, re-run with
at least one alternative specification (e.g., KNN k=6 vs. k=10) and report whether
conclusions hold — if they don't, report both and say which is more defensible for the
question, don't just pick the one that "worked."

**Islands** (observations with no neighbors under the chosen weights) break spatial lag
computation and can crash or destabilize spatial regression estimators — KNN weights
guarantee connectivity and sidestep this; contiguity/distance-band weights may need an
increased threshold or documented exclusion of the isolated units.

## Interpreting Moran's I and LISA Results — Write-Up Template

Report all of: the statistic, the inference (permutation p-value and number of
permutations), direction (positive = clustering, negative = dispersion), magnitude in
context (for most socioeconomic variables, I≈0.2–0.5 is moderate, >0.5 is strong), and the
weights specification used — this last one is easy to omit and is essential metadata.

> Poverty rates exhibit significant positive spatial autocorrelation (Moran's I = 0.47,
> p < 0.001, 999 permutations, Queen contiguity), indicating high-poverty counties tend to
> neighbor other high-poverty counties.

For LISA, report counts by quadrant (HH/LL/HL/LH) at a stated significance threshold,
describe the geographic pattern, and flag caveats: multiple testing across many locations,
boundary effects (edge units have fewer neighbors), and weights sensitivity. Common
interpretation errors to avoid: reading Moran's I as a percentage or R² (it isn't one);
asserting an HH cluster "causes" high values (LISA shows pattern, not mechanism); treating
a non-significant Moran's I as proof of no spatial pattern (absence of evidence isn't
evidence of absence); and reporting LISA results without first filtering to the
significant locations.

## Spatial Regression Diagnostics Write-Up

> OLS residuals showed significant spatial autocorrelation (Moran's I = 0.31, p < 0.001,
> Queen contiguity). LM diagnostics favored the spatial error specification (LM-error =
> 24.7, p < 0.001; robust LM-error = 18.2, p < 0.001) over spatial lag (LM-lag = 8.3,
> p = 0.004; robust LM-lag = 1.8, p = 0.18). The SEM (λ = 0.38, p < 0.001) resolved
> residual autocorrelation (Moran's I on SEM residuals = 0.02, p = 0.41); the coefficient
> of interest moved from 0.42 (OLS) to 0.51 (SEM), consistent with OLS attenuating the
> estimate by absorbing spatially structured variation into the error. Results were robust
> to alternative weights (KNN k=6, k=10).

Always report both the non-spatial (OLS) and spatial results side by side, the weights
specification, and diagnostics confirming the spatial model actually resolved the
residual autocorrelation — a spatial model that still shows significant residual Moran's I
didn't fix the problem it was fit to fix.

## Areal Interpolation — the Extensive/Intensive Distinction (if working outside Stata)

When transferring data across mismatched boundaries (e.g., census tracts to school
districts), the single most common error is treating an intensive variable like an
extensive one. **Extensive** variables (population counts, total spending) scale with
area — multiply by the area-overlap fraction. **Intensive** variables (rates, densities,
percentages) don't scale with area — use an area-weighted *average*, never a sum. Summing
a poverty rate across overlapping sub-polygons produces a meaningless number. Simple
area-weighting also assumes the variable is uniformly distributed within the source
zone (false for population — nobody lives in the lake portion of a tract); dasymetric
methods that incorporate land-use/building-footprint data as auxiliary weights are more
accurate when available. After interpolating, check that extensive-variable totals are
conserved (source sum ≈ target sum) as a basic sanity check.

## Further Reading

Rey, Arribas-Bel & Wolf, *Geographic Data Science with Python* · Anselin (1995) "Local
Indicators of Spatial Association" · Anselin & Rey (2014), *Modern Spatial Econometrics in
Practice* · Goodchild & Lam (1980) on areal interpolation.
