---
name: stata
description: >
  Expert guidance for writing, running, and managing Stata do-files in an economics research environment on BU SCC.
  Use this skill whenever the user writes Stata code, creates or edits .do files, asks about Stata syntax or errors,
  runs regressions, builds data cleaning pipelines, or mentions Stata, reghdfe, csdid, did2s, gcollapse, parmest,
  or related econometrics packages. Also use when the user asks about code organization, reproducibility, data
  management, or research workflow for empirical work — even if "Stata" isn't mentioned explicitly.
---

# Stata in Economics Research

Stata is the dominant statistical computing environment in economics and social science. Work happens in
**do-files** (`.do` — plain-text scripts that Stata executes line-by-line), which read and write
**datasets** (`.dta` format) and produce **logs** (`.log` — plain-text record of all output).
The goal of every do-file is that running it from scratch produces the same result every time.

**Reading documentation**: Always read documentation in machine-readable formats (`.md`, `.txt`) rather than
PDFs. If only a PDF exists, convert it first: `pdftotext file.pdf file.txt`.

---

## Running Stata on BU SCC

Load the module before any Stata command:
```bash
module load stata-mp/18      # version 18 (current default)
module load stata-mp/19      # version 19 also available
```

**Batch mode** (most common on the cluster — runs in background, writes `.log`):
```bash
stata-mp -b do path/to/script.do
stata-mp -b do script.do arg1 arg2    # pass arguments accessed via `args` or `local 0`
```
The log lands in the same directory as the do-file, named `script.log`.

**Interactive** (requires X11 forwarding via `ssh -Y`):
```bash
stata-mp
```

**SGE batch job** — create a `.qsub` file:
```bash
#!/bin/bash -l
#$ -N my_stata_job
#$ -l h_rt=04:00:00
#$ -l mem_total=8G
#$ -pe omp 4           # stata-mp uses up to 4 cores
#$ -j y
#$ -o logs/

module load stata-mp/18
stata-mp -b do code/my_analysis.do
```
Submit with `qsub my_job.qsub`. Check status with `qstat`. Output in `my_stata_job.o<jobid>`.

**Absolute path** (when modules aren't loaded): `/share/pkg.8/stata-mp/18/install/bin/stata-mp`

---

## Do-file Structure

Every script follows this opening sequence (Gentzkow & Shapiro, §2 Automation):

```stata
cap log close
clear all
set more off

* --- Set working directory ---
cd "$project_dir"           // global set in globals.do, or hardcode cluster path
// cd "/projectnb/econdept/snitkin/myproject"

* --- Local path aliases (set once, use everywhere) ---
local raw       "data/raw"
local processed "data/processed"
local results   "results"
local figures   "`results'/figures"
local tables    "`results'/tables"

cap mkdir `figures'
cap mkdir `tables'

// --- 1. Load and clean ---
// --- 2. Merge ---
// --- 3. Analysis ---
// --- 4. Export ---
```

Hardcoding a path string more than once creates maintenance debt. One alias at the top means one place to change.

### Feature flags for expensive steps

Gate slow operations (infile, reshape, large merges) with a flag so reruns skip them:
```stata
local REBUILD_RAW = 0
if `REBUILD_RAW' {
    infile using "`raw'/bigfile.csv", ...
    save "`processed'/cleaned.dta", replace
}
use "`processed'/cleaned.dta", clear
```
Alternatively, `capture confirm file` to condition on file existence.

### Tempfiles for intermediates

Datasets only needed as merge sources within a script don't need permanent names:
```stata
tempfile county_pop
preserve
    keep county year population
    save `county_pop'
restore
merge m:1 county year using `county_pop', keep(1 3) nogen
```

---

## Data Management Best Practices (Gentzkow & Shapiro)

**G&S is the authoritative reference** for research code quality. Read it at:
`/projectnb/econdept/snitkin/resources/shapiro_coding_guide.md`

Key principles:

### Automate everything (G&S §2)
No manual steps between raw data and final output. If you opened Excel and saved a file by hand, that step needs to be a script. A meta-file (or `meta_file.do`) should be able to reproduce everything from scratch.

### Keys and uniqueness (G&S §5)
Every dataset has a unit of observation. Assert it before merges:
```stata
gisid leaid year                     // errors if not unique
duplicates report leaid year         // inspect before dropping
```
After a merge, check what matched:
```stata
merge m:1 leaid year using "`processed'/covars.dta", keep(1 3) nogen
assert _merge == 3 if year >= 2000   // document expected match rate
drop _merge
```
Specify `keep(1 3)` (keep master + matched) or `keep(3)` (matched only) explicitly — never let Stata keep all three silently.

### Abstraction (G&S §6)
When the same block of code appears in multiple places, wrap it in a `program`:
```stata
program define clean_sentinel_vals
    args varlist
    foreach var in `varlist' {
        replace `var' = . if `var' < 0
    }
end
```
Collect related variable names into a local rather than repeating them:
```stata
local outcome_vars math_score read_score grad_rate
keep leaid year `outcome_vars'
foreach v in `outcome_vars' {
    replace `v' = . if `v' < 0
}
```

### Negative-value / sentinel cleaning
CCD, SEDA, and many admin datasets use -1, -2, -9 to encode missing. Replace immediately after loading:
```stata
foreach v of varlist _all {
    capture replace `v' = . if `v' < 0 & `v' != .
}
```

---

## Preferred Packages

| Task | Package | Install |
|---|---|---|
| TWFE panel FE regression | `reghdfe` | `ssc install reghdfe` |
| Callaway-Sant'Anna DiD | `csdid`, `csdid_estat` | `net install csdid, from(https://raw.githubusercontent.com/friosavila/csdid/main/installation/)` |
| Gardner et al. 2SLS DiD | `did2s` | `net install did2s, from(https://raw.githubusercontent.com/kylebutts/did2s_stata/main/installation/)` |
| Fast collapse / sort | `gcollapse`, `gsort`, `gisid` | `ssc install gtools` |
| Coefficient export to dataset | `parmest` | `ssc install parmest` |
| Shapefiles → Stata | `shp2dta`, `geoinpoly` | `ssc install shp2dta` / `ssc install geoinpoly` |
| State FIPS crosswalk | `statastates` | `ssc install statastates` |
| Event-study plots | `event_plot` | `ssc install event_plot` |

**Rule**: prefer `gcollapse` over `collapse`, `gisid` over `isid` — they're faster and the syntax is identical.

### reghdfe usage
```stata
reghdfe outcome treatment covar1 covar2 [aw=w], ///
    absorb(leaid#grade year) ///
    vce(cluster leaid)

* Store results
parmest, saving("`tables'/results.dta", replace) label
```
Always specify `absorb()` and `vce()` explicitly. Weights go in `[aw=w]` with a pre-generated variable.

---

## Comments and Documentation (G&S §7)

Comment the **why** — hidden constraints, workarounds, non-obvious decisions. Never narrate what the next line does.

Good:
```stata
* CCD reports -1/-2 for suppressed cells (FERPA), not actual negative enrollment
replace enrollment = . if enrollment < 0
```

Redundant (skip it):
```stata
* Replace negative values with missing
replace enrollment = . if enrollment < 0
```

---

## Reference Files

Read these only when the task requires them:

- **Graph conventions** (twoway, export, slide formatting): `references/graph_conventions.md`
- **Parallelism / SGE array jobs** (large regression batteries): `references/parallelism.md`
- **Help resources** (Statalist, SSC, package docs, log debugging): `references/help_resources.md`
