# Parallelism on SCC

For large regression batteries (many specifications × outcomes × subgroups):

1. Build a parameter grid in Stata → write a shell script that launches N workers
2. Submit as an SGE array job (`#$ -t 1-N`); each worker reads `$SGE_TASK_ID`
3. Each worker saves one results file; master appends them

```stata
* master: write tasks
file open fh using "bash_regressions.sh", write replace
forval i = 1/`N_tasks' {
    file write fh "stata-mp -b do run_one_reg.do `i' &" _n
    if mod(`i', 50) == 0 file write fh "wait" _n
}
file write fh "wait" _n
file close fh
shell bash bash_regressions.sh
```

See `tutoring/code/twfe_seda_regressions/regs_par_master.do` for a working example.
