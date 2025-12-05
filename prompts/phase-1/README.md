# Phase 1 Meta-Prompts: Discovery Track

## Overview

Phase 1 (Discovery) consists of three parallel research tracks that run simultaneously after Phase 0 initialization. Each track investigates a different aspect of the automation task.

## Tracks

| Track | File | Version | Lines | Purpose |
|-------|------|---------|-------|---------|
| Technical Research | `technical-research.md` | v1.2 | 889 | APIs, integrations, constraints |
| Workflow Research | `workflow-research.md` | v1.3 | 943 | Current state, pain points, stakeholders |
| Human-AI Research | `human-ai-research.md` | v1.2 | 1,037 | Automation boundaries, handoffs, oversight |

## Execution Model

```
Phase 0 Output (task_parsed)
           │
           ▼
    ┌──────┴──────┐
    │  PARALLEL   │
    ▼      ▼      ▼
Technical  Workflow  Human-AI
Research   Research  Research
    │      │      │
    └──────┬──────┘
           ▼
    Phase 2 Synthesis
```

## Common Design Patterns

All three tracks share:
- **45-minute time budget** (soft target)
- **3 core directives** per track
- **≤3 nesting levels** in output schema
- **Schema validation** before quality assessment
- **proceed_recommendation** field for gate decision

## Validation Status

All three meta-prompts have been adversarially reviewed and revised based on critical findings including:
- Classification system operationalizability
- Sparse input handling
- Verification protocol constraints
- RICEF conflict resolution
- Over-automation guards
- Citation faithfulness

## Usage

Inject `task_parsed` from Phase 0 into the `task_context` section of each meta-prompt, then execute all three tracks in parallel.
