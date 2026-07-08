# Geospatial Analysis Methodology

When and how to think about spatial data — code-agnostic concepts. For Stata syntax load
the `stata` skill. Draws on Rey, Arribas-Bel & Wolf, *Geographic Data Science with Python*
(open access, geographicdata.science/book) and Dorman et al., *Geocomputation with Python*
(py.geocompx.org) for concepts; cite these directly.

**Stata coverage, stated plainly:** `spmap` handles choropleth mapping; `spmatrix`/
`spatwmat` build spatial weights; `spregress`/`spivregress`/`spxtregress` (Stata 15+)
cover spatial lag/error regression; `estat moran` tests residual autocorrelation after
spregress. Point pattern analysis (Ripley's G/F, KDE for events), raster operations, and
geographically weighted regression have **no strong native Stata support** — if the task
genuinely needs these, tell the user plainly rather than approximating with weaker tools,
and consider whether geopandas/PySAL (Python) or `sf`/`spatstat` (R) are the better fit for
that specific step.

## Spatial Thinking

Tobler's First Law — "near things are more related than distant things" — has a testable
implication: spatial autocorrelation. If it holds for a variable, standard regression's
independence assumption is violated: OLS SEs are too small, CIs too narrow, p-values too
optimistic. Reach for spatial methods when the question is inherently about place ("do
poverty rates cluster geographically?"), when geography is a plausible confounder (urban/
rural differences confounding a class-size effect), or at minimum adjust inference
(cluster SEs by geography, Conley spatial-HAC SEs, or test residuals for spatial
autocorrelation) whenever observations might be spatially correlated even if space itself
isn't the question.

**MAUP** (Modifiable Areal Unit Problem): results change with both the scale of
aggregation (county vs. tract vs. state) and how boundaries are drawn at a fixed scale —
this is a property of aggregated spatial data, not a bug in any particular analysis. State
the geographic unit explicitly, acknowledge MAUP sensitivity as a limitation, and test
robustness across aggregation levels when feasible; finer resolution isn't automatically
better if it just adds noise.

**Ecological fallacy**: a county-level correlation between income and test scores doesn't
imply the same relationship holds for individuals — it could be entirely a compositional
effect. Area-level data supports area-level claims only ("districts with higher poverty
have lower graduation rates," not "poor students graduate less").

## CRS and Projections

Getting the coordinate reference system wrong silently corrupts distances, areas, and
spatial joins. Geographic CRS (lat/lon, degrees, e.g. WGS84/EPSG:4326) is for storage and
exchange, not calculation — one degree of longitude is ~111km at the equator and ~0km at
the poles, so distance/area/buffer computed in a geographic CRS is wrong. **Project before
computing anything geometric.** Pick the projection by what you need preserved: equal-area
(Albers, EPSG:5070 for the continental US) for choropleth/thematic mapping; UTM for local-
scale distance accuracy; conformal (Lambert Conformal Conic) for angle/bearing work; Web
Mercator (EPSG:3857) for display only, never analysis. "Setting" a CRS declares what the
existing coordinates already are (no numbers change); "transforming"/reprojecting
recomputes coordinates into a new system (numbers change) — confusing the two silently
plots data in the wrong place. Geocoded coordinates rarely warrant more than 4–5 decimal
places (~11m precision); census tract/block-group centroids are good to ~3–4 — don't carry
14 decimal places from a geocoding API as if that implied sub-atomic precision.

## Spatial Data Types

Vector (points/lines/polygons — Simple Features standard) for discrete, boundary-defined
entities (schools, districts, parcels); raster (regular grids) for continuous phenomena
without natural boundaries (elevation, temperature, satellite imagery) or when the source
data is inherently gridded. Prefer GeoPackage or GeoParquet over Shapefile for anything new
(Shapefile has a 2GB limit and 10-character column-name truncation that silently mangles
variable names).

## Method Selection

| Question | Method |
|---|---|
| Where are things concentrated? | point pattern analysis — KDE, Ripley's G/F (weak Stata support; consider R/Python) |
| Is there spatial clustering overall? | global Moran's I / Geary's C |
| Where specifically are the clusters? | LISA (local Moran's I), Getis-Ord Gi* |
| Combining spatial layers | spatial join / overlay |
| Transferring data across incompatible boundaries | areal interpolation |
| Does space affect my regression? | spatial regression diagnostics → model choice (below) |
| Values at unobserved locations | interpolation (IDW, kriging) |
| Summarizing raster within polygons | zonal statistics |

## Spatial Autocorrelation

Global Moran's I is the standard summary: correlates a variable with its spatial lag
(neighbors' weighted average). Positive I → similar values cluster; near zero → spatial
randomness; negative → checkerboard-like dissimilarity (rare). Inference is by permutation
(shuffle values across locations ~999 times, compare to the observed statistic), not a
parametric formula. Results are sensitive to the weights matrix choice (k-nearest-neighbor
vs. distance band vs. contiguity) — report which was used and check sensitivity to
alternatives; human pattern-recognition on a choropleth is prone to false positives, so
this formal test matters even when a map "looks" clustered.

**LISA** (local Moran's I) decomposes the global statistic by location, classifying each
observation into HH (hot spot: high value, high neighbors), LL (cold spot), or the spatial-
outlier quadrants LH/HL. Report the quadrant classification, the significance from
permutation testing, and a cluster map showing only significant locations — a raw local-I
choropleth alone is not sufficient. Watch for the rate problem: local Moran's I on a rate
(events/population) with a varying denominator needs Empirical Bayes smoothing, and testing
many locations simultaneously needs a multiple-testing adjustment or cautious
interpretation.

## Spatial Regression

Diagnose before modeling: map the residuals from a standard regression, then test them
formally (`estat moran` after `regress`/`spregress`, or Moran's I on saved residuals). If
residuals show spatial structure, work through this progression rather than jumping
straight to the most complex model:

1. **Spatial feature engineering** — distance-to-amenity, neighbor averages as covariates,
   estimable with plain OLS. Try this first; it often resolves the problem without any
   spatial estimator.
2. **Spatial fixed effects** (county/state dummies) — absorbs unobserved factors constant
   within the spatial unit; standard in applied economics, no spatial software needed.
3. **Spatial regimes** — let intercepts/slopes vary by region; test with a Chow test.
4. **SLX** (spatial lag of X) — neighbors' X as a regressor; separates direct from
   indirect/spillover effects; still OLS-estimable since lagged X is exogenous.
5. **Spatial error model (SEM)** (`spregress ..., errorlag(W)`) — correlated errors across
   neighbors; OLS point estimates stay unbiased but SEs are wrong without proper
   estimation; appropriate when spatial correlation is a nuisance, not the object of study.
6. **Spatial lag model (SAR)** (`spregress ..., dlag(W)`) — neighbors' *outcome* as a
   regressor; creates endogeneity requiring IV/ML estimation; coefficients aren't direct
   marginal effects because of feedback — use `estat impact` (direct/indirect/total
   effects) rather than reading the raw coefficient.

Use Lagrange Multiplier tests (`estat lmerror`, `estat lmlag` post-regress, or the LM
battery in `spregress` diagnostics) to choose between SAR and SEM: if only LM-lag is
significant, use SAR; if only LM-error, use SEM; if both, compare the *robust* versions and
prefer whichever has the larger robust statistic; if neither is significant, OLS is
adequate even if global Moran's I flagged something — the LM tests are more specific about
the source. GWR is exploratory only (no native Stata command) — useful for spotting *where*
relationships vary before a confirmatory spatial-regimes model, not for hypothesis testing
itself.

## Map Design

**Never choropleth raw counts** — a county-level map of "total enrollment" is really a
population-size map; large counties visually dominate regardless of the phenomenon. Use
choropleths only for rates, densities, or proportions; use proportional symbols or dot
density for raw counts. Sequential palettes (light→dark) for data with no natural
midpoint; diverging palettes (two hues meeting at a neutral center) only when there's a
genuinely meaningful center (change from baseline, above/below average) — using diverging
colors on data without a real center implies a break that isn't there. 4–7 classes is the
usable range for a reader to distinguish; quantile binning guarantees visual variation but
can create a stark color break between two nearly identical values sitting on either side
of a bin edge — inspect the actual bin edges, don't trust the default. Every map needs a
title, legend with classification scheme noted, scale bar, source/date, and the projection
used (especially if equal-area was chosen for a thematic map).

## Further Reading

Rey, Arribas-Bel & Wolf, *Geographic Data Science with Python* (free, geographicdata.science/book)
· Dorman, Graser, Nowosad & Lovelace, *Geocomputation with Python* (free, py.geocompx.org)
· Anselin (1995) "Local Indicators of Spatial Association" · Openshaw (1984) on MAUP ·
Robinson (1950) on the ecological fallacy · Anselin (2005) on the LM test protocol.
