<div align="center">

# Autoresearch

**Turn [Claude Code](https://docs.anthropic.com/en/docs/claude-code) into an autonomous research engine.**

Inspired by [Karpathy's autoresearch](https://github.com/karpathy/autoresearch) — constraint + evaluation + autonomous iteration = compounding gains.

[![Claude Code Skill](https://img.shields.io/badge/Claude_Code-Skill-blue?logo=anthropic&logoColor=white)](https://docs.anthropic.com/en/docs/claude-code)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

<br>

*"Define the GOAL → Set the JUDGE → Run the LOOP → Wake up to results"*

<br>

[How It Works](#how-it-works) · [Quick Start](#quick-start) · [Judge Types](#judge-types) · [Examples](#examples)

</div>

---

## Why This Exists

Karpathy's [autoresearch](https://github.com/karpathy/autoresearch) showed that autonomous iteration against a fixed metric produces compounding gains overnight. His implementation targets ML training with a single numeric metric (`val_bpb`).

**But many real problems don't have a single number.** Search result quality, LLM output relevance, UX improvements — these need richer evaluation. Is result set A better than B? Does this prompt produce more accurate classifications?

**This project generalizes autonomous research with a pluggable judge.** The judge can be:
- A numeric metric (test coverage, latency, bundle size)
- An LLM comparing two outputs head-to-head
- A test suite checking correctness
- Any combination with weights

The loop, branching, and experiment management stay the same. The judge is what changes.

---

## How It Works

```
LOOP (forever or N iterations):
  1. Review state — git log, results.tsv, what worked/failed
  2. Hypothesize — pick one focused change
  3. Branch — throwaway branch (exp/XXX)
  4. Implement — write experiment script
  5. Commit & Run — execute with timeout
  6. Evaluate — judge compares new vs current best
  7. Keep or Discard:
     - Won → merge branch, save as new best
     - Lost → delete branch
  8. Log result → results.tsv
  9. Repeat
```

Every improvement stacks. Failures are discarded via branch deletion — no `git revert`, no polluted history. The main branch only accumulates wins.

### Key Differences from Karpathy's Approach

| | Karpathy | This Project |
|---|---|---|
| **Scope** | Single file (`train.py`) | Self-contained workspace directory |
| **Metric** | Single number (`val_bpb`) | Pluggable judge (numeric, LLM, test suite, hybrid) |
| **Failure handling** | `git revert` (stays in history) | Branch deletion (clean history) |
| **Evaluation** | Lower is better | Pairwise comparison — A vs B |
| **Experiments** | Modify in-place | Each experiment is a standalone script |
| **Context** | Agent reads codebase | Pre-captured reference material in sandbox |

### 8 Rules

| # | Rule |
|---|------|
| 1 | **Loop until stopped** — never ask "should I continue?" |
| 2 | **Read before write** — understand full context first |
| 3 | **One change per experiment** — atomic for clear attribution |
| 4 | **Mechanical evaluation only** — use the judge, never subjective |
| 5 | **Failures are cheap** — throwaway branches, no destructive commands |
| 6 | **Judge is sacred** — never modify judge.py during the loop |
| 7 | **Git is memory** — results.tsv + git log track everything |
| 8 | **When stuck, think harder** — re-read reference/, combine wins, try radical approaches |

---

## Quick Start

### Installation

Copy the plugin into your Claude Code configuration:

```bash
# Clone
git clone https://github.com/tingyiy/autoresearch.git /tmp/autoresearch

# Install globally
cp -r /tmp/autoresearch/claude-plugin/* ~/.claude/
```

Or copy to a project's `.claude/` directory for project-scoped installation.

### Usage

```
/autoresearch
```

Claude will ask you:
1. **What's the goal?** — what you're trying to achieve
2. **What test cases do you have?** — inputs with expected outputs
3. **How should we evaluate?** — LLM judge, metric, test suite, or hybrid
4. **What APIs/tools are available?** — LLM providers, search APIs, etc.
5. **Any constraints?** — fixed rules, off-limits files

Then it generates the workspace, confirms with you, and begins the autonomous loop.

---

## Judge Types

The judge is the fixed evaluation harness. It compares two result sets and picks a winner. **Never modified during the loop.**

### Numeric Metric

For optimizing a single number — test coverage, latency, bundle size:

```python
# judge.py
def compare(cases, results_a, results_b):
    metric_a = compute_metric(results_a)  # e.g., coverage %
    metric_b = compute_metric(results_b)
    winner = "B" if metric_b > metric_a else "A" if metric_a > metric_b else "tie"
    return winner, report
```

### LLM Pairwise

For subjective quality — search results, generated text, classifications:

```python
# judge.py
async def compare(cases, results_a, results_b):
    """LLM compares A vs B per test case, picks winner."""
    comparisons = [llm_judge(case, a, b) for case, a, b in zip(...)]
    wins_a = sum(1 for c in comparisons if c.winner == "A")
    wins_b = sum(1 for c in comparisons if c.winner == "B")
    winner = "B" if wins_b > wins_a else "A" if wins_a > wins_b else "tie"
    return winner, report
```

### Test Suite

For correctness — all tests must pass, then optimize a secondary metric:

```python
# judge.py
def compare(cases, results_a, results_b):
    pass_a, pass_b = count_passing(results_a), count_passing(results_b)
    if pass_b > pass_a: return "B", report
    if pass_b == pass_a and secondary_metric_b > secondary_metric_a: return "B", report
    return "A", report
```

### Hybrid

Combine multiple criteria with weights:

```python
# judge.py
def compare(cases, results_a, results_b):
    """60% correctness + 30% quality + 10% latency"""
    score_a = 0.6 * correct_a + 0.3 * quality_a + 0.1 * speed_a
    score_b = 0.6 * correct_b + 0.3 * quality_b + 0.1 * speed_b
    return ("B" if score_b > score_a else "A"), report
```

---

## Workspace Structure

```
<workspace>/
├── program.md          # Goal, constraints, evaluation criteria
├── judge.py            # Evaluation harness (NEVER modify during loop)
├── test_cases.json     # Test inputs with expected outputs
├── reference/          # Pre-captured context (docs, code snapshots, data)
├── results/            # Experiment outputs (JSON, logs)
├── results.tsv         # Experiment log
├── best_results.json   # Current best results
├── .env                # API keys (gitignored)
└── experiment_XXX.py   # Individual experiment scripts
```

### results.tsv

```
id	area	wins	losses	ties	latency_s	status	description
001	baseline	0	0	0	42.1	baseline	Initial approach
002	query	5	1	6	38.5	keep	Few-shot prompt examples
003	model	4	6	2	35.0	discard	Smaller model — worse quality
004	arch	7	3	2	12.0	keep	Parallel execution + caching
```

---

## Examples

### Search Pipeline Optimization

Goal: Find the best web search pipeline for a benefits platform.
Judge: LLM pairwise (Gemini Pro compares result sets per query).

```
/autoresearch
Goal: Optimize service search pipeline — category matching, web search, result filtering
Test cases: 12 benefit queries (summer camps, therapist, yoga, etc.) × 3 users with different program enrollments
Judge: LLM pairwise — Gemini compares result relevance, location accuracy, eligibility correctness
APIs: Gemini (LLM), Tavily (search), Weaviate (embeddings)
Constraints: Eligibility rules are fixed (same as browser extension)
```

Result: 17 experiments → best pipeline found (few-shot category matching + Tavily basic + LLM URL filter). Category accuracy went from 83% to 94%.

### Test Coverage Improvement

Goal: Increase test coverage from 72% to 90%.
Judge: Numeric metric (higher is better).

```
/autoresearch
Goal: Increase test coverage to 90%
Scope: src/**/*.test.ts, src/**/*.ts
Metric: coverage % (higher is better)
Verify: npm test -- --coverage | grep "All files" | awk '{print $10}'
Guard: npm test
Iterations: 50
```

### LLM Prompt Optimization

Goal: Improve classification accuracy on a benchmark.
Judge: Hybrid (80% accuracy + 20% latency).

```
/autoresearch
Goal: Improve product categorization accuracy
Test cases: 200 products with known categories
Judge: Hybrid — 80% accuracy (exact match) + 20% latency (lower is better)
Constraints: Must use Gemini Flash Lite, budget < $0.01/call
```

---

## Crash Recovery

| Failure | Response |
|---------|----------|
| Syntax error | Fix on same branch, re-run |
| API rate limit | Wait or switch provider |
| Timeout (5 min) | Simplify approach, discard |
| Judge error | Fix test data (not judge.py) |

---

## Attribution

- [Karpathy's autoresearch](https://github.com/karpathy/autoresearch) — the original concept of autonomous ML research
- [uditgoenka/autoresearch](https://github.com/uditgoenka/autoresearch) — generalized single-metric approach for Claude Code

This project extends the concept with pluggable evaluation (pairwise judges, hybrid metrics) and sandboxed experiment management via throwaway branches.

---

## License

MIT
