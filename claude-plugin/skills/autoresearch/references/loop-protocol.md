# Autonomous Loop Protocol

## Pre-Loop: Workspace Setup

Before the loop begins, the workspace must contain:

```
<workspace>/
├── program.md          # Goal, constraints, areas to explore
├── judge.py            # Fixed evaluation harness
├── test_cases.json     # Test inputs with expected outputs
├── reference/          # Pre-captured context
├── results/            # Output directory
├── results.tsv         # Header row: id\tarea\twins\tlosses\tties\tlatency_s\tstatus\tdescription
└── .env                # API keys (gitignored)
```

## The Loop

```
LOOP FOREVER (or N iterations if bounded):

1. REVIEW
   - Read results.tsv — what experiments ran, what won/lost
   - Read FINDINGS.md if it exists — accumulated learnings
   - Read git log — recent changes
   - Identify: most promising direction OR unexplored area

2. HYPOTHESIZE
   - Decide what to try next
   - Document hypothesis as a comment at top of experiment script
   - One focused change per experiment

3. BRANCH
   - Create throwaway branch:
     git checkout -b exp/XXX
   - XXX = sequential experiment number (001, 002, ...)

4. IMPLEMENT
   - Write experiment_XXX.py (self-contained script)
   - Must load test cases, run pipeline, output results, call judge

5. COMMIT
   - git add -A
   - git commit -m "experiment XXX: <description>"

6. RUN
   - timeout 300 python experiment_XXX.py > results/experiment_XXX.log 2>&1
   - Exit code 124 = timeout → crash
   - Check log for errors

7. EVALUATE
   - Run judge.compare() against best_results.json
   - Print per-case breakdown (winner + reasoning)
   - Print overall: wins/losses/ties

8. LOG
   - Append to results.tsv:
     id\tarea\twins\tlosses\tties\tlatency_s\tstatus\tdescription
   - Status: baseline | keep | discard | crash | tie

9. DECIDE
   Keep (won overall):
     git checkout <main-branch>
     git merge exp/XXX
     git branch -d exp/XXX
     Save new best_results.json

   Discard (lost or tied):
     git checkout <main-branch>
     git branch -D exp/XXX

   Crash:
     Fix if trivial (on same branch), re-run
     If fundamental, discard

10. SYNTHESIZE
    - Every ~5 experiments, update FINDINGS.md
    - Summarize: what works, what doesn't, emerging patterns
    - Note: winning experiment numbers and their key innovations

11. REPEAT
```

## Experiment 001: Baseline

The first experiment is special:
- No prior best_results.json exists
- Run the baseline approach
- Save output as best_results.json
- Status = "baseline"
- All future experiments compare against this

## Branching Strategy

```
main-branch (accumulates wins only)
├── exp/001 → baseline → merge
├── exp/002 → won → merge
├── exp/003 → lost → delete
├── exp/004 → crash → delete
├── exp/005 → won → merge
└── ...
```

The main branch has a clean history of only successful experiments.
Failed experiments vanish — no reverts, no noise.

## Timeout

Each experiment has a hard 5-minute timeout via `timeout 300`.
Exit code 124 = killed by timeout → treat as crash/discard.

To stay under the limit:
- Run a representative subset of test cases (e.g., 8 of 12)
- Avoid serial API calls when parallel is possible
- Cache expensive results across test cases

## Never Stop

Once the loop begins, do NOT pause to ask the human.
Do NOT ask "should I continue?" or "is this a good stopping point?".
The human expects you to work indefinitely until manually stopped.

If you run out of ideas:
- Re-read reference/ for unexplored angles
- Combine winning experiments
- Try radical approaches (different APIs, architectures)
- Revisit discarded experiments with a twist
- Add new edge case test cases
