# Meta-Prompt Template V1.0

## Overview

This template defines the standard structure for all meta-prompts used in the Composable Development Infrastructure. Version 1.0 incorporates V2.1 hardening requirements including uncertainty assessment and run_id propagation.

---

## Template Structure

```yaml
meta_prompt:
  # HEADER (Required)
  header:
    id: "MP-{module}-{sequence}"  # e.g., MP-RESEARCH-001
    name: "Descriptive name for this meta-prompt"
    version: "1.0.0"
    module: [research, specification, build, validation, synthesis, extraction]
    target_tool: "claude | perplexity | chatprd | devin | github"  # V2.1: Target AI tool
    run_id: "${RUN_ID}"  # Injected at runtime, propagated to all outputs
    created_date: "YYYY-MM-DD"
    last_updated: "YYYY-MM-DD"
    author: "Creator name"
    effectiveness_rating: 0.0  # V2.1: Updated after each use (0.0-1.0)

  # CONTEXT (Required)
  context:
    purpose: |
      Clear, concise description of what this meta-prompt accomplishes.
      Maximum 3 sentences.

    inputs:
      - name: "input_name"
        type: "string | object | array"
        required: true | false
        description: "What this input provides"

    outputs:
      - name: "output_name"
        type: "string | object | array"
        description: "What this output contains"

    dependencies:
      - "List of required artifacts from previous phases"

  # ACTIVE LESSONS (Injected at runtime)
  active_lessons:
    max_count: 30  # V2.1: Reduced from 50
    injection_point: "Before main prompt"
    format: |
      ## Active Lessons for This Run
      {lessons_yaml}

  # MAIN PROMPT (Required)
  prompt:
    system_context: |
      You are operating as part of the Composable Development Infrastructure,
      a multi-agent orchestration system. Your role is {role_description}.

      Current run_id: ${RUN_ID}

      All outputs MUST include:
      - run_id in metadata
      - uncertainty assessment
      - timestamp

    task_description: |
      {Detailed description of the task}

    constraints:
      - "List of constraints and boundaries"

    output_format: |
      Your response MUST follow this structure:

      ## Metadata
      - run_id: ${RUN_ID}
      - timestamp: {ISO8601}
      - phase: {current_phase}

      ## Main Content
      {content_structure}

      ## Uncertainty Assessment (REQUIRED - V2.1)

      ### Unknowns
      List information that is missing or uncertain:
      1. {unknown_1}
      2. {unknown_2}
      ...

      ### Assumptions
      List assumptions this output relies on:
      1. {assumption_1} - Critical: yes/no - Validated: yes/no
      2. {assumption_2} - Critical: yes/no - Validated: yes/no
      ...

      ### Reversibility Assessment
      Is this decision easily reversible? (yes/no/partially)
      Explanation: {brief_explanation}

      ### Impact Scope
      What downstream phases does this affect?
      - {phase_list}

  # ADVERSARIAL PASS (Required for critical gates)
  adversarial_pass:
    enabled: true | false  # true for Gates 2, 4, 6, 7
    prompt: |
      Review this output and list 3 specific ways it could fail or cause
      problems downstream that the scoring rubric might miss. Focus on:
      - Hidden assumptions that could be wrong
      - Edge cases not covered
      - Integration risks with other phases
      - Failure modes that only appear at runtime

  # VALIDATION (Required)
  validation:
    schema_check: true  # Validate output against expected schema
    required_fields:
      - "run_id"
      - "timestamp"
      - "unknowns"
      - "assumptions"
      - "reversibility"

  # GATE INTEGRATION (If applicable)
  gate_integration:
    gate_name: "Gate name if this output feeds a gate"
    rubric_reference: "Link to rubric file"
    escalation_rules:
      unknowns_threshold: 3
      critical_assumption_requires_human: true
      irreversible_architectural_requires_human: true
```

---

## Usage Guidelines

### 1. Creating a New Meta-Prompt

1. Copy this template
2. Fill in all required sections
3. Assign appropriate ID following naming convention
4. Define clear inputs/outputs with types
5. Include uncertainty assessment section in output format
6. Specify if adversarial pass is enabled

### 2. Runtime Injection

At runtime, the orchestration layer will:
1. Generate run_id (UUID) at Phase 0
2. Inject run_id into all meta-prompts
3. Load active lessons (max 30) and inject before main prompt
4. Validate outputs against schema

### 3. Version Control

- All meta-prompts are stored in `/knowledge/meta-prompts/{module}/`
- Use semantic versioning (MAJOR.MINOR.PATCH)
- Document changes in commit messages
- Maintain effectiveness ratings after each use

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2025-12-06 | Initial V2.1 hardened template with uncertainty assessment, run_id, adversarial pass |

---

## Related Documents

- [Gate Logic V2.1](/specs/gates/gate-logic-v2.1.md)
- [Phase I/O Contracts](/specs/io-contracts/phase-io-contracts.md)
- [Lesson Schema](/knowledge/lessons/_schema.yaml)
