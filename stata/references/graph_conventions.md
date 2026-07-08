# Graph Conventions

```stata
local format png           // change in one place
local figures "results/figures"

twoway (scatter y x), ///
    ylab(, nogrid) ///
    yline(0, lpattern(dash) lcolor(gs8)) ///
    title("") ///
    name(g1, replace)

local filepath "`figures'/scatter_`subj'_`grp'"
graph export "`filepath'.`format'", replace
```

Strip titles for slides (`gr_edit .title.text = {}`). Always `cap mkdir` the figures directory before saving.
