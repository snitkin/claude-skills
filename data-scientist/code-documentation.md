# Code Documentation (Stata)

How to comment do-files so the reasoning survives, not just the syntax.

## The Core Principle

**Code shows HOW. Comments carry WHY, WHAT FOR, and WHAT'S ASSUMED.**

A reader can already see that you ran `merge 1:1 id year using controls.dta`. What they
can't see: why 1:1 rather than m:1, why `id year` is the right key, what you expect the
match rate to be, and what you'll do about the unmatched observations. That's what the
comment is for.

Bad comments restate the code (`// merge on id and year`). Good comments explain a choice
that had alternatives, a threshold that could have been different, or an assumption about
the data that — if wrong — would break the result.

Every non-trivial block should make clear:

1. **GOAL** — what this block accomplishes
2. **REASONING** — why this approach over the alternatives
3. **ASSUMES** — what must be true about the data for this to be correct
4. **EXPECT** — what the result should look like, so a bad outcome is caught immediately

## Do-File Header

Every do-file starts with a header — the shapiro_coding_guide's directory/version-control
conventions depend on this being consistent across a project:

```stata
/*==============================================================================
FILE:     02_clean_enrollment.do
PURPOSE:  Clean raw enrollment extract; collapse to school-year panel
INPUT:    raw/enrollment_2015_2022.csv
OUTPUT:   build/enrollment_panel.dta
AUTHOR:   [name]                       DATE: [date]
NOTES:    Assumes raw file has one row per student-year; see EDA log for
          missingness patterns in `disability_status`.
==============================================================================*/

clear all
set more off
version 18
```

## Commenting Transformations

```stata
* ----------------------------------------------------------------------------
* GOAL: restrict to active, degree-seeking undergraduates
* REASONING: part-time and non-degree students have incomplete outcome data
*   (confirmed in EDA: 40% missing on `grad_status` for non-degree flag == 1)
* ASSUMES: `enrollment_status` is populated for all rows (verified: 0 missing)
* EXPECT: roughly 70% of rows retained, based on prior cohort
* ----------------------------------------------------------------------------
count
local n_pre = r(N)

keep if enrollment_status == "active" & degree_seeking == 1

count
di "Retained " %4.1f (r(N)/`n_pre'*100) "% of rows (expected ~70%)"
```

## Commenting Merges — the highest-risk operation

Merges are where silent data loss happens. Document the key, the type, and the expected
match rate, then check it:

```stata
* ----------------------------------------------------------------------------
* GOAL: attach district-level spending data to the student panel
* KEY: district_id year — verified unique in `spending.dta` via isid below
* JOIN TYPE: m:1, left-preserving (keep all students; unmatched spending is
*   a data gap to investigate, not grounds for dropping students)
* EXPECT: ~95% match rate; districts formed after 2020 will not match
* ----------------------------------------------------------------------------
preserve
    use spending.dta, clear
    isid district_id year          // fails loudly if key assumption is wrong
restore

merge m:1 district_id year using spending.dta
tab _merge
* ADVERSARIAL CHECK: does the _merge==2 count match "districts with no
* students," or does it suggest a key mismatch (e.g., district_id coded
* differently pre/post 2020)? Investigate before dropping _merge==2.
drop if _merge == 2
```

## Commenting Collapses and Reshapes

```stata
* GOAL: collapse student-level test scores to school-year averages
* REASONING: mean, not median — outcome is used in a linear regression
*   downstream; median would be more robust but less consistent with that model
* ASSUMES: `test_score` missing-at-random within school-year (spot-checked:
*   missingness driven by absentee day, not ability — see EDA log)
local n_students = _N
collapse (mean) test_score (count) n_students=test_score, by(school_id year)
* Sanity check: no school-year cell should be built from fewer than ~15 kids
sum n_students, detail
assert n_students >= 5
```

## Adversarial Self-Review (do this before moving on)

After writing a block, re-read it as if you did not write it and are trying to find the
bug. This is not restating what the code does — it's actively hunting for the failure
modes Gentzkow & Shapiro flag as the most common in applied work:

- Could this `merge` be silently dropping matched or unmatched rows I need?
- Is the key actually unique on **both** sides of the merge (did I check `isid`, not just
  assume it)?
- Does a hard-coded path, date, or sample restriction break silently if the raw data updates?
- Am I `replace`-ing a variable I still need somewhere downstream?
- Would this code produce a plausible-but-wrong answer instead of an obvious error if the
  input format changed slightly?

If any answer is "not sure," add the check (`isid`, `assert`, `count` before/after) rather
than trusting the comment alone.

## Inline Validation, Not Separate Test Files

Validate at the point of use, in the same do-file, rather than building a separate test
suite:

```stata
assert !missing(student_id)
isid student_id year
count if age < 3 | age > 25
assert r(N) == 0     // ages outside plausible range for this sample
```

## Good vs. Bad

**Bad — restates the code:**
```stata
* loop over years
foreach y of numlist 2015/2022 {
```

**Good — explains a choice:**
```stata
* Loop rather than reshape wide-to-long here because each year's raw file has
* a different variable naming convention (renamed in 2019); a single reshape
* would require harmonizing names first, which is more error-prone than
* looping and standardizing on the way in.
foreach y of numlist 2015/2022 {
```

**Bad — no context:**
```stata
drop if income < 0
```

**Good — states the reasoning and checks the result:**
```stata
* Negative income values are entry errors (confirmed with data provider: no
* valid negative-income scenario in this survey). Drops ~1.5% of rows.
count if income < 0
di "Dropping " r(N) " rows with negative income"
drop if income < 0
```
