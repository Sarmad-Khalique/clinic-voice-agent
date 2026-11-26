# tests/

| Folder | What | Tier |
|--------|------|------|
| `api/` | Booking, idempotency, concurrent double-book | MVP → V1 |
| `agent/` | LiveKit text-mode tests: tool choice, arguments, confirmation, mind changes | V1 |
| `evals/` | LLM-judge evals (medical refusal, intent match) | V1 |
| `worker/` | Retry / crash recovery | V1 |
| `simulations/` | Simulated multi-turn callers | Stretch |
| `load/` | Concurrent-call load test | Stretch |
