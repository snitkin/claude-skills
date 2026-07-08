# Research Questions

Framing analytical questions, matching rigor to stakes, and communicating findings.

## From Vague to Answerable

Translate loose questions into something a regression can actually target: specify the
outcome and unit of measurement, define the population/sample, set the time frame,
establish a comparison group or baseline, and state what magnitude would count as
meaningful.

| Vague | Answerable |
|---|---|
| "Did the policy help?" | "Did state minimum wage increases raise teen employment in border counties, relative to counties in non-adopting states, 2015–2022?" |
| "Why is this school district struggling?" | "Which observable district characteristics (spending per pupil, teacher turnover, enrollment change) predict the decline in test scores since 2019?" |

Before starting, get answers to: what decision does this inform, what would a convincing
answer look like, how much precision is actually needed, and has this question been
studied before (check NBER/SSRN/top journals per your usual source order before assuming
it's novel).

## Scope

State explicitly, in writing, before starting: what's in scope, what's out of scope
(including the adjacent questions someone will inevitably ask), and the assumptions and
data dependencies the analysis rests on. This is what prevents scope creep from silently
changing what a result actually claims to show.

## Matching Rigor to Stakes

| Situation | Rigor |
|---|---|
| Exploratory / deciding whether to pursue a question | Lower — a back-of-envelope regression is fine |
| Result will inform an internal decision | Medium |
| Submission-track result, or informs a real policy recommendation | High — robustness checks, sensitivity analysis, pre-registration where relevant |

Simplify when the decision wouldn't change with a more precise estimate, or when data
quality caps the achievable precision regardless of method. Add rigor when stakes are high,
findings are surprising, or the result will face outside scrutiny (referees, policymakers).

## Checking In

Check in with the advisor/coauthor/user when: multiple valid identification strategies
exist with real tradeoffs, an assumption materially affects the conclusion, a finding
contradicts priors or the existing literature, or the analysis needs to expand beyond the
original question to be useful. Frame the check-in as a decision with options, not an
open-ended question: "I can identify this via DiD, which requires parallel trends [tradeoff],
or via IV using [instrument], which requires exclusion [tradeoff]. I'd lean toward X because
Y — does that fit what you need?"

## Communicating Results

Default to correlational language ("X is associated with Y," "X predicts Y") unless the
identification strategy earns causal language — random assignment, a defensible IV/RD/DiD
design with stated assumptions, or (rarely) unusually strong observational controls. State
the estimate with its uncertainty, not as a bare point: "The estimated effect is 0.15 SD
(95% CI: 0.05–0.25), based on N=12,400 student-year observations."

Every write-up needs a limitations section covering the data (what's excluded, known
quality issues), the method (assumptions, what alternative approaches would have shown),
and generalizability (what population this result does and doesn't speak to). State
plainly what the analysis cannot answer — that's information, not weakness.

## Keeping a Decision Log

Track methodological choices as they're made, not reconstructed after the fact:

```markdown
### [Date] — [Question decided]
Options considered: [A vs. B, with tradeoffs]
Decision: [what was chosen]
Rationale: [why]
```

And an assumptions register for anything the result depends on:

| Assumption | Justification | Risk if wrong | Checked? |
|---|---|---|---|
| District ID is a stable key 2015–2022 | Confirmed with NCES documentation | High — panel would be miscoded | Yes, via `isid` |
| Parallel trends holds pre-treatment | Event-study coefficients flat pre-period | High — DiD estimate invalid otherwise | Yes, see event-study plot |

## Handoff Checklist

- [ ] Precise question stated
- [ ] Data source, sample restrictions, and known limitations documented
- [ ] Methodology and why it was chosen over alternatives
- [ ] All material assumptions logged and checked where possible
- [ ] Results reported with uncertainty, not just point estimates
- [ ] Limitations and what the analysis cannot answer stated explicitly
- [ ] Do-files organized so the result is reproducible end to end (see `shapiro_coding_guide.md`)
