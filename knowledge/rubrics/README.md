# Gate Evaluation Rubrics

## Purpose

This directory contains scoring rubrics used by gates to evaluate phase outputs. Each rubric defines weighted criteria for quality assessment.

## Contents

| File | Gate | Description |
|------|------|-------------|
| `gate-1-rubric.md` | Gate 1 | Evaluates Phase 1 (Discovery) output quality |
| `gate-2-rubric.md` | Gate 2 | Evaluates Phase 2 (Synthesis V1) — CRITICAL GATE |
| `gate-3-rubric.md` | Gate 3 | Evaluates Phase 3 (Stress Test V1) |
| `gate-4-rubric.md` | Gate 4 | Evaluates Phase 4 (Synthesis V2) — CRITICAL GATE |
| `gate-5-rubric.md` | Gate 5 | Evaluates Phase 5 (Stress Test V2) |
| `gate-6-rubric.md` | Gate 6 | Evaluates Phase 6 (Specification) — CRITICAL GATE |
| `gate-7-rubric.md` | Gate 7 | Evaluates Phase 7 (Build) |
| `gate-8-rubric.md` | Gate 8 | Evaluates Phase 8 (Retrospective) |

## Rubric Structure

Each rubric follows this schema:

```yaml
gate_id: "gate_N"
phase_evaluated: "phase_N_{name}"
criteria:
  - name: "Criterion Name"
    weight: 0.25
    description: "What this criterion evaluates"
    scoring_guide:
      excellent: { score: 100, description: "..." }
      good: { score: 80, description: "..." }
      acceptable: { score: 60, description: "..." }
      poor: { score: 40, description: "..." }
      fail: { score: 0, description: "..." }
```

## Usage

During gate evaluation, the n8n workflow:
1. Loads the rubric for the current gate
2. Sends phase output + rubric to Claude for scoring
3. Calculates weighted total score
4. Applies escalation rules based on score

## Related Specs

- [Gate Logic](/specs/gates/gate-logic-v2.1.md) — Defines escalation rules and thresholds
