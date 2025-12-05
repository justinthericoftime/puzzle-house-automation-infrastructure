# Workflow State

## Purpose

This directory stores persistent state that must survive across workflow runs. Unlike n8n workflow variables (which reset each execution), files here persist indefinitely.

## Contents

| File | Description | Updated By |
|------|-------------|------------|
| `circuit-breakers.json` | Current state of all 6 circuit breakers | Circuit breaker module |
| `pending-escalations.json` | Gate decisions queued for morning batch review | Gate logic module |
| `api-baselines.json` | Baseline latency metrics per API (for health monitoring) | Health monitoring module |

## Schema: circuit-breakers.json

```json
{
  "last_updated": "2025-12-05T19:00:00Z",
  "updated_by_run": "uuid-of-last-run",
  "breakers": {
    "CB-1": { "state": "closed", "trip_count": 0, "last_tripped": null },
    "CB-2": { "state": "closed", "trip_count": 0, "accumulated_cost_usd": 0 }
  }
}
```

## Schema: pending-escalations.json

```json
{
  "last_updated": "2025-12-05T19:00:00Z",
  "escalations": [
    {
      "gate_id": "gate_2",
      "run_id": "uuid",
      "queued_at": "2025-12-05T03:00:00Z",
      "rubric_score": 72,
      "artifact_path": "/artifacts/runs/{run_id}/phase_2_output.json"
    }
  ]
}
```

## Concurrency

Multiple runs could theoretically write to these files simultaneously. The workflow uses GitHub's SHA-based optimistic locking:
1. Read file (note SHA)
2. Modify contents
3. Write with SHA (fails if file changed)
4. On conflict: re-read, merge, retry (max 3x)

## Related Specs

- [Circuit Breakers](/specs/reliability/circuit-breakers-v2.1.md) — Defines breaker state schema
- [Gate Logic](/specs/gates/gate-logic-v2.1.md) — Defines pending escalation schema
