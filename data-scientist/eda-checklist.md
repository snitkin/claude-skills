# EDA Checklist (Stata)

Run these checks before any analysis or transformation on new data.

## Initial Inspection

```stata
describe                       // shape, types, existing labels
codebook, compact              // range, missing, unique values per var
list in 1/5                    // head
list in -5/-1                  // tail — catches footer/truncation issues
preserve
    sample 5, count
    list, clean
restore                        // random sample, avoids bias from sorted data
```

Watch for: dates or numbers stored as `str` (check `describe` types against what the
variable should be), variable names with inconsistent case or embedded spaces (`rename`
issues later), and mixed-type string variables that should be numeric (`destring` will
warn on non-numeric characters).

## Missing Values

```stata
misstable summarize             // count and % missing per variable, only vars with missing
misstable patterns              // which variables tend to be missing together
```

Classify each missingness pattern before deciding how to handle it — the mechanism
determines whether dropping/imputing is safe:

| Pattern | Meaning | Handling |
|---|---|---|
| MCAR | missing completely at random | safe to drop or impute |
| MAR | missingness depends on *observed* data | impute using related variables |
| MNAR | missingness depends on the *unobserved* value itself | needs domain knowledge; dropping biases the result |

Also check for sentinel codes masquerading as valid data (`-99`, `9999`, `"N/A"`, `"."`
typed as a string) — `misstable summarize` won't catch these because Stata doesn't know
they mean missing:
```stata
tab income if income < 0 | income == -99 | income == 9999
```

## Distributions

```stata
summarize, detail               // per numeric var: mean, sd, skewness, quantiles
tabulate category_var, sort     // categorical: frequencies, sorted

* Red flags: mean far from median (skew), sd == 0 (constant column),
* a single category dominating >95% of observations
```

For date/time variables:
```stata
summarize date_var
* Check: does the max value fall in the future? Does the min predate the
* program/dataset's existence? Both usually indicate entry errors.
```

## Outliers

```stata
* IQR method
summarize amount, detail
local lb = r(p25) - 1.5*(r(p75)-r(p25))
local ub = r(p75) + 1.5*(r(p75)-r(p25))
count if amount < `lb' | amount > `ub'
di "Outliers: " r(N) " (" %4.1f (r(N)/_N*100) "%), bounds [" `lb' ", " `ub' "]"

* z-score method
egen z_amount = std(amount)
count if abs(z_amount) > 3
drop z_amount
```

**Never drop outliers automatically.** Examine the flagged rows (`list if ...`),
understand why they're extreme, consult a domain expert if one is available, and document
the decision — including if the decision is to keep them.

## Uniqueness and Granularity

The most important question for any dataset: **what does one row represent?**

```stata
isid id                          // errors out if not unique — that's the useful part
isid id year                     // test composite keys
duplicates report                // exact duplicate rows
duplicates report id year        // duplication on a candidate key specifically
duplicates list id year if _N > 1
```

Cardinality guide for deciding how to treat a variable:

| Unique values | Likely role |
|---|---|
| 1 | constant — consider dropping |
| 2 | binary/indicator |
| 3–~20 | categorical, `tabulate`/`i.var` |
| high, non-integer pattern | continuous measure |
| ≈ row count | identifier, not a feature |

```stata
foreach v of varlist _all {
    quietly levelsof `v', local(lv)
    di "`v': " r(r) " unique values"
}
```

## Correlation

```stata
pwcorr var1 var2 var3, sig star(0.05)   // pairwise correlations with significance
* Flag |r| > 0.7-0.9 pairs as possible multicollinearity before regressing

tabulate cat1 cat2, chi2                 // categorical association + chi-squared
```

## Red Flags Checklist

- [ ] High missingness (>10%) in a variable you plan to use
- [ ] Unexpected types (numeric var stored as `str`, sentinel codes for missing)
- [ ] Constant or near-constant column (one value dominates)
- [ ] Duplicates on the assumed key (`isid` fails)
- [ ] Extreme outliers, unexamined
- [ ] Future dates, or dates before the program/data source existed
- [ ] Negative values where only positive is possible
- [ ] Cardinality mismatch (an "ID-like" column with few unique values, or vice versa)
- [ ] Suspiciously round or uniform distributions
- [ ] Same category spelled multiple ways (`"NY"` vs `"New York"`) — check with `tabulate`
