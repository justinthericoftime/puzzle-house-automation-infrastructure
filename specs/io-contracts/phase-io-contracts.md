# Phase I/O Contracts Specification

## Metadata
- **Version:** 2.1.1
- **Status:** Draft
- **Last Updated:** 2025-12-05
- **Dependencies:** None (standalone universal metadata)
- **Dependents:** checkpoint-spec.md, all phase implementations
- **Owner:** Infrastructure Team
- **Stress Test Findings Addressed:** None (foundational schema specification)

## Overview

This specification defines the input/output contracts for all phases in the Composable Development Infrastructure. It establishes universal metadata requirements and phase-specific schemas to ensure consistent data flow between phases. These contracts enable reliable checkpointing, debugging, and cross-phase validation.

## Platform Constraints

- **n8n Limitation:** JSON is the primary data format; complex nested structures should be flattened where possible
- **GitHub Storage:** Large outputs may need to be stored as separate files with references
- **LLM Context:** Phase outputs must be serializable for injection into LLM prompts

## Scope

### In Scope
- Universal metadata schema (required for all phase outputs)
- Phase-specific schemas for all 9 phases (0, 0.5, 1-8)
- Validation rules (on input, on output)
- n8n implementation patterns

### Out of Scope
- Gate logic (see gate-logic-v2.1.md)
- Checkpoint storage (see checkpoint-spec.md)
- Error handling (see error-handling-v2.1.md)

## Specification

### Universal Metadata Schema

```yaml
universal_metadata:
  description: |
    Every phase output MUST include this metadata block.
    This enables tracing, debugging, and cost tracking across runs.

  required_fields:
    run_id:
      type: "string"
      format: "UUID v4"
      description: "Unique identifier for this run, generated at Phase 0"
      propagation: "Passed to all phases and outputs"
      example: "550e8400-e29b-41d4-a716-446655440000"

    phase:
      type: "string"
      pattern: "phase_{number}_{name}"
      description: "Which phase produced this output"
      examples:
        - "phase_0_initialization"
        - "phase_0.5_hardening"
        - "phase_1_research"
        - "phase_2_architecture"
        - "phase_3_specification"
        - "phase_4_planning"
        - "phase_5_implementation"
        - "phase_6_validation"
        - "phase_7_build"
        - "phase_8_synthesis"

    timestamp:
      type: "string"
      format: "ISO8601"
      description: "When this phase completed"
      example: "2025-12-05T19:00:00Z"

    duration_ms:
      type: "integer"
      description: "How long this phase took in milliseconds"
      example: 45000

    source_apis:
      type: "array"
      items: "string"
      description: "Which APIs were called during this phase"
      example: ["claude", "perplexity", "github"]

    cost_usd:
      type: "number"
      description: "Estimated cost of this phase in USD"
      calculation: "Sum of API call costs"
      example: 0.15

    health_summary:
      type: "object"
      description: "Health status at phase completion"
      schema:
        overall_status:
          type: "string"
          enum: ["healthy", "degraded", "critical"]
        api_statuses:
          type: "object"
          description: "Per-API health status"
        fallbacks_active:
          type: "array"
          description: "List of APIs using fallback"

  optional_fields:
    iteration_count:
      type: "integer"
      description: "If phase iterated, how many iterations"
      default: 1

    gate_score:
      type: "number or null"
      description: "Score from gate evaluation (if applicable)"

    warnings:
      type: "array"
      description: "Non-fatal issues encountered"

    parent_run_id:
      type: "string or null"
      description: "If this is a retry/continuation, reference to original run"
```

### Phase-Specific Schemas

#### Phase 0: Initialization

```yaml
phase_0_initialization:
  description: "System initialization and run setup"

  inputs:
    task_description:
      type: "string"
      required: true
      description: "What the user wants to build"

    user_context:
      type: "object"
      required: false
      description: "Additional context from user"

  outputs:
    metadata:
      type: "universal_metadata"
      required: true

    run_id:
      type: "string"
      format: "UUID v4"
      description: "Generated run identifier"

    task_parsed:
      type: "object"
      schema:
        original_description:
          type: "string"
        interpreted_goal:
          type: "string"
        success_criteria:
          type: "array"
        constraints_identified:
          type: "array"

    pre_flight_results:
      type: "object"
      schema:
        checks_passed:
          type: "integer"
        checks_failed:
          type: "integer"
        required_failures:
          type: "array"
        fallback_flags:
          type: "object"

    circuit_breaker_state:
      type: "object"
      description: "Initial state loaded from GitHub"

    active_lessons:
      type: "array"
      max_items: 30
      description: "Lessons loaded for this run"
```

#### Phase 0.5: Hardening

```yaml
phase_0_5_hardening:
  description: "System hardening and validation setup"

  inputs:
    phase_0_output:
      type: "phase_0_initialization.outputs"
      required: true

  outputs:
    metadata:
      type: "universal_metadata"
      required: true

    hardening_complete:
      type: "boolean"

    validation_rules_loaded:
      type: "array"
      description: "Validation rules active for this run"

    meta_prompts_loaded:
      type: "array"
      description: "Meta-prompts loaded and ready"

    heartbeat_configured:
      type: "object"
      schema:
        service:
          type: "string"
        url:
          type: "string"
        initial_ping_sent:
          type: "boolean"
```

#### Phase 1: Research

```yaml
phase_1_research:
  description: "Research and information gathering"

  inputs:
    task_parsed:
      type: "phase_0_initialization.outputs.task_parsed"
      required: true

    active_lessons:
      type: "array"
      required: false

  outputs:
    metadata:
      type: "universal_metadata"
      required: true

    research_findings:
      type: "array"
      items:
        topic:
          type: "string"
        source:
          type: "string"
          enum: ["perplexity", "claude", "github", "user_provided"]
        summary:
          type: "string"
        confidence:
          type: "string"
          enum: ["high", "medium", "low"]
        citations:
          type: "array"

    technology_recommendations:
      type: "array"
      items:
        category:
          type: "string"
        recommendation:
          type: "string"
        rationale:
          type: "string"
        alternatives:
          type: "array"

    constraints_discovered:
      type: "array"
      description: "New constraints found during research"

    uncertainty_assessment:
      type: "object"
      schema:
        unknowns:
          type: "array"
        assumptions:
          type: "array"
        reversibility:
          type: "string"
        impact_scope:
          type: "array"
```

#### Phase 2: Architecture

```yaml
phase_2_architecture:
  description: "System architecture design"

  inputs:
    research_findings:
      type: "phase_1_research.outputs.research_findings"
      required: true

    technology_recommendations:
      type: "phase_1_research.outputs.technology_recommendations"
      required: true

  outputs:
    metadata:
      type: "universal_metadata"
      required: true

    architecture_document:
      type: "object"
      schema:
        overview:
          type: "string"
        components:
          type: "array"
          items:
            name:
              type: "string"
            responsibility:
              type: "string"
            interfaces:
              type: "array"
        data_flow:
          type: "array"
        technology_stack:
          type: "object"
        deployment_model:
          type: "string"

    architecture_decisions:
      type: "array"
      items:
        decision_id:
          type: "string"
        title:
          type: "string"
        context:
          type: "string"
        decision:
          type: "string"
        consequences:
          type: "array"

    uncertainty_assessment:
      type: "object"
      required: true
```

#### Phase 3: Specification

```yaml
phase_3_specification:
  description: "Detailed technical specifications"

  inputs:
    architecture_document:
      type: "phase_2_architecture.outputs.architecture_document"
      required: true

    architecture_decisions:
      type: "phase_2_architecture.outputs.architecture_decisions"
      required: true

  outputs:
    metadata:
      type: "universal_metadata"
      required: true

    specifications:
      type: "array"
      items:
        spec_id:
          type: "string"
        title:
          type: "string"
        component:
          type: "string"
        content:
          type: "object"
          description: "Specification content (varies by type)"
        format:
          type: "string"
          enum: ["markdown", "yaml", "json", "openapi"]

    api_contracts:
      type: "array"
      items:
        endpoint:
          type: "string"
        method:
          type: "string"
        request_schema:
          type: "object"
        response_schema:
          type: "object"

    data_models:
      type: "array"
      items:
        model_name:
          type: "string"
        fields:
          type: "array"
        relationships:
          type: "array"

    uncertainty_assessment:
      type: "object"
      required: true
```

#### Phase 4: Implementation Planning

```yaml
phase_4_planning:
  description: "Implementation planning and task breakdown"

  inputs:
    specifications:
      type: "phase_3_specification.outputs.specifications"
      required: true

  outputs:
    metadata:
      type: "universal_metadata"
      required: true

    implementation_plan:
      type: "object"
      schema:
        phases:
          type: "array"
          items:
            phase_name:
              type: "string"
            tasks:
              type: "array"
            dependencies:
              type: "array"
            estimated_duration:
              type: "string"

    task_breakdown:
      type: "array"
      items:
        task_id:
          type: "string"
        title:
          type: "string"
        description:
          type: "string"
        component:
          type: "string"
        priority:
          type: "string"
          enum: ["critical", "high", "medium", "low"]
        estimated_effort:
          type: "string"
        dependencies:
          type: "array"

    risk_assessment:
      type: "array"
      items:
        risk_id:
          type: "string"
        description:
          type: "string"
        likelihood:
          type: "string"
        impact:
          type: "string"
        mitigation:
          type: "string"

    uncertainty_assessment:
      type: "object"
      required: true
```

#### Phase 5: Implementation

```yaml
phase_5_implementation:
  description: "Code implementation (pre-build)"

  inputs:
    task_breakdown:
      type: "phase_4_planning.outputs.task_breakdown"
      required: true

    specifications:
      type: "phase_3_specification.outputs.specifications"
      required: true

  outputs:
    metadata:
      type: "universal_metadata"
      required: true

    implementation_artifacts:
      type: "array"
      items:
        artifact_id:
          type: "string"
        type:
          type: "string"
          enum: ["code", "config", "script", "documentation"]
        path:
          type: "string"
        content_hash:
          type: "string"
        task_id:
          type: "string"

    code_review_notes:
      type: "array"
      items:
        file:
          type: "string"
        line_range:
          type: "string"
        note:
          type: "string"
        severity:
          type: "string"
          enum: ["info", "warning", "error"]

    test_coverage:
      type: "object"
      schema:
        unit_tests:
          type: "integer"
        integration_tests:
          type: "integer"
        coverage_percentage:
          type: "number"

    uncertainty_assessment:
      type: "object"
      required: true
```

#### Phase 6: Validation

```yaml
phase_6_validation:
  description: "Pre-build validation and review"

  inputs:
    implementation_artifacts:
      type: "phase_5_implementation.outputs.implementation_artifacts"
      required: true

  outputs:
    metadata:
      type: "universal_metadata"
      required: true

    validation_results:
      type: "object"
      schema:
        passed:
          type: "boolean"
        checks_run:
          type: "integer"
        checks_passed:
          type: "integer"
        checks_failed:
          type: "integer"

    validation_details:
      type: "array"
      items:
        check_name:
          type: "string"
        status:
          type: "string"
          enum: ["pass", "fail", "skip", "warn"]
        message:
          type: "string"

    build_readiness:
      type: "object"
      schema:
        ready:
          type: "boolean"
        blockers:
          type: "array"
        warnings:
          type: "array"

    uncertainty_assessment:
      type: "object"
      required: true
```

#### Phase 7: Build

```yaml
phase_7_build:
  description: "Build execution (Devin integration)"

  inputs:
    implementation_artifacts:
      type: "phase_5_implementation.outputs.implementation_artifacts"
      required: true

    build_readiness:
      type: "phase_6_validation.outputs.build_readiness"
      required: true

  outputs:
    metadata:
      type: "universal_metadata"
      required: true

    build_results:
      type: "object"
      schema:
        success:
          type: "boolean"
        build_id:
          type: "string"
        duration_ms:
          type: "integer"
        artifacts_produced:
          type: "array"

    devin_session:
      type: "object"
      schema:
        session_id:
          type: "string"
        tasks_completed:
          type: "integer"
        tasks_failed:
          type: "integer"
        logs_url:
          type: "string"

    test_results:
      type: "object"
      schema:
        total:
          type: "integer"
        passed:
          type: "integer"
        failed:
          type: "integer"
        skipped:
          type: "integer"

    deployment_artifacts:
      type: "array"
      items:
        artifact_type:
          type: "string"
        location:
          type: "string"
        checksum:
          type: "string"
```

#### Phase 8: Synthesis

```yaml
phase_8_synthesis:
  description: "Final synthesis and documentation"

  inputs:
    all_phase_outputs:
      type: "object"
      description: "Outputs from all previous phases"
      required: true

  outputs:
    metadata:
      type: "universal_metadata"
      required: true

    run_summary:
      type: "object"
      schema:
        total_duration_ms:
          type: "integer"
        total_cost_usd:
          type: "number"
        phases_completed:
          type: "integer"
        gates_passed:
          type: "integer"
        iterations_total:
          type: "integer"

    deliverables:
      type: "array"
      items:
        deliverable_id:
          type: "string"
        title:
          type: "string"
        type:
          type: "string"
        location:
          type: "string"
        description:
          type: "string"

    lessons_proposed:
      type: "array"
      items:
        lesson_id:
          type: "string"
        description:
          type: "string"
        type:
          type: "string"
          enum: ["success_pattern", "failure_pattern", "optimization"]
        evidence:
          type: "string"

    final_documentation:
      type: "object"
      schema:
        readme:
          type: "string"
        architecture_doc:
          type: "string"
        api_docs:
          type: "string"
        deployment_guide:
          type: "string"
```

### Validation Rules

```yaml
validation_rules:
  on_input:
    description: "Validate inputs before phase execution"

    rules:
      - rule: "required_fields_present"
        action: "Check all required fields exist"
        on_fail: "Abort phase with error"

      - rule: "type_validation"
        action: "Validate field types match schema"
        on_fail: "Abort phase with error"

      - rule: "run_id_propagation"
        action: "Verify run_id matches across inputs"
        on_fail: "Log warning, continue"

      - rule: "reference_integrity"
        action: "Verify referenced artifacts exist"
        on_fail: "Abort phase with error"

  on_output:
    description: "Validate outputs after phase execution"

    rules:
      - rule: "metadata_complete"
        action: "Verify universal metadata is present and complete"
        on_fail: "Add missing fields with defaults, log warning"

      - rule: "schema_compliance"
        action: "Validate output matches phase schema"
        on_fail: "Log error, flag for review"

      - rule: "cost_tracking"
        action: "Verify cost_usd is calculated"
        on_fail: "Estimate from API calls, log warning"

      - rule: "health_summary_present"
        action: "Verify health_summary is included"
        on_fail: "Generate from current state"
```

## n8n Implementation Pattern

```yaml
n8n_implementation:
  metadata_generation:
    node_type: "Code"
    purpose: "Generate universal metadata for phase output"
    placement: "At end of each phase, before output"

    code_pattern: |
      const startTime = $input.phase_start_time;
      const endTime = Date.now();

      const metadata = {
        run_id: $input.run_id,
        phase: $input.phase_name,
        timestamp: new Date().toISOString(),
        duration_ms: endTime - startTime,
        source_apis: $input.apis_called || [],
        cost_usd: $input.cost_accumulated || 0,
        health_summary: {
          overall_status: $input.health_status || 'healthy',
          api_statuses: $input.api_health || {},
          fallbacks_active: $input.fallback_flags || []
        }
      };

      return { metadata, ...phaseSpecificOutput };

  input_validation:
    node_type: "Code"
    purpose: "Validate phase inputs against schema"
    placement: "At start of each phase"

    code_pattern: |
      const requiredFields = $input.schema.required || [];
      const missing = requiredFields.filter(f => !$input.data[f]);

      if (missing.length > 0) {
        throw new Error(`Missing required fields: ${missing.join(', ')}`);
      }

      return { validated: true, data: $input.data };

  output_validation:
    node_type: "Code"
    purpose: "Validate phase outputs against schema"
    placement: "After phase completion, before checkpoint"

    code_pattern: |
      const output = $input.phase_output;

      if (!output.metadata) {
        output.metadata = generateDefaultMetadata();
      }

      if (!output.metadata.run_id) {
        console.warn('Missing run_id in metadata');
        output.metadata.run_id = $input.run_id;
      }

      return { validated_output: output };

  schema_storage:
    location: "GitHub /schemas/phase-io/"
    format: "JSON Schema files per phase"
    loading: "Fetch at Phase 0, cache in workflow variable"
```

## Cross-References

| Related Spec | Relationship | Key Integration Points |
|--------------|--------------|------------------------|
| [checkpoint-spec.md](../checkpoints/checkpoint-spec.md) | depended by | Phase outputs are checkpoint contents |
| [gate-logic-v2.1.md](../gates/gate-logic-v2.1.md) | related | Gate decisions reference phase outputs |
| [health-monitoring-v2.1.md](../reliability/health-monitoring-v2.1.md) | related | Health summary schema used in metadata |
| [error-handling-v2.1.md](../reliability/error-handling-v2.1.md) | related | Error entries reference phase context |
| [circuit-breakers-v2.1.md](../reliability/circuit-breakers-v2.1.md) | related | Cost tracking feeds CB-2 |

## Validation Criteria

1. **Metadata Test:** Verify all phase outputs include complete universal metadata
2. **Schema Test:** Validate sample outputs against JSON schemas
3. **Propagation Test:** Verify run_id is consistent across all phase outputs
4. **Cost Tracking Test:** Verify cost_usd accumulates correctly across phases
5. **Health Summary Test:** Verify health_summary reflects actual API status

## Changelog

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 2.1.1 | 2025-12-05 | Devin | Initial creation with universal metadata and all 9 phase schemas |
