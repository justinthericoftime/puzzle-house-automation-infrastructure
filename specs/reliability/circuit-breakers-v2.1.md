# Circuit Breakers Specification

## Metadata
- **Version:** 2.1.1
- **Status:** Draft
- **Last Updated:** 2025-12-05
- **Dependencies:** error-handling-v2.1.md, retry-logic-v2.1.md
- **Dependents:** gate-logic-v2.1.md
- **Owner:** Infrastructure Team
- **Stress Test Findings Addressed:** State persistence across runs (n8n workflow variables don't persist)

## Overview

This specification defines the circuit breaker system for the Composable Development Infrastructure. Circuit breakers protect the system from cascading failures by detecting error patterns and halting or pausing operations before they cause further damage. The design explicitly addresses the n8n constraint that workflow variables don't persist across runs.

## Platform Constraints

- **n8n Limitation:** Workflow variables do NOT persist across workflow executions
- **n8n Limitation:** No built-in circuit breaker primitives
- **GitHub API:** State must be stored externally in GitHub repository
- **Concurrency:** Multiple runs could theoretically conflict on state updates

## Scope

### In Scope
- Circuit breaker state persistence to GitHub
- All 6 circuit breaker definitions (CB-1 through CB-6)
- Precedence resolution algorithm
- Oscillation detection algorithm
- Breaker state machine
- n8n implementation patterns

### Out of Scope
- Error classification (see error-handling-v2.1.md)
- Retry logic (see retry-logic-v2.1.md)
- Gate decision logic (see gate-logic-v2.1.md)

## Specification

### State Persistence (CRITICAL)

```yaml
state_persistence:
  rationale: |
    CRITICAL: n8n workflow variables don't persist across runs.
    Circuit breaker state MUST be stored externally in GitHub.
    This enables cross-run tracking (e.g., CB-6 tracks Devin failures across runs).

  storage_location: "GitHub /state/circuit-breakers.json"

  state_schema:
    last_updated:
      type: "string"
      format: "ISO8601"
    updated_by_run:
      type: "string"
      format: "UUID"
    breakers:
      CB-1:
        state:
          type: "string"
          enum: ["closed", "open", "half_open"]
        trip_count:
          type: "integer"
        last_tripped:
          type: "string or null"
          format: "ISO8601"
        reset_at:
          type: "string or null"
          format: "ISO8601"
        trip_reason:
          type: "string or null"
      CB-2:
        state: "closed | open | half_open"
        trip_count: "integer"
        last_tripped: "ISO8601 or null"
        reset_at: "ISO8601 or null"
        accumulated_cost_usd: "number"
      CB-3:
        state: "closed | open | half_open"
        trip_count: "integer"
        last_tripped: "ISO8601 or null"
        reset_at: "ISO8601 or null"
        per_api_failure_counts: "object"
      CB-4:
        state: "closed | open | half_open"
        trip_count: "integer"
        last_tripped: "ISO8601 or null"
        reset_at: "ISO8601 or null"
        gate_decision_hashes: "array"
      CB-5:
        state: "closed | open | half_open"
        trip_count: "integer"
        last_tripped: "ISO8601 or null"
        reset_at: "ISO8601 or null"
      CB-6:
        state: "closed | open | half_open"
        trip_count: "integer"
        last_tripped: "ISO8601 or null"
        reset_at: "ISO8601 or null"
        consecutive_devin_failures: "integer"

  read_pattern:
    when: "At workflow start (Phase 0)"
    action: "Fetch circuit-breakers.json from GitHub"
    if_missing: "Initialize with all breakers in 'closed' state"
    if_parse_error: "Log error, initialize fresh state, flag for review"

  write_pattern:
    when: "Whenever breaker state changes"
    action: "Update circuit-breakers.json in GitHub"
    conflict_handling: "Use GitHub's SHA-based concurrency; retry on conflict"
    max_retries: 3

  initial_state:
    description: "Default state when no file exists"
    template:
      last_updated: "{{current_timestamp}}"
      updated_by_run: "{{run_id}}"
      breakers:
        CB-1: { state: "closed", trip_count: 0, last_tripped: null, reset_at: null }
        CB-2: { state: "closed", trip_count: 0, last_tripped: null, reset_at: null, accumulated_cost_usd: 0 }
        CB-3: { state: "closed", trip_count: 0, last_tripped: null, reset_at: null, per_api_failure_counts: {} }
        CB-4: { state: "closed", trip_count: 0, last_tripped: null, reset_at: null, gate_decision_hashes: [] }
        CB-5: { state: "closed", trip_count: 0, last_tripped: null, reset_at: null }
        CB-6: { state: "closed", trip_count: 0, last_tripped: null, reset_at: null, consecutive_devin_failures: 0 }
```

### Circuit Breaker State Machine

```yaml
state_machine:
  states:
    closed:
      description: "Normal operation; requests flow through"
      transitions:
        - to: "open"
          trigger: "trip_condition_met"
          action: "Execute breaker action, log trip"

    open:
      description: "Breaker tripped; requests blocked or modified"
      transitions:
        - to: "half_open"
          trigger: "reset_condition_met OR manual_reset"
          action: "Allow limited requests to test recovery"
        - to: "closed"
          trigger: "manual_force_reset"
          action: "Immediate reset (admin override)"

    half_open:
      description: "Testing if system has recovered"
      transitions:
        - to: "closed"
          trigger: "test_requests_succeed"
          action: "Resume normal operation"
        - to: "open"
          trigger: "test_requests_fail"
          action: "Re-trip breaker"

  automatic_reset:
    applies_to: [CB-3, CB-4, CB-5]
    behavior: "Transition to half_open after cooldown period"
    cooldown_periods:
      CB-3: "Next successful API call"
      CB-4: "End of current run"
      CB-5: "Start of new run"

  manual_reset_required:
    applies_to: [CB-1, CB-2, CB-6]
    behavior: "Remain open until human explicitly resets"
    reset_mechanism: "API call or manual file edit with review"
```

### Circuit Breaker Definitions

```yaml
circuit_breakers:
  CB-1:
    id: "CB-1"
    name: "Meta-Breaker"
    priority: 1
    description: "Trips when multiple other breakers have tripped, indicating systemic issues"

    trigger:
      condition: "More than 3 different circuit breakers have tripped in this run"
      detection: "Count distinct breaker IDs in run's trip log"
      evaluation_frequency: "After any breaker trips"

    action:
      immediate: "Halt workflow"
      notification: "Immediate escalation (SMS/call if configured)"
      workflow_state: "Stop and Error"
      message: "META-BREAKER: Multiple system failures detected. Architectural review required."

    reset:
      automatic: false
      manual_only: true
      reset_requires: "Human review and explicit reset command"

  CB-2:
    id: "CB-2"
    name: "Cost Threshold"
    priority: 2
    description: "Trips when run cost exceeds safety threshold"

    trigger:
      condition: "Run cost exceeds $100"
      detection: "Sum cost_this_phase across all completed phases"
      evaluation_frequency: "After each phase completes"

    action:
      immediate: "Pause workflow"
      notification: "Immediate escalation"
      workflow_state: "Checkpoint and wait for human approval"
      message: "COST THRESHOLD: Run has exceeded $100. Approve to continue or cancel."

    reset:
      automatic: false
      reset_requires: "Human approval to continue"

  CB-3:
    id: "CB-3"
    name: "Consecutive API Failures"
    priority: 3
    description: "Trips when same API fails repeatedly within a run"

    trigger:
      condition: "Same API fails 3 times consecutively WITHIN THIS RUN"
      detection: "Track per-API failure count; reset count on success"
      note: "Failure counting is within-run only. The count resets to 0 at the start of each new run."
      evaluation_frequency: "After each API call"

    action:
      immediate: "Stop retrying this API"
      notification: "Log and escalate"
      workflow_state: "Switch to fallback or pause"
      message: "API FAILURE: {api} has failed 3 consecutive times. Using fallback or pausing."

    reset:
      automatic: true
      reset_on: "Start of next run (count resets to 0) OR next successful call within current run"
      clarification: "The breaker STATE (open/closed) persists to GitHub, but the failure COUNT resets each run."

  CB-4:
    id: "CB-4"
    name: "Gate Inconsistency"
    priority: 4
    description: "Trips when gate produces different decisions on same input"

    trigger:
      condition: "Same gate produces different decisions on substantively same input"
      detection: |
        Hash the gate input (minus timestamps)
        If same hash seen before with different decision, trigger
      note: "Indicates non-deterministic or unstable gate logic"
      evaluation_frequency: "After each gate evaluation"

    action:
      immediate: "Escalate to human"
      notification: "Deferred (not urgent)"
      workflow_state: "Continue but flag for review"
      message: "GATE INCONSISTENCY: {gate} produced conflicting decisions. Review needed."

    reset:
      automatic: true
      reset_on: "End of run"

  CB-5:
    id: "CB-5"
    name: "Runtime Threshold"
    priority: 5
    description: "Trips when workflow runs too long"

    trigger:
      condition: "Workflow has been running for more than 4 hours"
      detection: "Compare current time to workflow start time"
      evaluation_frequency: "At each phase boundary"

    action:
      immediate: "Checkpoint current state"
      notification: "Alert operator"
      workflow_state: "Continue but notify"
      message: "RUNTIME: Workflow has exceeded 4 hours. Checkpointed at {phase}."

    reset:
      automatic: true
      reset_on: "New run"

  CB-6:
    id: "CB-6"
    name: "Build Failure Pattern"
    priority: 6
    description: "Trips when Devin fails builds repeatedly across runs"

    trigger:
      condition: "Devin has failed 3 builds consecutively ACROSS RUNS"
      detection: "Read from persistent state; track Devin failures"
      note: "This IS cross-run; requires state persistence to GitHub"
      evaluation_frequency: "At Phase 7 start and after each Devin task"

    action:
      immediate: "Block Phase 7"
      notification: "Escalate for integration review"
      workflow_state: "Cannot proceed to build phase"
      message: "DEVIN PATTERN: 3 consecutive build failures. Reassess Devin integration."

    reset:
      automatic: false
      reset_requires: "Human review of Devin integration"
```

### Precedence Resolution

```yaml
precedence_rules:
  description: |
    When multiple circuit breakers trigger simultaneously,
    only the highest-priority breaker's action is executed.
    All triggered breakers are logged for post-mortem.

  priority_order: [CB-1, CB-2, CB-3, CB-4, CB-5, CB-6]

  algorithm:
    step_1: "Collect all breakers whose conditions are currently true"
    step_2: "Sort by priority (1 = highest)"
    step_3: "Execute action for priority-1 breaker only"
    step_4: "Log all triggered breakers to error log"
    step_5: "Update state for all triggered breakers"

  example:
    scenario: "CB-2 (cost) and CB-5 (runtime) both trigger"
    resolution: "Execute CB-2 action (pause for approval), log both"
```

### Oscillation Detection

```yaml
oscillation_detection:
  scope: "Gate decisions across iterations within a single run"

  definition: |
    Oscillation occurs when gate scores alternate without convergence.
    Example: 75 → 68 → 78 → 70 → 76 (bouncing around threshold)

  detection_algorithm:
    window_size: 4
    minimum_iterations: 4

    detection_logic: |
      scores = [last 4 gate scores]
      deltas = [scores[i+1] - scores[i] for i in 0..2]

      signs = [sign(d) for d in deltas]
      is_oscillating = signs alternate (+, -, + or -, +, -)

      magnitude_decreasing = abs(deltas[-1]) < abs(deltas[0])

      OSCILLATION = is_oscillating AND NOT magnitude_decreasing

  action:
    trigger: "oscillation_detected == true"
    response: "Escalate to human"
    message: |
      OSCILLATION: Gate '{gate_name}' is not converging.
      Score history: {scores}
      Iteration is unproductive. Human decision required.

  storage:
    location: "Workflow variable during run"
    schema: "$json.gate_scores[gate_name] = [score1, score2, ...]"
```

## n8n Implementation Pattern

```yaml
n8n_implementation:
  state_read:
    node_type: "HTTP Request"
    method: "GET"
    url: "https://api.github.com/repos/{owner}/{repo}/contents/state/circuit-breakers.json"
    headers:
      Authorization: "Bearer {{$env.GITHUB_TOKEN}}"
      Accept: "application/vnd.github.v3+json"
    on_404: "Initialize fresh state"
    on_success: "Parse JSON, store in workflow variable"

  state_write:
    node_type: "HTTP Request"
    method: "PUT"
    url: "https://api.github.com/repos/{owner}/{repo}/contents/state/circuit-breakers.json"
    headers:
      Authorization: "Bearer {{$env.GITHUB_TOKEN}}"
      Accept: "application/vnd.github.v3+json"
    body:
      message: "Update circuit breaker state"
      content: "{{base64_encoded_json}}"
      sha: "{{current_file_sha}}"
    on_409_conflict: "Re-fetch, merge, retry (max 3x)"

  breaker_evaluation:
    description: "Code node to evaluate all breaker conditions"
    node_type: "Code"
    inputs:
      - current_state: "From state read"
      - run_context: "run_id, phase, cost, errors, etc."
    outputs:
      - triggered_breakers: "Array of breaker IDs"
      - highest_priority: "Breaker ID to act on"
      - updated_state: "New state to write"

  action_routing:
    node_type: "Switch"
    routing_field: "highest_priority_breaker"
    routes:
      CB-1: "Halt workflow node"
      CB-2: "Pause and notify node"
      CB-3: "Fallback routing node"
      CB-4: "Flag and continue node"
      CB-5: "Checkpoint and notify node"
      CB-6: "Block Phase 7 node"
      none: "Continue normal flow"

  workflow_structure:
    phase_0:
      - "Read circuit breaker state from GitHub"
      - "Check if any breakers are open (block if CB-1, CB-2, CB-6)"
      - "Initialize run-level tracking variables"
    each_phase:
      - "Evaluate breaker conditions after phase completes"
      - "Update state if any breaker trips"
      - "Route to appropriate action"
    phase_7:
      - "Check CB-6 before starting"
      - "Update Devin failure count after each task"
```

## Cross-References

| Related Spec | Relationship | Key Integration Points |
|--------------|--------------|------------------------|
| [error-handling-v2.1.md](./error-handling-v2.1.md) | depends on | Error counts feed CB-3 trip condition |
| [retry-logic-v2.1.md](./retry-logic-v2.1.md) | depends on | Retry exhaustion triggers error logging |
| [health-monitoring-v2.1.md](./health-monitoring-v2.1.md) | related | API health status informs CB-3 |
| [gate-logic-v2.1.md](../gates/gate-logic-v2.1.md) | depended by | Oscillation detection feeds gate logic |

## Validation Criteria

1. **State Persistence Test:** Trip a breaker, end workflow, start new workflow, verify breaker state persisted
2. **Precedence Test:** Trigger CB-2 and CB-5 simultaneously, verify only CB-2 action executes
3. **Oscillation Detection Test:** Feed alternating scores, verify oscillation is detected
4. **Cross-Run Tracking Test:** Fail Devin 3 times across runs, verify CB-6 trips
5. **Conflict Handling Test:** Simulate concurrent state updates, verify SHA-based retry works

## Changelog

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 2.1.1 | 2025-12-05 | Devin | Initial creation with GitHub state persistence (addressing n8n variable limitation) |
