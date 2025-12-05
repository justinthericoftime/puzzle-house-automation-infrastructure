# Architecture Decision Records (ADRs)

## Purpose

This directory contains Architecture Decision Records documenting significant technical decisions made during the design and evolution of the Composable Development Infrastructure.

## What is an ADR?

An ADR captures:
- **Context:** The situation and constraints at decision time
- **Decision:** What we decided
- **Consequences:** The implications (positive and negative)
- **Status:** Proposed, Accepted, Deprecated, or Superseded

## Contents

| ADR | Title | Status |
|-----|-------|--------|
| ADR-001 | Manager Pattern for Workflow Orchestration | Pending |
| ADR-002 | Sequential Error Sinks (vs Parallel) | Pending |
| ADR-003 | External Heartbeat Architecture | Pending |
| ADR-004 | Uncertainty-Based Gating (vs Confidence-Based) | Pending |

## ADR Template

```markdown
# ADR-{number}: {Title}

## Status
{Proposed | Accepted | Deprecated | Superseded by ADR-XXX}

## Context
{What is the issue? What constraints exist?}

## Decision
{What did we decide?}

## Consequences

### Positive
- {Benefit 1}
- {Benefit 2}

### Negative
- {Tradeoff 1}
- {Tradeoff 2}

### Neutral
- {Observation}

## References
- {Link to relevant spec or discussion}
```

## When to Write an ADR

Write an ADR when:
- Choosing between multiple valid approaches
- Making a decision that's hard to reverse
- Deviating from common patterns
- Learning from a failure (post-mortem ADR)

## Related

- [V2.1 Execution Plan](/docs/) — High-level architecture overview
- [Specs](/specs/) — Detailed technical specifications
