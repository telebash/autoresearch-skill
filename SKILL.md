---
name: autoresearch
description: 'Set up an autonomous experiment loop in any project — agent edits one artifact, runs a fixed-budget evaluation, keeps changes that improve a single metric and reverts the rest, logging every attempt to results.tsv. Use when the user wants Claude to iterate overnight on optimization tasks (model training, site performance, copy/A-B testing, prompt tuning, blog idea generation, etc.). Triggers: "autoresearch", "run experiments overnight", "iterate on X autonomously", "optimize X in a loop".'
---

# autoresearch skill

Adapt the autoresearch workflow (Karpathy, March 2026) to any optimization task.

The workflow: agent edits one **artifact**, a read-only **judge** computes one **metric**, every experiment is one git commit, improvements advance the branch, regressions get reverted. Loop forever until the human stops it.

This skill has two phases:
1. **Setup** — interview the user, scaffold the files, create the branch.
2. **Handoff** — print the kickoff instructions; the user starts the loop.

---

## Phase 1: Setup

Run these steps in order. Do not skip the interview — vague answers produce a useless loop.

### Step 1. Confirm the working directory is a git repo

```bash
git rev-parse --is-inside-work-tree
```

If not a repo, ask the user whether to `git init`. Do not proceed without git — the loop relies on commits and reverts.

### Step 2. Interview the user (use AskUserQuestion, one or two questions at a time)

You need five things. Ask in plain language, give examples from the user's likely domain. Do not move on until each is concrete.

1. **Goal in one sentence** — what are we optimizing? ("lower LCP on the marketing site", "find blog post titles with high predicted virality")
2. **Metric** — one number, with direction (lower-is-better or higher-is-better). Reject vague answers like "make it better". Push for something measurable. Examples:
   - `val_bpb` (lower) — model loss
   - `lcp_ms` (lower) — Largest Contentful Paint median of 3 runs
   - `virality_score` (higher) — 0–100 from a frozen LLM-judge prompt
3. **Artifact** — the single file or directory the agent edits. Everything else is off-limits. Examples: `train.py`, `src/`, `ideas.md`, `prompts/system.md`.
4. **Judge** — the read-only command that runs the experiment and prints the metric. Must:
   - Print a line like `metric_name: <number>` to stdout (so it's grep-able)
   - Be deterministic enough that small artifact changes give comparable scores (average multiple runs if noisy)
   - Have a hard time budget (kill itself or be killable after N minutes)
   
   If the user doesn't have one yet, offer to scaffold a starter `measure.py` / `measure.sh` for their domain. Common shapes:
   - **Training**: `uv run train.py > run.log 2>&1` (already prints `val_bpb:`)
   - **Web perf**: `node measure.js` runs Lighthouse 3× and prints median `lcp_ms:`
   - **LLM-judged content**: `python score.py <artifact>` calls Claude API with a frozen prompt, prints `score:`
5. **Time budget per experiment** — wall clock minutes. Used to estimate "experiments per night" and to enforce timeouts.

After collecting all five, restate them back in a compact summary block and ask "ship it?" before generating files.

### Step 3. Pick a run tag and create the branch

```bash
TAG=$(date +%b%d | tr '[:upper:]' '[:lower:]')   # e.g. may07
git checkout -b autoresearch/$TAG
```

If the branch already exists, append `-2`, `-3`, etc. Never reuse a branch.

### Step 4. Generate `program.md` in the project root

Copy `${CLAUDE_SKILL_DIR}/program-template.md` to `<project>/program.md` and fill in the placeholders from the interview:

- `{{GOAL}}` — the one-sentence goal
- `{{METRIC_NAME}}`, `{{METRIC_DIRECTION}}` — e.g. `val_bpb`, `lower`
- `{{ARTIFACT}}` — the file/dir the agent edits
- `{{JUDGE_CMD}}` — the exact shell command that runs an experiment
- `{{TIME_BUDGET_MIN}}` — integer minutes
- `{{KILL_BUDGET_MIN}}` — 2× time budget; runs exceeding this are killed
- `{{METRIC_GREP}}` — the grep pattern for extracting the metric line, e.g. `^val_bpb:`

If the user doesn't have a judge yet, also drop a starter `measure.py` (or `.sh`/`.js`) tailored to their domain — see `${CLAUDE_SKILL_DIR}/examples.md` for templates.

### Step 5. Initialize `results.tsv`

Write **only** the header row:

```
commit	metric	status	description
```

Tab-separated. Add `results.tsv` to `.gitignore` — the log is a working artifact, not source.

### Step 6. Sanity check

Run the judge once on the unmodified artifact to establish that it works and produces a parseable metric. If it crashes or prints nothing matching `{{METRIC_GREP}}`, fix the judge before going further. The loop is worthless if the metric pipeline is broken.

Record the baseline as the first row of `results.tsv` with status `keep` and description `baseline`.

---

## Phase 2: Handoff

Print to the user:

```
Setup complete.
  branch:    autoresearch/<tag>
  artifact:  <artifact>
  metric:    <metric_name> (<direction> is better)
  baseline:  <metric_value>
  budget:    ~<N> experiments/hour

To start the loop:
  "Read program.md and start the experiment loop. NEVER STOP."

To monitor progress:
  watch -n 30 'tail -20 results.tsv | column -t -s "$(printf \\t)"'
```

Do **not** start the loop yourself in the same conversation as setup. The user should start a fresh agent session pointed at `program.md` so context isn't polluted by setup chatter.

---

## What this skill is NOT

- Not a hyperparameter sweeper. The agent uses judgment, not grid search.
- Not safe for tasks that touch shared/production systems. The judge runs locally; the artifact is local code. If the user's "metric" requires shipping to prod, stop and rethink.
- Not for tasks where the metric takes >30 minutes per run. At that point you get <50 experiments/night and the loop economics break down.

## Files in this skill

- `SKILL.md` (this file) — orchestration
- `program-template.md` — the template copied into the project
- `examples.md` — concrete judge scaffolds for common domains (training, web perf, content scoring)

When the user asks for a specific domain, read `examples.md` and adapt the relevant template instead of writing from scratch.
