# autoresearch: {{GOAL}}

Autonomous experiment loop. You modify ONE artifact, a read-only judge produces ONE metric, every change is one git commit. Improvements stay; regressions get reverted.

## Goal

{{GOAL}}

Optimize **{{METRIC_NAME}}** ({{METRIC_DIRECTION}} is better).

## Rules

**You CAN:**
- Modify `{{ARTIFACT}}` — this is the only thing you edit. Architecture, content, config, copy — fair game.

**You CANNOT:**
- Modify the judge command, its config, or any file it depends on for measurement.
- Add new dependencies unless the existing artifact already imports them.
- Skip logging an experiment to `results.tsv`.
- Pause to ask the human "should I continue?" once the loop has started. The human is asleep or away.

**Simplicity criterion**: All else equal, simpler wins. A {{METRIC_NAME}} improvement that adds 50 lines of hacky code is suspect. A null-effect change that deletes code is a keep.

## Judge

Run an experiment with:

```
{{JUDGE_CMD}} > run.log 2>&1
```

Extract the metric:

```
grep "{{METRIC_GREP}}" run.log
```

If grep is empty, the run crashed. `tail -n 50 run.log` to diagnose.

**Time budget**: each experiment ~{{TIME_BUDGET_MIN}} min. If it exceeds {{KILL_BUDGET_MIN}} min, kill it and treat as a crash.

## Logging

Append every experiment to `results.tsv` (tab-separated, NOT comma — commas break descriptions). Header:

```
commit	metric	status	description
```

- `commit` — short git hash (7 chars)
- `metric` — the {{METRIC_NAME}} value, or `0` for crashes
- `status` — `keep`, `discard`, or `crash`
- `description` — short text: what the experiment tried

Do NOT commit `results.tsv` — it stays untracked.

## The loop

LOOP FOREVER:

1. Look at git state — current branch, latest kept commit, recent results.tsv rows for context.
2. Form an experimental hypothesis. Edit `{{ARTIFACT}}` to test it.
3. `git add -A && git commit -m "<short description>"`
4. Run: `{{JUDGE_CMD}} > run.log 2>&1`
5. `grep "{{METRIC_GREP}}" run.log` → extract metric
6. Append row to `results.tsv`
7. If {{METRIC_DIRECTION}} (vs. last keep): keep — branch advances, commit stays
8. Else: `git reset --hard HEAD~1` — discard
9. GOTO 1

**Never stop.** If you're out of ideas, think harder. Re-read `{{ARTIFACT}}`. Look at the last 20 results for patterns — what kinds of changes worked, what didn't? Try combining near-misses. Try something radical. The loop ends only when the human interrupts.

**Crashes**: dumb fixes (typo, missing import) — fix and rerun. Fundamentally broken idea — log `crash`, revert, move on.

**Context discipline**: redirect ALL judge output to `run.log`. Never `tee`, never let stdout flood your context. You only need the grep result.
