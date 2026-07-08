# Data Documentation

Understanding existing documentation, and building it when none exists — in Stata.

## Questions to Ask Before Analysis

| About | Ask | Why it matters |
|---|---|---|
| Source | Where did this come from, who maintains it, when was it last updated? | Reliability and staleness |
| Collection | Survey, admin records, sensor, scrape? What time period, what population? | Each collection method has its own biases |
| Quality | Known issues? How are errors handled? Special codes for missing (-99, "N/A")? | Avoid rediscovering known problems, or mishandling sentinel values as real data |
| Context | What decisions rely on this data? Who are the domain experts to ask? | Calibrates how much scrutiny it needs |

## Understanding Each Variable

For every variable, know: what it represents in the real world, its unit, what a missing
value means (not applicable vs. unknown vs. not collected), its valid range or categories,
and whether it's raw or derived (and from what).

Use Stata's own metadata as the starting point, then fill gaps by hand:

```stata
codebook, compact           // type, range, missing, unique values — one line per var
describe                    // types and existing variable labels
label list                  // all defined value labels
misstable summarize         // missingness by variable, with patterns
```

If `codebook` and `misstable` don't answer a question (e.g., "is -99 a sentinel for
missing, or a real value?"), that's the signal to go find the documentation or ask the data
owner — don't guess.

## Working With Undocumented Data

**Step 1 — profile it.**
```stata
codebook, compact
misstable summarize
duplicates report
```

**Step 2 — find the granularity.** What does one row represent, and what combination of
variables uniquely identifies it?
```stata
isid id                     // fails with an error if not unique — that's useful information
isid id year                // try likely composite keys
duplicates report id year   // if isid fails, see how bad the duplication is
```

**Step 3 — infer variable meaning from names and behavior**, then confirm rather than
assume:
- `*_id`, `*_code` → likely a key or category code — check `codebook varname` for range/uniqueness
- `*_date`, `*_yr` → temporal — check `format` and range
- `*_amt`, `*_cost`, `*_price` → monetary — check units (dollars? thousands?) against a
  known benchmark value
- Binary 0/1 with a `label list` entry → confirm which value means "yes"

**Step 4 — write it down as you learn it**, immediately, in the do-file that builds the
dataset:
```stata
label variable district_id "NCES district identifier, 7-digit"
label variable enrl_total "Total enrollment, fall count"
note enrl_total: "Excludes pre-K per NCES definition; confirmed with state DOE 2023-04-02"
```

`label variable` and `note` travel with the `.dta` file — they're documentation that
can't go stale from being in a separate, forgotten file.

## Minimum Viable Data Dictionary

If a companion `.md` file is more useful than in-file labels (e.g., for a data dictionary
shared with coauthors), keep it this short and update it as part of the same commit that
changes the data:

```markdown
## Data Dictionary: [Dataset Name]

- **Source**: [where it comes from]
- **Granularity**: [what one row represents — confirmed via `isid`]
- **Date range / row count**: [coverage]

| Variable | Type | Description | Missing means | Notes |
|---|---|---|---|---|
| district_id | str7 | NCES district identifier | never missing | primary key |
| enrl_total | int | Fall enrollment count | true zero if closed | excludes pre-K |

### Known Issues
- [quality issue, and how you handled it]

### Change Log
- [date]: [what changed and why]
```

Generate the variable table mechanically as a starting point, then fill in the
description/notes columns by hand — don't leave `[TODO]` placeholders in a file you ship:

```stata
codebook, compact
* Copy the variable/type/range columns into the dictionary table above,
* then add business-context descriptions manually.
```

## Red Flags to Surface, Not Silently Work Around

| Red flag | Check | Concern |
|---|---|---|
| High missingness on a key variable | `misstable summarize` | May bias any analysis using it |
| `isid` fails on the assumed key | `isid`, `duplicates report` | Granularity assumption is wrong |
| Values outside plausible range | `summarize, detail`, `assert` | Data entry error or unit mismatch |
| Dates in the future or before program existed | `summarize date_var` | Entry error or miscoded field |
| No documentation exists at all | — | Flag before analyzing, don't just proceed |
| Convenience sample / low response rate | ask the data owner | Findings may not generalize |

When you find one of these, say so explicitly to the user rather than quietly filtering it
out — a dropped observation that "looked wrong" is exactly the kind of silent decision that
later changes a result and can't be reconstructed.
