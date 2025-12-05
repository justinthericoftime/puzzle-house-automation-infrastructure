# Phase 1 Meta-Prompts

## Purpose

This directory contains meta-prompt templates for Phase 1 (Discovery) of the workflow. Phase 1 runs three parallel research tracks, each with its own meta-prompt.

## Contents

| File | Description |
|------|-------------|
| `technical-research.md` | Meta-prompt for technical research track (APIs, integrations, technologies) |
| `workflow-research.md` | Meta-prompt for workflow research track (current processes, pain points) |
| `human-ai-research.md` | Meta-prompt for human-AI collaboration research track (handoff points, automation boundaries) |

## Usage

During Phase 1, the n8n workflow:
1. Loads the appropriate meta-prompt from this directory
2. Injects task-specific context (from Phase 0 output)
3. Sends to the designated AI tool (Perplexity or Claude)
4. Collects and synthesizes results

## Naming Convention

Meta-prompts follow the pattern: `{track-name}-research.md`

## Related Specs

- [Phase I/O Contracts](/specs/io-contracts/phase-io-contracts.md) — Defines Phase 1 output schema
- [Gate Logic](/specs/gates/gate-logic-v2.1.md) — Defines Gate 1 evaluation criteria
