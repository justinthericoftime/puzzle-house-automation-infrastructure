# Workflow Logs

## Purpose

This directory stores structured logs from workflow executions. Logs are organized by run_id for easy debugging and post-mortem analysis.

## Structure

```
/logs/
└── {run_id}/
    ├── errors/
    │   └── {timestamp}_{error_id}.json
    ├── gates/
    │   └── {gate_id}.json
    └── health/
        └── {phase}_health_summary.json
```

## Contents

| Subdirectory | Contents | Written By |
|--------------|----------|------------|
| `/errors/` | Error entries (classification, sink results, resolution) | Error handling module |
| `/gates/` | Gate decisions (scores, uncertainty assessment, action taken) | Gate logic module |
| `/health/` | Health summaries per phase (API status, latencies, fallbacks) | Health monitoring module |

## Retention

Logs follow the same retention policy as checkpoints:
- Most recent 10 runs: Keep all logs
- Runs 11-50: Keep gate decisions only
- Runs 51+: Delete all logs

## Usage

For debugging a failed run:
1. Navigate to `/logs/{run_id}/`
2. Check `/errors/` for any logged errors
3. Check `/gates/` to see decision history
4. Check `/health/` to see API status at each phase

## Related Specs

- [Error Handling](/specs/reliability/error-handling-v2.1.md) — Defines error entry schema
- [Gate Logic](/specs/gates/gate-logic-v2.1.md) — Defines gate decision schema
- [Health Monitoring](/specs/reliability/health-monitoring-v2.1.md) — Defines health summary schema
