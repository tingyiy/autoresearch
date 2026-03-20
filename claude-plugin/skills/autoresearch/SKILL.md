---
name: autoresearch
description: "Autonomous research loop — iteratively prototype and improve a feature by running self-contained experiments with pluggable evaluation (LLM judge, metric comparison, test suite). Use when you need to explore, benchmark, or optimize through repeated experimentation."
---

# Autoresearch — Autonomous Experimentation Skill

## When to Use

Use `/autoresearch` when:
- Prototyping a new feature and need to find the best approach
- Optimizing performance, quality, or accuracy through iteration
- Comparing search APIs, LLM prompts, algorithms, or architectures
- Any task where you need to try many approaches and keep the best

## How It Works

```
Set GOAL → Define JUDGE → Run LOOP → Wake up to results
```

The agent runs an unbounded experiment loop:
1. Make one focused change per iteration
2. Evaluate mechanically (never subjectively)
3. Keep improvements, discard failures
4. Progress compounds — every win stacks

## Setup Phase

When the user invokes `/autoresearch`, work with them to define:

### 1. Workspace

Create a self-contained directory for the research. All experiments, results, and reference material live here. The agent must NOT modify files outside this directory during the loop.

```
<workspace>/
├── program.md          # Goal, constraints, evaluation criteria (you generate this)
├── judge.py            # Evaluation harness (you generate this)
├── test_cases.json     # Test inputs with expected outputs (you generate this)
├── reference/          # Pre-captured context (architecture, data, code snapshots)
├── results/            # Experiment outputs (JSON, logs)
├── results.tsv         # Experiment log (tab-separated)
├── best_results.json   # Current best results
└── .env                # API keys (gitignored)
```

### 2. Goal & Constraints

Ask the user:
- **What are you trying to achieve?** (e.g., "find the best search pipeline", "improve classification accuracy", "reduce latency")
- **What can the agent modify?** (scope — files, APIs, approaches)
- **What must NOT change?** (constraints — e.g., "eligibility rules are fixed")
- **What APIs/tools are available?** (LLM providers, search APIs, databases)

### 3. Test Cases

Define test inputs with expected outputs. These are the ground truth the judge evaluates against. Format:

```json
{
  "cases": [
    {
      "id": "t01",
      "input": "...",
      "expected": "...",
      "context": {},
      "notes": "why this case matters"
    }
  ]
}
```

### 4. Judge (Pluggable Evaluation)

The judge is the fixed evaluation harness. **Never modify during the loop.** Three judge types:

#### Type A: LLM Pairwise Judge
For subjective quality (search results, generated text, UX).
Compares two result sets head-to-head via LLM, picks a winner per test case.

```python
# judge.py
async def compare(cases, results_a, results_b) -> (winner, report):
    """Returns "A", "B", or "tie" with per-case reasoning."""
```

#### Type B: Metric Judge
For numeric optimization (latency, accuracy, coverage, bundle size).

```python
# judge.py
def compare(cases, results_a, results_b) -> (winner, report):
    """Compare by metric. Higher/lower is better (configurable)."""
    metric_a = compute_metric(results_a)
    metric_b = compute_metric(results_b)
    if metric_b > metric_a:  # or < for "lower is better"
        return "B", report
```

#### Type C: Test Suite Judge
For correctness (all tests must pass, then compare secondary metric).

```python
# judge.py
def compare(cases, results_a, results_b) -> (winner, report):
    """B wins if: all tests pass AND secondary metric improves."""
    pass_a = count_passing(results_a)
    pass_b = count_passing(results_b)
    if pass_b > pass_a:
        return "B", report
    if pass_b == pass_a and metric_b > metric_a:
        return "B", report
```

#### Type D: Hybrid Judge
Combine multiple criteria with weights.

```python
# judge.py
def compare(cases, results_a, results_b) -> (winner, report):
    """Weighted: 60% correctness + 30% quality + 10% latency."""
```

### 5. Reference Material

Pre-capture all context the agent needs so it never leaves the workspace:
- Architecture docs, code snapshots, API schemas
- Data samples, config files
- Jira ticket, requirements

### 6. Generate & Confirm

After gathering requirements, generate:
- `program.md` — complete specification
- `judge.py` — evaluation harness
- `test_cases.json` — test inputs
- `reference/` — populated with context
- `results.tsv` — header row only

Show the user the generated files. Confirm before starting the loop.

## The Experiment Loop

Once confirmed, the agent runs autonomously. **Do NOT pause to ask the user.**

```
LOOP FOREVER:

1. REVIEW — Read results.tsv, git log, FINDINGS.md. What worked? What failed?
            Identify the most promising direction or unexplored area.

2. HYPOTHESIZE — Decide what to try. Document hypothesis at top of experiment script.

3. BRANCH — Create throwaway branch:
   git checkout -b exp/XXX

4. IMPLEMENT — Write experiment script. One focused change per experiment.

5. COMMIT — Record the experiment:
   git add -A && git commit -m "experiment XXX: <description>"

6. RUN — Execute with timeout:
   timeout 300 python experiment_XXX.py > results/experiment_XXX.log 2>&1
   Exit code 124 = timeout → treat as crash.

7. EVALUATE — Run judge.compare() against best_results.json.
   Print per-case breakdown and overall result.

8. LOG — Append to results.tsv:
   id  area  wins  losses  ties  latency_s  status  description

9. DECIDE — No destructive git commands needed:
   KEEP (won):
     git checkout <main-branch> && git merge exp/XXX && git branch -d exp/XXX
     Save new best_results.json.
   DISCARD (lost/tied):
     git checkout <main-branch> && git branch -D exp/XXX
   CRASH:
     Fix if trivial (same branch), else discard.

10. SYNTHESIZE — Every ~5 experiments, update FINDINGS.md with learnings.

11. REPEAT
```

## Results Logging

Tab-separated `results.tsv`:

```
id	area	wins	losses	ties	latency_s	status	description
001	baseline	0	0	0	42.1	baseline	Initial approach
002	query	8	2	2	38.5	keep	Benefit-aware query formulation
003	model	4	6	2	35.0	discard	Switch to smaller model
```

Status values: `baseline`, `keep`, `discard`, `crash`, `tie`

## Rules

1. **Loop until stopped** — never ask "should I continue?"
2. **Read before write** — understand full context before modifying
3. **One change per experiment** — atomic changes for clear attribution
4. **Mechanical evaluation only** — use the judge, never "looks good to me"
5. **Failures are cheap** — throwaway branches, no destructive commands
6. **Judge is sacred** — never modify judge.py during the loop
7. **Git is memory** — experiments are committed; git log + results.tsv track history
8. **When stuck, think harder** — re-read reference/, combine near-misses, try radical approaches, add new test cases

## Crash Recovery

| Failure | Response |
|---------|----------|
| Syntax error | Fix on same branch, re-run |
| API rate limit | Wait or switch API, re-run |
| Timeout (exit 124) | Simplify approach, discard |
| Import error | Fix deps, re-run |
| Judge error | Fix test data (not judge.py), re-run |

## When Stuck

- Re-read `reference/` for angles not yet explored
- Look at what `keep` experiments could be combined
- Try radical approaches (different APIs, architectures)
- Revisit `discard` experiments with a twist
- Add new edge case test cases
- Profile latency bottlenecks

## Example Invocation

User: `/autoresearch`

Agent asks:
1. What's the goal?
2. What test cases/inputs do you have?
3. What APIs/tools are available?
4. How should we evaluate? (LLM judge / metric / test suite / hybrid)
5. Any constraints or fixed rules?

Then generates workspace, confirms with user, and begins the loop.
