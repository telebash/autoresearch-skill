# autoresearch (Claude Code skill)

A Claude Code [skill](https://docs.claude.com/en/docs/claude-code/skills) that sets up an autonomous experiment loop in any project. The agent edits one artifact, runs a fixed-budget evaluation, keeps changes that improve a single metric and reverts the rest, logging every attempt to `results.tsv`.

Adapted from the **autoresearch** workflow by [Andrej Karpathy](https://github.com/karpathy/autoresearch) (March 2026):

- agent edits one **artifact**
- a read-only **judge** computes one **metric**
- every experiment is one git commit
- improvements advance the branch, regressions get reverted
- loop until the human stops it

## When to use

Run overnight on optimization tasks like:

- model / training-recipe tuning
- site performance
- copy / A-B testing
- prompt tuning
- blog idea generation
- anything where you can express progress as one number

Triggers: *"autoresearch"*, *"run experiments overnight"*, *"iterate on X autonomously"*, *"optimize X in a loop"*.

## Install

### Option 1 — via [`skills`](https://github.com/vercel-labs/skills) CLI (recommended)

Globally (installs into `~/.claude/skills/`):

```bash
npx skills add telebash/autoresearch-skill -g
```

Or project-scoped (omit `-g`).

### Option 2 — git clone

```bash
git clone https://github.com/telebash/autoresearch-skill.git ~/.claude/skills/autoresearch
```

### Use

In any Claude Code session, ask:

> autoresearch this repo — optimize <metric>

Claude will pick up the skill, run the setup interview, scaffold the files, and hand off the kickoff instructions.

## Files

- `SKILL.md` — the skill definition (interview, scaffolding, handoff).
- `program-template.md` — template for the per-experiment program the loop runs.
- `examples.md` — worked examples across domains.

## License

MIT
