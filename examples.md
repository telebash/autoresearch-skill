# Judge scaffolds for common domains

Reference templates for the read-only `measure.*` script that produces the metric. Adapt to the user's stack — these are starting points, not drop-in code.

The contract every judge must satisfy:
1. Runs from a single shell command
2. Prints exactly one line matching `{{METRIC_GREP}}` to stdout (e.g. `lcp_ms: 2140`)
3. Self-imposes a wall-clock timeout
4. Is deterministic enough that two runs on the same artifact give similar scores (average multiple runs if not)

---

## Domain 1: ML training (the original autoresearch case)

Already covered by `train.py` in this repo. Metric: `val_bpb` (lower better). Budget: 5 min.

---

## Domain 2: Web performance

**Goal**: lower LCP / raise Lighthouse score on a local build.

**Artifact**: `src/` (or whatever the user's frontend root is).

**Judge** (`measure.js`):

```js
// measure.js — read-only
import { execSync } from "child_process";
import { writeFileSync } from "fs";

execSync("npm run build", { stdio: "inherit" });
const server = execSync("npm run preview &", { stdio: "inherit" });

// wait for port
execSync("npx wait-on http://localhost:4173");

const runs = [];
for (let i = 0; i < 3; i++) {
  const out = execSync(
    `npx lighthouse http://localhost:4173 --quiet --output=json --chrome-flags="--headless"`
  ).toString();
  const r = JSON.parse(out);
  runs.push(r.audits["largest-contentful-paint"].numericValue);
}
runs.sort((a, b) => a - b);
console.log(`lcp_ms: ${Math.round(runs[1])}`);  // median
```

**Run**: `node measure.js`
**Grep**: `^lcp_ms:`
**Budget**: ~3 min/experiment

Caveat: Lighthouse is noisy. Median-of-3 is the floor; median-of-5 is more honest if you can spare the time.

---

## Domain 3: LLM-judged content (blog ideas, copy, headlines)

**Goal**: maximize a virality / clarity / engagement score from a frozen LLM rubric.

**Artifact**: `ideas.md` (one markdown file with the candidate the agent is iterating on).

**Judge** (`score.py`) — the prompt is **frozen and committed**, never edited by the agent:

```python
# score.py — read-only
import sys, anthropic, re

artifact = open(sys.argv[1]).read()

JUDGE_PROMPT = """You are a brutal editor scoring blog post ideas for virality.

Score 0-100 based on:
- Hook strength (0-30): does the title stop the scroll?
- Specificity (0-25): concrete claim vs. generic platitude?
- Curiosity gap (0-25): would a reader click to find out?
- Shareability (0-20): would someone repost this without reading?

Output ONLY a single line: `score: <integer>`"""

client = anthropic.Anthropic()
resp = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=64,
    system=JUDGE_PROMPT,
    messages=[{"role": "user", "content": artifact}],
)
text = resp.content[0].text
match = re.search(r"score:\s*(\d+)", text)
print(f"score: {match.group(1)}" if match else "score: 0")
```

**Run**: `python score.py ideas.md`
**Grep**: `^score:`
**Budget**: ~30 sec/experiment (so ~100/hour, ~1000/night)

**Critical caveats**:
- The judge prompt **must be in version control and not in the artifact** — otherwise the agent will "optimize" by editing the rubric to score itself higher.
- Single LLM call is noisy. Average 3 calls if budget allows, or use lower temperature.
- LLM judges drift over model versions. Pin the model ID.
- Consider augmenting with an external signal (Reddit/HN API for "did similar headlines actually go viral") to ground the score in reality.

---

## Domain 4: Prompt optimization

**Goal**: raise accuracy of a Claude-powered task on a fixed eval set.

**Artifact**: `prompts/system.md` (the system prompt the agent is iterating).

**Judge** (`eval.py`):

```python
# eval.py — read-only
import json, anthropic
from pathlib import Path

system_prompt = Path("prompts/system.md").read_text()
eval_set = json.loads(Path("eval/cases.json").read_text())  # [{input, expected}, ...]

client = anthropic.Anthropic()
correct = 0
for case in eval_set:
    resp = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=512,
        system=system_prompt,
        messages=[{"role": "user", "content": case["input"]}],
    )
    if case["expected"].strip() in resp.content[0].text:
        correct += 1
print(f"accuracy: {correct / len(eval_set):.4f}")
```

**Run**: `python eval.py`
**Grep**: `^accuracy:`
**Budget**: depends on eval set size. For 50 cases on Haiku, ~1 min/experiment.

**Caveats**:
- Eval set is read-only. The agent edits ONLY the system prompt.
- Use a held-out test set after the loop ends to check the prompt didn't overfit.
- Cache the eval inputs via prompt caching to cut cost ~10×.

---

## Domain 5: SQL query optimization

**Goal**: lower query execution time on a fixed dataset.

**Artifact**: `queries/report.sql`

**Judge** (`measure.sh`):

```bash
#!/bin/bash
# measure.sh — read-only
set -e

# warm cache (3 throwaway runs)
for i in 1 2 3; do
  psql -f queries/report.sql > /dev/null
done

# measured runs
times=()
for i in 1 2 3 4 5; do
  t=$(psql -c "\timing on" -f queries/report.sql 2>&1 | grep "^Time:" | awk '{print $2}')
  times+=("$t")
done

# median
median=$(printf '%s\n' "${times[@]}" | sort -n | awk 'NR==3')
echo "exec_ms: $median"
```

**Run**: `bash measure.sh`
**Grep**: `^exec_ms:`
**Budget**: depends on query, typically 30 sec – 5 min

**Caveat**: ensure the agent can't "optimize" by changing the query semantics. Add a correctness check: run the query, hash the result set, compare to the baseline hash. If hashes differ, score is `0` (crash).

---

## Anti-patterns (do not propose these to the user)

- **"Score" that's just the agent's own self-assessment** — agent grades its own work, score goes up forever, nothing improves.
- **Metric depending on internet weather** — flaky external API in the judge means runs aren't comparable.
- **Multi-objective without aggregation** — "minimize latency AND bundle size AND TTFB" — pick one or define a fixed weighted sum, otherwise the agent has no signal.
- **Time-budgeted by token count, not wall clock** — agent's tokens-per-experiment vary wildly; use real seconds.
