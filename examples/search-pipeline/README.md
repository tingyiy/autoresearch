# Example: Search Pipeline Optimization

Real-world example from a benefits platform. The goal was to build a web search pipeline that finds benefit-eligible services (therapists, summer camps, yoga classes) and correctly labels which benefit programs cover them.

## Setup

```
/autoresearch
Goal: Optimize service search — category matching, web search, result filtering
Judge: LLM pairwise (Gemini Pro compares result relevance per query)
Test cases: 12 queries × 3 users with different program enrollments
Constraints: Eligibility rules must match browser extension exactly
```

## Result

17 experiments over ~40 minutes. Key progression:

| Exp | Change | Result |
|-----|--------|--------|
| 001 | Baseline: Gemini Flash Lite + Tavily + Flash Lite filter | baseline |
| 002 | Stricter category prompt | tie |
| **003** | **Few-shot category examples + boundary detection** | **winner (5W/1L/6T)** |
| 005 | Tavily advanced depth | loss — worse snippets |
| 007 | Parallel execution | loss (same quality, Tavily variance) |
| 012 | GPT-4o-mini for categories | loss — 50% accuracy vs 94% |
| 014 | DuckDuckGo instead of Tavily | loss — lower quality |
| 015 | Gemini grounding search | loss — redirect URLs |
| 016 | All wins combined | ready (rate limited) |

## Files

- `program.md` — full specification
- `judge.py` — Gemini Pro pairwise judge
- `test_queries.json` — 12 queries with expected programs
- `reference/niceco_config.json` — company programs, categories, taxonomy
- `reference/eligibility_rules.md` — fixed rules matching browser extension

## Key Findings

1. **Few-shot examples in prompts** are the highest-impact optimization
2. **Gemini Flash Lite** matches Flash quality at lower cost for classification
3. **Tavily basic** beats advanced depth, DDG, and Gemini grounding
4. **LLM URL filter** is necessary — skipping it hurts quality
5. **Category accuracy** (83% → 94%) matters more than search quality
