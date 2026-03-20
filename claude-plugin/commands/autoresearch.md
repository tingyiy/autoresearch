---
name: autoresearch
description: "Autonomous research loop — iteratively experiment and improve with pluggable evaluation."
argument-hint: "[Goal: <text>] [Judge: metric|llm|test|hybrid] [Iterations: N]"
---

EXECUTE IMMEDIATELY — do not deliberate before reading the protocol.

## Argument Parsing

Extract from $ARGUMENTS:
- `Goal:` — what to achieve
- `Judge:` — evaluation type (metric, llm, test, hybrid)
- `Iterations:` — max iterations (omit for unbounded)

## Execution

1. Read the skill: `.claude/skills/autoresearch/SKILL.md`
2. Read the loop protocol: `.claude/skills/autoresearch/references/loop-protocol.md`
3. Read the judge reference: `.claude/skills/autoresearch/references/judge-types.md`
4. If Goal and Judge are extracted → proceed to workspace setup
5. If missing → ask the user interactively
6. Generate workspace (program.md, judge.py, test_cases.json, reference/)
7. Confirm with user
8. Execute the autonomous loop

IMPORTANT: Start executing immediately. Never stop until interrupted or max iterations reached.
