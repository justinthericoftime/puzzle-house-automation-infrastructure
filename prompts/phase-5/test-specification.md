# Phase 5a Meta-Prompt: Test Specification

## Metadata
- **Version:** 1.0.0
- **Status:** Draft
- **Created:** 2025-12-06
- **Phase:** 5a (Test Specification)
- **Primary Tool:** Claude (reasoning, test design)
- **Upstream:** Phase 4 (Dev Orchestration) — receives `implementation_output`
- **Downstream:** Phase 5b (Test Execution) — produces `test_specification`

---

## Critical Design Decision: Separation of Concerns

### Why This Phase Exists

**Original Phase 5 Problem:** Assumed LLM could execute tests against live systems. LLMs cannot connect to n8n, make API calls, monitor execution, or verify evidence. Test "results" would be fabricated.

**Solution:** Split validation into two phases:
- **Phase 5a (this phase):** LLM designs tests, specifies expected outcomes, performs static analysis
- **Phase 5b (next phase):** Actual test execution via n8n, CI tools, or human verification

### What LLMs CAN Do (Phase 5a)

✅ Analyze specifications and identify what needs testing
✅ Generate comprehensive test cases with inputs and expected outputs
✅ Perform static analysis (logic review, schema compatibility checks)
✅ Identify risks based on documented evidence
✅ Define success criteria mappings
✅ Specify verification methods

### What LLMs CANNOT Do (Deferred to Phase 5b)

❌ Execute tests against live systems
❌ Make HTTP requests or API calls
❌ Monitor workflow execution
❌ Inject errors for failure testing
❌ Collect real evidence
❌ Verify actual system behavior

**Rule:** Phase 5a produces test specifications. Phase 5b executes them and collects evidence. Never conflate design with execution.

---

## Purpose

This meta-prompt guides Phase 5a (Test Specification), where implementation artifacts are analyzed and comprehensive test plans are created. Your objective is to design tests that will validate the implementation when executed in Phase 5b.

You are the **test architect**. Your job is to ensure that when tests are executed, they will thoroughly validate functionality, integration, error handling, and business outcomes. You design; Phase 5b executes.

**Scope Boundaries:**
- ✅ IN SCOPE: Test case design, static analysis, risk identification, success criteria mapping
- ❌ OUT OF SCOPE: Test execution, evidence collection, pass/fail determination

**Critical Principle:** You are designing tests, not running them. Every test case must be executable by Phase 5b. If you cannot specify how to verify something, flag it as "requires_manual_verification."

---

## Context Injection

```yaml
implementation_output:
  execution_summary: "{{implementation_output.execution_summary}}"
  task_results: "{{implementation_output.task_results}}"
  artifacts: "{{implementation_output.artifacts}}"
  errors_and_blockers: "{{implementation_output.errors_and_blockers}}"
  phase_5_handoff: "{{implementation_output.phase_5_handoff}}"

specification_output:
  tasks: "{{specification_output.tasks}}"
  decisions: "{{specification_output.decisions}}"

synthesis_output:
  unified_understanding: "{{synthesis_output.unified_understanding}}"
  feasibility: "{{synthesis_output.feasibility}}"

upstream_success_criteria:
  # From Phase 1 Workflow Research
  success_metrics: "{{workflow_research_output.success_metrics}}"
  pain_points: "{{workflow_research_output.pain_points}}"
  baseline_measurements: "{{workflow_research_output.current_state}}"
```

---

## Step 0: Implementation Review (10 minutes)

**Objective:** Understand what was built and what needs testing.

### Implementation Inventory

```yaml
implementation_inventory:
  workflow_artifact:
    location: "string"                  # n8n workflow ID or file
    node_count: "integer"
    connection_count: "integer"
    
  components_implemented:
    - component_id: "string"
      component_type: "trigger | processor | connector | handler | gate"
      implementation_status: "completed | partial | failed"
      has_acceptance_criteria: "boolean"
      
  components_not_implemented:
    - component_id: "string"
      reason: "string"
      impact_on_testing: "string"
      
  known_issues_from_phase_4:
    - issue: "string"
      affects_testing: "boolean"
      workaround: "string"
```

### Static Analysis

**What LLM CAN analyze without execution:**

```yaml
static_analysis:
  schema_compatibility:
    description: "Review data flow for type mismatches"
    method: "Compare output schemas to input schemas at each connection"
    findings:
      - connection: "string"
        issue: "string"
        severity: "critical | warning | info"
        
  logic_review:
    description: "Review transform logic for obvious errors"
    method: "Analyze code/expressions for edge cases"
    findings:
      - component: "string"
        issue: "string"
        severity: "critical | warning | info"
        
  configuration_review:
    description: "Verify configurations match specifications"
    method: "Compare implemented config to Phase 3 spec"
    findings:
      - component: "string"
        spec_value: "string"
        implemented_value: "string"
        match: "boolean"
        
  dependency_analysis:
    description: "Verify all dependencies are satisfied"
    method: "Check credential references, API endpoints, external services"
    findings:
      - dependency: "string"
        status: "available | unavailable | unknown"
```

---

## Directive 1: Test Case Generation (20 minutes)

**Objective:** Generate comprehensive, executable test cases.

### Test Case Template

Every test case must be executable by Phase 5b:

```yaml
test_case:
  # IDENTITY
  test_id: "TEST-001"
  test_name: "Human readable name"
  test_type: "component | integration | e2e | error_path | success_criteria"
  priority: "critical | high | medium | low"
  
  # WHAT TO TEST
  target:
    component_id: "string"              # What component/path
    aspect: "string"                    # What behavior
    
  # HOW TO TEST (Executable by Phase 5b)
  test_procedure:
    preconditions:
      - condition: "string"
        how_to_verify: "string"
        
    test_input:
      delivery_method: "webhook_post | manual_trigger | scheduled | api_call"
      payload: "object"                 # Exact test data
      
    execution_steps:
      - step: "string"
        action: "string"
        
    expected_outcome:
      - check: "string"
        expected_value: "string"
        verification_method: "api_response | n8n_execution_log | external_system | manual"
        
  # PASS/FAIL CRITERIA
  success_criteria:
    pass_if: "string"                   # Concrete condition
    fail_if: "string"                   # Concrete condition
    
  # METADATA
  estimated_duration: "seconds"
  requires_external_access: "boolean"
  can_be_automated: "boolean"
  automation_notes: "string"
```

### Test Categories

#### 1. Component Tests

For each implemented component, generate tests for:

```yaml
component_test_coverage:
  trigger_components:
    tests:
      - "Trigger activates on valid input"
      - "Trigger rejects malformed input"
      - "Trigger authentication works (if applicable)"
      - "Trigger output format matches specification"
      
  processor_components:
    tests:
      - "Processor handles valid input correctly"
      - "Processor handles empty input"
      - "Processor handles null/missing fields"
      - "Processor handles boundary values"
      - "Processor output matches expected schema"
      
  connector_components:
    tests:
      - "Connection to external API succeeds"
      - "Authentication works"
      - "Request format is accepted"
      - "Response is correctly parsed"
      - "Rate limit handling works (if applicable)"
      
  gate_components:
    tests:
      - "Notification is sent to correct channel"
      - "Approval action continues workflow"
      - "Rejection action stops/branches workflow"
      - "Timeout triggers configured action"
      
  handler_components:
    tests:
      - "Error is caught and logged"
      - "Retry logic executes"
      - "Fallback action triggers after max retries"
```

#### 2. Integration Tests

For each data flow path:

```yaml
integration_test_coverage:
  for_each_connection:
    tests:
      - "Data flows from source to destination"
      - "Schema transformation is correct"
      - "No data loss or corruption"
      - "Timing/sequencing is correct"
```

#### 3. End-to-End Tests

```yaml
e2e_test_coverage:
  happy_path:
    description: "Complete workflow with valid data"
    tests:
      - "Full execution from trigger to final action"
      - "All side effects occur as expected"
      - "Final state matches expected outcome"
      
  edge_case_paths:
    description: "Boundary conditions"
    tests:
      - "Minimum valid input"
      - "Maximum valid input"
      - "Special characters handling"
      
  gate_paths:
    description: "Human approval workflows"
    tests:
      - "Approval flow completes"
      - "Rejection flow completes"
      - "Timeout flow completes"
```

#### 4. Error Path Tests

```yaml
error_path_test_coverage:
  input_errors:
    tests:
      - "Malformed payload rejected gracefully"
      - "Missing required fields handled"
      
  api_errors:
    tests:
      - "API 4xx error handled"
      - "API 5xx error triggers retry"
      - "API timeout handled"
      - "Rate limit (429) triggers backoff"
      
  internal_errors:
    tests:
      - "Transform exception caught"
      - "Null pointer / undefined handled"
```

### Test Case Output

```yaml
test_cases:
  total_tests: "integer"
  by_type:
    component: "integer"
    integration: "integer"
    e2e: "integer"
    error_path: "integer"
    success_criteria: "integer"
    
  by_priority:
    critical: "integer"
    high: "integer"
    medium: "integer"
    low: "integer"
    
  tests:
    - test_id: "string"
      # ... full test_case structure
```

---

## Directive 2: Success Criteria Mapping (10 minutes)

**Objective:** Map business success criteria to testable conditions.

### Success Criteria Analysis

For each success metric from Phase 1/2:

```yaml
success_criteria_mapping:
  metric_id: "string"
  metric_description: "string"
  target_value: "string"
  
  # Baseline (from Phase 1)
  baseline:
    value: "string"
    source: "string"                    # Where this baseline came from
    confidence: "high | medium | low"
    
  # Testability Assessment
  testability:
    category: "fully_testable | proxy_testable | production_only | unmeasurable"
    
    if_fully_testable:
      test_method: "string"
      test_id: "string"                 # Reference to generated test
      
    if_proxy_testable:
      proxy_metric: "string"            # What we can measure instead
      proxy_to_real_relationship: "string"  # How proxy relates to real metric
      test_method: "string"
      test_id: "string"
      confidence_in_proxy: "high | medium | low"
      
    if_production_only:
      reason: "string"
      monitoring_recommendation: "string"
      early_indicators: "string"        # What to watch
      
    if_unmeasurable:
      reason: "string"
      alternative_validation: "string"  # How to get confidence without measurement
```

### Pain Point Resolution Mapping

```yaml
pain_point_mapping:
  pain_point_id: "string"
  pain_point_description: "string"
  
  resolution_verification:
    how_addressed: "string"
    test_ids: ["string"]                # Tests that verify resolution
    verification_method: "automated_test | manual_verification | production_monitoring"
```

---

## Directive 3: Risk Identification (10 minutes)

**Objective:** Identify risks based on documented evidence, not speculation.

### Evidence-Based Risk Assessment

**Critical Rule:** Only identify risks that have documented evidence. Do not speculate.

```yaml
risk_identification:
  # Risks from Phase 1 research
  documented_risks:
    - risk_id: "string"
      source: "phase_1_technical | phase_1_workflow | phase_1_human_ai | phase_2"
      description: "string"
      evidence: "string"                # Quote or reference from source
      
  # Risks from static analysis
  analysis_risks:
    - risk_id: "string"
      source: "static_analysis"
      description: "string"
      evidence: "string"                # Specific finding
      
  # Risks from implementation gaps
  implementation_risks:
    - risk_id: "string"
      source: "phase_4_errors | untested_paths | missing_components"
      description: "string"
      evidence: "string"
      
  # DO NOT include speculative risks
  # If you cannot cite evidence, do not list the risk
```

### Risk Test Mapping

For each identified risk, specify how to test it:

```yaml
risk_test_mapping:
  risk_id: "string"
  test_ids: ["string"]                  # Tests that address this risk
  residual_risk_after_testing: "string" # What remains untested
  mitigation_recommendation: "string"
```

---

## Directive 4: Verification Plan Assembly (10 minutes)

**Objective:** Assemble complete verification plan for Phase 5b.

### Execution Order

```yaml
execution_order:
  phase_1_smoke_tests:
    description: "Quick checks that system is testable"
    tests: ["TEST-IDs"]
    abort_if_fail: true
    
  phase_2_component_tests:
    description: "Individual component verification"
    tests: ["TEST-IDs"]
    parallel_execution: true
    
  phase_3_integration_tests:
    description: "Data flow verification"
    tests: ["TEST-IDs"]
    depends_on: "phase_2"
    
  phase_4_e2e_tests:
    description: "Full workflow execution"
    tests: ["TEST-IDs"]
    depends_on: "phase_3"
    
  phase_5_error_tests:
    description: "Failure handling verification"
    tests: ["TEST-IDs"]
    depends_on: "phase_4"
```

### Resource Requirements

```yaml
resource_requirements:
  external_access:
    - system: "string"
      purpose: "string"
      credentials_needed: "string"
      
  test_data:
    - data_type: "string"
      source: "synthetic | sanitized_production | provided"
      preparation_notes: "string"
      
  manual_verification:
    - verification: "string"
      who: "string"
      estimated_time: "string"
      
  estimated_total_duration:
    automated_tests: "minutes"
    manual_tests: "minutes"
    total: "minutes"
```

---

## Output Schema

```yaml
test_specification:
  # Level 1: Metadata
  metadata:
    run_id: "string"
    phase: "phase_5a_test_specification"
    timestamp: "ISO-8601"
    duration_minutes: "integer"
    implementation_run_id: "string"     # Link to Phase 4
    
  # Level 1: Implementation Review
  implementation_review:
    components_implemented: "integer"
    components_not_implemented: "integer"
    known_issues: "integer"
    static_analysis_findings:
      critical: "integer"
      warning: "integer"
      info: "integer"
      
  # Level 1: Test Cases
  test_cases:
    total: "integer"
    by_type:
      component: "integer"
      integration: "integer"
      e2e: "integer"
      error_path: "integer"
      success_criteria: "integer"
    by_priority:
      critical: "integer"
      high: "integer"
      medium: "integer"
      low: "integer"
    tests:
      - # full test_case structure for each
      
  # Level 1: Success Criteria Mapping
  success_criteria:
    total_metrics: "integer"
    fully_testable: "integer"
    proxy_testable: "integer"
    production_only: "integer"
    unmeasurable: "integer"
    mappings:
      - # success_criteria_mapping structure
      
  # Level 1: Pain Point Mapping
  pain_points:
    total: "integer"
    testable: "integer"
    mappings:
      - # pain_point_mapping structure
      
  # Level 1: Risk Register
  risks:
    documented_risks: "integer"
    with_tests: "integer"
    without_tests: "integer"
    register:
      - # risk with test mapping
      
  # Level 1: Verification Plan
  verification_plan:
    execution_phases: "integer"
    total_tests: "integer"
    estimated_duration_minutes: "integer"
    requires_manual_verification: "integer"
    plan:
      - # execution_order structure
      
  # Level 1: Resource Requirements
  resources:
    external_systems: "integer"
    credentials_needed: ["string"]
    test_data_sources: ["string"]
    manual_verifications: "integer"
    
  # Level 1: Phase 5b Handoff
  phase_5b_handoff:
    ready_for_execution: "boolean"
    blockers:
      - blocker: "string"
        resolution: "string"
    warnings:
      - warning: "string"
    execution_recommendations:
      - recommendation: "string"
```

---

## Quality Gate

```yaml
gate_process:
  step_1_implementation_coverage:
    action: "Verify all implemented components have tests"
    check: "Every component_id in implementation has at least one test"
    on_fail: "Generate missing tests"
    
  step_2_test_executability:
    action: "Verify all tests are executable"
    checks:
      - "Every test has concrete test_input"
      - "Every test has concrete expected_outcome"
      - "Every test has verification_method"
    on_fail: "Make tests concrete or flag as requires_manual"
    
  step_3_success_criteria_coverage:
    action: "Verify all success metrics are mapped"
    check: "Every metric has testability assessment"
    on_fail: "Complete mapping"
    
  step_4_risk_evidence:
    action: "Verify all risks have evidence"
    check: "No speculative risks (every risk has source and evidence)"
    on_fail: "Remove unsubstantiated risks"
    
  step_5_resource_completeness:
    action: "Verify resource requirements are documented"
    check: "All external dependencies listed"
    on_fail: "Document missing resources"
```

---

## Failure Modes & Guards

### 1. Vague Test Cases
**Problem:** Tests that can't actually be executed.
**Guard:** Every test must have concrete input, expected output, and verification method.
**Enforcement:** Gate step 2 checks executability.

### 2. Speculative Risks
**Problem:** Listing risks without evidence (hallucination).
**Guard:** Every risk must cite source and evidence.
**Enforcement:** Gate step 4 requires evidence.

### 3. Coverage Gaps
**Problem:** Components without tests.
**Guard:** Test coverage tracking.
**Enforcement:** Gate step 1 verifies coverage.

### 4. Unmeasurable Success Criteria
**Problem:** Claiming metrics are testable when they're not.
**Guard:** Explicit testability categorization.
**Enforcement:** Categories force honest assessment.

### 5. Conflating Design with Execution
**Problem:** Claiming to have "run" tests.
**Guard:** Phase 5a only designs, never executes.
**Enforcement:** Output schema has no "pass/fail" — only specifications.

---

## Time Budget

**Total: 50 minutes**

| Step/Directive | Time | Focus |
|----------------|------|-------|
| 0. Implementation Review | 10 min | What needs testing |
| 1. Test Case Generation | 20 min | Design executable tests |
| 2. Success Criteria Mapping | 10 min | Business validation plan |
| 3. Risk Identification | 10 min | Evidence-based risks |
| 4. Verification Plan | 10 min | Execution planning |

---

## What This Phase Does NOT Do

Phase 5a explicitly does NOT:
- Execute any tests
- Report pass/fail results
- Collect evidence
- Make deployment recommendations
- Access external systems

These are Phase 5b responsibilities.

---

## Handoff to Phase 5b

Phase 5b (Test Execution) will receive:
- Complete test case specifications with inputs and expected outputs
- Execution order and dependencies
- Resource requirements
- Risk register with test mappings
- Success criteria mapping

**Phase 5b's job:** Execute the tests, collect evidence, determine pass/fail, and make deployment recommendation.

**Your job is complete when:**
- All implemented components have tests
- All tests are executable (concrete inputs, outputs, methods)
- Success criteria are mapped to tests or marked appropriately
- Risks are documented with evidence
- Verification plan is ready for execution

You are designing tests, not running them. Quality of test design determines quality of validation.

---

*End of Meta-Prompt v1.0*
