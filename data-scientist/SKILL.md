---
name: data-scientist
description: >-
  Data science methodology for rigorous quantitative research in Stata. Covers EDA, data
  documentation, transformation verification, causal inference (IV, DiD, RD, synthetic
  control), descriptive analysis, statistical modeling, and adversarial code review. This
  skill governs *how to think and work*; the `stata` skill governs *do-file syntax*. Always
  consult this skill before starting a new analysis, cleaning data, running a regression,
  choosing a causal identification strategy, or reviewing someone else's (or your own)
  do-file for correctness — even if the user just says "clean this data" or "run this
  regression" without naming the skill.
metadata:
  audience: any-agent
  domain: research-methodology
---

# Data Scientist Skill

Methodology for Stata-based empirical research: how to inspect data, document it, transform
it safely, choose a statistical or causal method, and catch your own mistakes before they
reach a paper. This skill is about judgment, not syntax — for do-file syntax (`reghdfe`,
`csdid`, `gcollapse`, `parmest`, etc.) load the **`stata`** skill alongside this one.

The workflow conventions here (single master do-file, relative paths, one script per
directory stage, everything logged) follow `./shapiro_coding_guide.md` (Gentzkow & Shapiro,
*Code and Data for the Social Sciences*). Read that file directly when setting up a new
project's directory structure or version-control conventions — it is not summarized here.

## Core Principles

### 1. Check the data before you touch it

Before any analysis or transformation, run `describe`, `codebook`, `summarize, detail`, and
look at a random sample of rows (`sample`, then `list` or `browse`). Establish what
uniquely identifies an observation (`isid` or `duplicates report`) before writing a single
line of cleaning code. State findings out loud in comments — never assume the data is what
the codebook says it is.

### 2. Document the data, not just the code

Before analysis, find or write down: where the data came from, how it was collected, what
each variable means, and known quality issues. If a data dictionary doesn't exist, build one
as you go (`label variable`, `label define`, a companion `.md` or `.do` file). See
`./data-documentation.md`.

### 3. Verify every operation — including adversarially

Never assume a `merge`, `collapse`, `reshape`, or `egen` did what you intended.

- Record row counts and a few key aggregates before the operation.
- After, check `_merge` tallies, compare counts, and spot-check the same observations
  before and after.
- Then read the block back **as a skeptical reviewer, not the author**: assume there is a
  bug and try to find it. Ask specifically: could this merge be silently dropping matched
  or unmatched rows? Is the key actually unique on both sides (`isid` before every merge)?
  Would a hard-coded path, sample restriction, or `if` condition break silently on updated
  data? Is a `replace` overwriting a variable you still need downstream? This adversarial
  pass is not optional cleanup — it is the step that catches the errors Gentzkow & Shapiro
  describe as the most common (and most expensive) in applied work: a bad merge key that
  quietly drops observations and changes the results.
- Document what you checked and what you found, in comments, next to the code.

### 4. Comment for the reader who isn't you in six months

Every non-trivial block needs: what you're trying to accomplish, why you chose this
approach over the alternatives, and what you're assuming about the data for it to be
correct. See `./code-documentation.md` for the comment format and examples.

### 5. Match rigor to the question

Ask what decision the analysis informs and how much precision it actually needs. Check in
with the user when multiple valid methods exist, when there's a real precision/practicality
tradeoff, or when a result is surprising — don't silently pick a method and move on.

## Loading Other Skills

| Situation | Load |
|---|---|
| Writing or debugging any Stata code | `stata` skill (syntax, `reghdfe`/`csdid`/`gcollapse`/etc.) |
| Turning results into prose | `econ-writing` skill |
| Writing a figure/table note | `draft-exhibit-note` skill |
| Something Stata genuinely can't do (UMAP, SHAP, deep learning, exotic ML) | escalate to the user — don't silently switch to Python mid-project |

Stata is the default and expected tool for everything in the table below. Only reach for
Python/R if the user's project already uses it or the task is explicitly outside Stata's
reach.

## Reference Files — Read Only When Needed

Keep this file's body in context; load a reference file only for the task at hand.

| File | Read when |
|---|---|
| `eda-checklist.md` | Starting on unfamiliar data — missingness, duplicates, outliers, cardinality |
| `data-documentation.md` | Data dictionary doesn't exist, or provenance/quality is unclear |
| `code-documentation.md` | Writing or reviewing do-file comments |
| `research-questions.md` | Scoping the question, framing causal vs. correlational claims, stakeholder check-ins |
| `descriptive-analysis.md` | Summary stats, subgroups, distributions, trends, decompositions, weighting |
| `statistical-modeling.md` | Model selection, assumption checks, SE choice, coefficient interpretation |
| `causal-inference.md` | Any causal claim — DAGs, IV, RD, DiD, synthetic control, matching |
| `geospatial-analysis.md` / `geospatial-operations.md` | Spatial data, joins, weights, mapping |
| `exploratory-unsupervised.md` | Clustering, PCA, typology construction |
| `shapiro_coding_guide.md` | Project/directory setup, version control, master do-file structure |

Note: these reference files still contain Python-era examples pending conversion to Stata —
treat the *methodology* (what to check, what to choose, what to report) as authoritative,
and translate any code example into Stata syntax via the `stata` skill rather than copying
it as-is.

## Decision Trees

**Starting a new analysis** → do I have data documentation? Read/build it
(`data-documentation.md`) → have I profiled the data? Run the EDA checklist
(`eda-checklist.md`) → is the research question clear? If not, clarify
(`research-questions.md`) → proceed, documenting as you go.

**Describing vs. explaining** → characterizing what exists → `descriptive-analysis.md`.
Testing a hypothesis or modeling a relationship → `statistical-modeling.md`. Making a causal
claim → `causal-inference.md`.

**Choosing a causal design** → random assignment → RCT. Can control for all confounders →
regression/matching. Valid instrument → IV/2SLS. Score cutoff → RD. Policy change hits some
units → DiD. Few treated units, long pre-period → synthetic control. Not sure → start with a
DAG. All in `causal-inference.md`.

**Spatial data** → understanding concepts/method choice → `geospatial-analysis.md`.
Performing joins/weights/interpolation → `geospatial-operations.md`. Either way, load
`stata` (or `geopandas` if the project is Python) for syntax.

## Workflow Checklists

**New data**
- [ ] `describe`, `codebook`, random `sample` + `list`
- [ ] Missingness (`misstable summarize`), duplicates (`duplicates report`)
- [ ] `isid` (or documented candidate keys) — what uniquely identifies a row?
- [ ] Distributions (`summarize, detail`, `tabulate` for categoricals)
- [ ] Findings documented before any transformation

**Any transformation (merge, collapse, reshape, egen)**
- [ ] Pre-state recorded: row count, key aggregates, a sample of observations
- [ ] Comment states GOAL, REASONING, ASSUMPTIONS before the code
- [ ] Post-state checked: row count, `_merge` tally, same sample re-inspected
- [ ] Adversarial re-read: what's the most likely silent failure mode here?
- [ ] Result documented in comments

**Before sharing results**
- [ ] Limitations and caveats stated
- [ ] Causal language only where justified; correlational otherwise
- [ ] Uncertainty quantified (CIs, not just point estimates)
- [ ] Surprising results cross-checked against a sanity benchmark
