# Judge Types

The judge is the fixed evaluation harness. It takes two result sets (A = current best, B = new experiment) and returns a winner. **Never modify judge.py during the experiment loop.**

## Interface

All judges implement the same interface:

```python
async def compare(
    cases: list[dict],           # Test cases from test_cases.json
    results_a: dict,             # Current best results (keyed by case ID)
    results_b: dict,             # New experiment results (keyed by case ID)
) -> tuple[str, ComparisonReport]:
    """Returns ("A", "B", or "tie") and a detailed report."""
```

## Type 1: Numeric Metric

For optimizing a single measurable number.

**Use when:** test coverage, latency, bundle size, accuracy percentage, error rate.

```python
def compare(cases, results_a, results_b):
    metric_a = compute_metric(results_a)
    metric_b = compute_metric(results_b)

    if direction == "higher":
        winner = "B" if metric_b > metric_a else "A" if metric_a > metric_b else "tie"
    else:  # lower is better
        winner = "B" if metric_b < metric_a else "A" if metric_a < metric_b else "tie"

    return winner, ComparisonReport(
        metric_a=metric_a, metric_b=metric_b,
        delta=metric_b - metric_a, winner=winner,
    )
```

**Guard (optional):** A secondary check that must always pass. If the metric improves but the guard fails (e.g., tests break), the experiment is discarded.

## Type 2: LLM Pairwise

For subjective quality where no single number captures "better."

**Use when:** search result relevance, generated text quality, classification accuracy with nuance, UX comparisons.

```python
async def compare(cases, results_a, results_b):
    comparisons = await asyncio.gather(*[
        llm_judge_one_case(case, results_a[case["id"]], results_b[case["id"]])
        for case in cases
    ])

    wins_a = sum(1 for c in comparisons if c.winner == "A")
    wins_b = sum(1 for c in comparisons if c.winner == "B")
    ties = sum(1 for c in comparisons if c.winner == "tie")

    if wins_b > wins_a: winner = "B"
    elif wins_a > wins_b: winner = "A"
    else: winner = "tie"

    return winner, ComparisonReport(
        wins_A=wins_a, wins_B=wins_b, ties=ties,
        query_comparisons=comparisons, winner=winner,
    )
```

**LLM judge prompt guidelines:**
- Use a capable model (e.g., Gemini Pro, GPT-4) as judge
- Provide: test case input, expected output, both result sets
- Ask for structured output: `{"winner": "A"|"B"|"tie", "reasoning": "..."}`
- Temperature = 0 for consistency
- The judge prompt is part of judge.py and must not change during the loop

## Type 3: Test Suite

For correctness-first evaluation.

**Use when:** all outputs must match expected values, and a secondary metric differentiates ties.

```python
def compare(cases, results_a, results_b):
    correct_a = sum(1 for c in cases if is_correct(results_a[c["id"]], c["expected"]))
    correct_b = sum(1 for c in cases if is_correct(results_b[c["id"]], c["expected"]))

    if correct_b > correct_a:
        return "B", report  # More correct = wins
    if correct_b == correct_a:
        # Tie-break on secondary metric (e.g., latency)
        if secondary_b > secondary_a:
            return "B", report
    return "A", report
```

## Type 4: Hybrid

Combine multiple criteria with configurable weights.

**Use when:** you care about multiple dimensions simultaneously.

```python
def compare(cases, results_a, results_b):
    # Example: 60% correctness + 30% quality + 10% speed
    score_a = (
        0.6 * correctness(results_a) +
        0.3 * quality_score(results_a) +
        0.1 * speed_score(results_a)
    )
    score_b = (
        0.6 * correctness(results_b) +
        0.3 * quality_score(results_b) +
        0.1 * speed_score(results_b)
    )

    winner = "B" if score_b > score_a else "A" if score_a > score_b else "tie"
    return winner, report
```

## Choosing a Judge Type

| Situation | Judge Type |
|-----------|-----------|
| One number to optimize | Metric |
| "Is output A or B better?" | LLM Pairwise |
| Must pass tests, then optimize | Test Suite |
| Multiple goals (accuracy + speed) | Hybrid |
| Not sure | Start with LLM Pairwise — most flexible |

## Results Logging

All judge types log to the same results.tsv format:

```
id	area	wins	losses	ties	latency_s	status	description
```

- For metric judges: wins=1 if improved, losses=1 if worsened, ties=0
- For pairwise judges: wins/losses/ties counted per test case
- Status: `baseline` | `keep` | `discard` | `crash` | `tie`
