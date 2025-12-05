# Checkpoint Specification

## Metadata
- **Version:** 2.1.1
- **Status:** Draft
- **Last Updated:** 2025-12-05
- **Dependencies:** phase-io-contracts.md
- **Dependents:** None (terminal spec)
- **Owner:** Infrastructure Team
- **Stress Test Findings Addressed:** GitHub rate limits on checkpoint writes

## Overview

This specification defines the checkpoint system for the Composable Development Infrastructure. Checkpoints enable workflow resumption after failures, provide audit trails, and support debugging. The design explicitly addresses GitHub API rate limits to ensure reliable checkpoint persistence.

## Platform Constraints

- **GitHub API Rate Limits:** 5000 requests/hour for authenticated users; checkpoint writes count against this limit
- **GitHub File Size:** Individual files limited to 100MB; checkpoint data must be structured accordingly
- **n8n Limitation:** No built-in checkpoint primitives; must implement via GitHub API
- **Concurrency:** Multiple runs could theoretically conflict on checkpoint writes

## Scope

### In Scope
- Checkpoint triggers (automatic, manual, task-level)
- Checkpoint contents schema
- GitHub rate limit awareness and handling
- Atomic write pattern (write-then-verify)
- Resume capability specification
- Retention policy
- n8n implementation patterns

### Out of Scope
- Phase I/O schemas (see phase-io-contracts.md)
- Error handling (see error-handling-v2.1.md)
- Circuit breaker state (see circuit-breakers-v2.1.md)

## Specification

### Checkpoint Triggers

```yaml
checkpoint_triggers:
  automatic:
    description: "Checkpoints created automatically by the system"

    after_each_phase:
      trigger: "Phase completion (before gate evaluation)"
      contents: "Full phase output + metadata"
      priority: "high"
      note: "Primary recovery point"

    after_gate_decision:
      trigger: "Gate evaluation complete"
      contents: "Gate decision + scores + uncertainty assessment"
      priority: "high"
      note: "Captures decision context for audit"

    on_circuit_breaker_trip:
      trigger: "Any circuit breaker trips"
      contents: "Current state + breaker context"
      priority: "critical"
      note: "Enables investigation of trip cause"

  manual:
    description: "Checkpoints triggered by explicit request"

    on_escalation:
      trigger: "Escalation to human"
      contents: "Full context for human review"
      priority: "high"
      note: "Human needs complete picture"

    on_iteration_start:
      trigger: "Before iteration attempt"
      contents: "Pre-iteration state"
      priority: "medium"
      note: "Enables rollback if iteration fails"

    on_cost_threshold:
      trigger: "Cost approaches threshold (80%)"
      contents: "Cost breakdown + current state"
      priority: "high"
      note: "Snapshot before potential halt"

  task_level:
    description: "Fine-grained checkpoints within phases"

    after_expensive_operation:
      trigger: "After Devin task, long API call"
      contents: "Operation result + partial state"
      priority: "medium"
      note: "Avoid re-running expensive operations"

    before_destructive_operation:
      trigger: "Before file deletion, schema migration"
      contents: "Pre-operation state"
      priority: "high"
      note: "Enables rollback"
```

### Checkpoint Contents Schema

```yaml
checkpoint_contents:
  required_fields:
    checkpoint_id:
      type: "string"
      format: "UUID v4"
      description: "Unique identifier for this checkpoint"

    run_id:
      type: "string"
      format: "UUID v4"
      description: "Run this checkpoint belongs to"

    timestamp:
      type: "string"
      format: "ISO8601"
      description: "When checkpoint was created"

    trigger:
      type: "string"
      enum: ["phase_complete", "gate_decision", "circuit_breaker", "escalation", "iteration", "cost_threshold", "expensive_op", "destructive_op", "manual"]
      description: "What triggered this checkpoint"

    phase:
      type: "string"
      pattern: "phase_{number}_{name}"
      description: "Current phase at checkpoint time"

    sequence_number:
      type: "integer"
      description: "Checkpoint sequence within run (1, 2, 3...)"

  state_snapshot:
    description: "Complete state at checkpoint time"

    phase_outputs:
      type: "object"
      description: "Outputs from all completed phases"
      schema:
        phase_0: "phase_0_output or null"
        phase_0_5: "phase_0_5_output or null"
        phase_1: "phase_1_output or null"
        phase_2: "phase_2_output or null"
        phase_3: "phase_3_output or null"
        phase_4: "phase_4_output or null"
        phase_5: "phase_5_output or null"
        phase_6: "phase_6_output or null"
        phase_7: "phase_7_output or null"
        phase_8: "phase_8_output or null"

    current_phase_state:
      type: "object"
      description: "Partial state of in-progress phase"
      schema:
        phase:
          type: "string"
        started_at:
          type: "string"
        progress:
          type: "object"
          description: "Phase-specific progress indicators"
        partial_output:
          type: "object or null"

    gate_history:
      type: "array"
      description: "All gate decisions so far"
      items:
        gate_id:
          type: "string"
        decision:
          type: "string"
        score:
          type: "number"
        iteration:
          type: "integer"

    circuit_breaker_state:
      type: "object"
      description: "Current state of all circuit breakers"

    cost_accumulated:
      type: "number"
      description: "Total cost so far in USD"

    health_history:
      type: "array"
      description: "Health summaries from each phase"

    errors_encountered:
      type: "array"
      description: "All errors logged during run"

    active_lessons:
      type: "array"
      description: "Lessons active for this run"

  context_fields:
    description: "Additional context for debugging/resume"

    api_metrics:
      type: "object"
      description: "Per-API call counts and latencies"

    fallback_flags:
      type: "object"
      description: "Which APIs are using fallbacks"

    iteration_counts:
      type: "object"
      description: "Per-gate iteration counts"

    warnings:
      type: "array"
      description: "Non-fatal warnings accumulated"
```

### GitHub Rate Limit Handling (CRITICAL)

```yaml
rate_limit_handling:
  rationale: |
    CRITICAL: GitHub API has rate limits (5000/hour authenticated).
    Checkpoint writes must be rate-limit aware to avoid failures
    during critical moments (like circuit breaker trips).

  before_write:
    action: "Check X-RateLimit-Remaining header from last GitHub response"
    threshold: 100
    behavior: |
      if (remaining < 100) {
        // Low on rate limit budget
        log_warning("Rate limit low: " + remaining + " remaining");
        consider_batching = true;
      }

  on_403_rate_limit:
    detection: "HTTP 403 with 'rate limit' in response body"
    action: "Wait until X-RateLimit-Reset, then retry"
    implementation: |
      const resetTime = response.headers['X-RateLimit-Reset'];
      const waitMs = (resetTime * 1000) - Date.now() + 5000; // 5s buffer
      await sleep(Math.min(waitMs, 300000)); // Max 5 min wait
      retry();

  batching_strategy:
    description: "Reduce API calls by batching checkpoint data"

    approach: |
      Instead of writing each checkpoint as a separate file,
      batch multiple checkpoints into a single file when rate
      limit is low.

    batch_file_pattern: "/checkpoints/{run_id}/batch_{sequence_start}_{sequence_end}.json"

    batch_contents:
      checkpoints:
        type: "array"
        items: "checkpoint_contents"
      batch_metadata:
        first_sequence:
          type: "integer"
        last_sequence:
          type: "integer"
        checkpoint_count:
          type: "integer"

  rate_limit_tracking:
    storage: "Workflow variable"
    schema:
      last_remaining:
        type: "integer"
      last_reset:
        type: "integer"
      calls_this_run:
        type: "integer"

  emergency_fallback:
    condition: "Rate limit exhausted AND critical checkpoint needed"
    action: |
      1. Write to n8n execution data (temporary)
      2. Queue for GitHub write when rate limit resets
      3. Log warning about delayed persistence
      4. Continue workflow (don't halt for rate limit)
```

### Atomic Write Pattern

```yaml
atomic_write:
  description: "Ensure checkpoint writes are complete and valid"

  write_then_verify:
    step_1_write:
      action: "PUT checkpoint to GitHub"
      endpoint: "PUT /repos/{owner}/{repo}/contents/{path}"
      body:
        message: "Checkpoint {checkpoint_id}"
        content: "{{base64_encoded_checkpoint}}"
        sha: "{{existing_file_sha_if_update}}"

    step_2_verify:
      action: "GET checkpoint back from GitHub"
      endpoint: "GET /repos/{owner}/{repo}/contents/{path}"
      validation:
        - "Response status is 200"
        - "Content decodes successfully"
        - "checkpoint_id matches"
        - "Content hash matches"

    on_verification_failure:
      action: "Retry write (max 3 attempts)"
      escalation: "If all retries fail, log error and continue"
      note: "Don't halt workflow for checkpoint failure"

  conflict_handling:
    detection: "HTTP 409 Conflict on PUT"
    cause: "Another process updated the file"
    resolution: |
      1. GET current file (with SHA)
      2. Merge checkpoint data if possible
      3. Retry PUT with new SHA
      4. Max 3 attempts, then log error

  file_structure:
    individual_checkpoints: "/checkpoints/{run_id}/cp_{sequence}_{trigger}.json"
    latest_checkpoint: "/checkpoints/{run_id}/latest.json"
    checkpoint_index: "/checkpoints/{run_id}/index.json"

  index_schema:
    run_id:
      type: "string"
    checkpoint_count:
      type: "integer"
    checkpoints:
      type: "array"
      items:
        sequence:
          type: "integer"
        checkpoint_id:
          type: "string"
        trigger:
          type: "string"
        timestamp:
          type: "string"
        file_path:
          type: "string"
    last_updated:
      type: "string"
```

### Resume Capability

```yaml
resume_capability:
  description: "Enable workflow resumption from checkpoint"

  resume_trigger:
    manual: "Operator initiates resume with run_id"
    automatic: "Workflow detects incomplete run on startup"

  resume_process:
    step_1_load_checkpoint:
      action: "Fetch latest.json for run_id"
      fallback: "If missing, fetch index.json and load highest sequence"

    step_2_validate_checkpoint:
      checks:
        - "Checkpoint is parseable"
        - "Required fields present"
        - "Phase outputs are valid"
      on_invalid: "Log error, cannot resume, start fresh"

    step_3_restore_state:
      actions:
        - "Load phase_outputs into workflow"
        - "Restore circuit_breaker_state"
        - "Restore cost_accumulated"
        - "Restore active_lessons"
        - "Set current_phase to checkpoint phase"

    step_4_determine_resume_point:
      logic: |
        if (checkpoint.trigger === 'phase_complete') {
          // Resume at next phase
          resume_at = next_phase(checkpoint.phase);
        } else if (checkpoint.trigger === 'gate_decision') {
          // Resume at gate (may need re-evaluation)
          resume_at = checkpoint.phase + '_gate';
        } else if (checkpoint.trigger === 'circuit_breaker') {
          // Resume requires human decision
          resume_at = 'human_review';
        } else {
          // Resume at current phase start
          resume_at = checkpoint.phase;
        }

    step_5_notify:
      action: "Log resume event"
      message: |
        RESUME: Continuing run {run_id} from checkpoint {checkpoint_id}
        Resume point: {resume_at}
        Phases completed: {completed_phases}
        Cost so far: ${cost_accumulated}

  resume_limitations:
    external_state: |
      Checkpoints capture workflow state, not external state.
      If external systems changed (e.g., GitHub repo modified),
      resume may produce different results.

    time_sensitivity: |
      Some operations are time-sensitive (e.g., API rate limits reset).
      Resume after long delay may behave differently.

    lesson_changes: |
      If active lessons changed between checkpoint and resume,
      behavior may differ. Log warning if detected.
```

### Retention Policy

```yaml
retention_policy:
  description: "How long to keep checkpoints"

  rules:
    recent_runs:
      scope: "Most recent 10 runs"
      retention: "Keep all checkpoints"
      rationale: "Recent runs likely to need debugging"

    older_runs:
      scope: "Runs 11-50 most recent"
      retention: "Keep final checkpoint only"
      rationale: "Balance storage vs. audit needs"

    archive_runs:
      scope: "Runs older than 50 most recent"
      retention: "Keep run summary only (no checkpoints)"
      rationale: "Storage efficiency"

  cleanup_process:
    trigger: "After successful run completion"
    action: |
      1. Count total runs in /checkpoints/
      2. For runs 11-50: delete all except latest.json
      3. For runs 51+: delete all checkpoints, keep only summary
      4. Update retention log

  summary_schema:
    description: "Minimal record kept for archived runs"
    fields:
      run_id:
        type: "string"
      started_at:
        type: "string"
      completed_at:
        type: "string"
      phases_completed:
        type: "integer"
      final_status:
        type: "string"
      total_cost_usd:
        type: "number"
      errors_count:
        type: "integer"

  manual_override:
    description: "Keep specific runs indefinitely"
    mechanism: "Add run_id to /checkpoints/preserved_runs.json"
    use_case: "Runs with interesting failures or lessons"
```

### Checkpoint Storage Structure

```yaml
storage_structure:
  base_path: "/checkpoints"

  directory_layout:
    checkpoints:
      "{run_id}":
        "index.json": "Checkpoint index for this run"
        "latest.json": "Copy of most recent checkpoint (updated after each checkpoint write)"
        "cp_001_phase_complete.json": "First checkpoint"
        "cp_002_gate_decision.json": "Second checkpoint"
        "cp_003_circuit_breaker.json": "Third checkpoint"
        "...": "Additional checkpoints"
      "preserved_runs.json": "List of runs to keep indefinitely"
      "retention_log.json": "Log of cleanup actions"

  file_naming:
    pattern: "cp_{sequence:03d}_{trigger}.json"
    examples:
      - "cp_001_phase_complete.json"
      - "cp_002_gate_decision.json"
      - "cp_003_escalation.json"
      - "cp_004_circuit_breaker.json"

  size_management:
    max_checkpoint_size_mb: 10
    on_oversized: |
      1. Split large arrays (e.g., phase_outputs) into separate files
      2. Store references in main checkpoint
      3. Log warning about large checkpoint
```

## n8n Implementation Pattern

```yaml
n8n_implementation:
  checkpoint_write_workflow:
    description: "Reusable sub-workflow for checkpoint writes"

    inputs:
      - trigger: "What triggered this checkpoint"
      - state: "Current workflow state"
      - run_id: "Run identifier"

    nodes:
      1_check_rate_limit:
        type: "Code"
        purpose: "Check if we have rate limit budget"
        code_pattern: |
          const remaining = $input.rate_limit_remaining || 5000;
          const shouldBatch = remaining < 100;
          return { should_batch: shouldBatch, remaining };

      2_prepare_checkpoint:
        type: "Code"
        purpose: "Build checkpoint contents"
        code_pattern: |
          const checkpoint = {
            checkpoint_id: generateUUID(),
            run_id: $input.run_id,
            timestamp: new Date().toISOString(),
            trigger: $input.trigger,
            phase: $input.current_phase,
            sequence_number: $input.checkpoint_count + 1,
            state_snapshot: {
              phase_outputs: $input.phase_outputs,
              current_phase_state: $input.current_phase_state,
              gate_history: $input.gate_history,
              circuit_breaker_state: $input.circuit_breaker_state,
              cost_accumulated: $input.cost_accumulated,
              health_history: $input.health_history,
              errors_encountered: $input.errors_encountered,
              active_lessons: $input.active_lessons
            },
            context_fields: {
              api_metrics: $input.api_metrics,
              fallback_flags: $input.fallback_flags,
              iteration_counts: $input.iteration_counts,
              warnings: $input.warnings
            }
          };
          return { checkpoint };

      3_write_to_github:
        type: "HTTP Request"
        method: "PUT"
        url: "https://api.github.com/repos/{owner}/{repo}/contents/checkpoints/{run_id}/cp_{sequence}_{trigger}.json"
        headers:
          Authorization: "Bearer {{$env.GITHUB_TOKEN}}"
          Accept: "application/vnd.github.v3+json"
        body:
          message: "Checkpoint {{checkpoint_id}}"
          content: "{{base64_encode(JSON.stringify(checkpoint))}}"
          sha: "{{existing_sha_if_update}}"
        continueOnFail: true

      4_handle_rate_limit:
        type: "IF"
        condition: "{{ $json.status === 403 && $json.body.includes('rate limit') }}"
        true_branch: "Wait and retry node"
        false_branch: "Continue to verify"

      5_wait_for_reset:
        type: "Code"
        purpose: "Wait until rate limit resets"
        code_pattern: |
          const resetTime = $input.headers['X-RateLimit-Reset'];
          const waitMs = Math.min((resetTime * 1000) - Date.now() + 5000, 300000);
          await new Promise(resolve => setTimeout(resolve, waitMs));
          return { waited: true };

      6_verify_write:
        type: "HTTP Request"
        method: "GET"
        url: "https://api.github.com/repos/{owner}/{repo}/contents/checkpoints/{run_id}/cp_{sequence}_{trigger}.json"
        purpose: "Verify checkpoint was written correctly"

      7_update_latest:
        type: "HTTP Request"
        method: "PUT"
        url: "https://api.github.com/repos/{owner}/{repo}/contents/checkpoints/{run_id}/latest.json"
        purpose: "Update latest.json pointer"

      8_update_index:
        type: "Code + HTTP Request"
        purpose: "Add checkpoint to index"

  resume_workflow:
    description: "Workflow for resuming from checkpoint"

    nodes:
      1_fetch_latest:
        type: "HTTP Request"
        method: "GET"
        url: "https://api.github.com/repos/{owner}/{repo}/contents/checkpoints/{run_id}/latest.json"

      2_parse_checkpoint:
        type: "Code"
        purpose: "Decode and validate checkpoint"

      3_restore_state:
        type: "Code"
        purpose: "Populate workflow variables from checkpoint"

      4_determine_resume_point:
        type: "Code"
        purpose: "Calculate where to resume"

      5_route_to_phase:
        type: "Switch"
        purpose: "Jump to appropriate phase"

  rate_limit_tracking:
    description: "Track rate limit across workflow"

    after_each_github_call:
      type: "Code"
      code_pattern: |
        const remaining = $input.response.headers['X-RateLimit-Remaining'];
        const reset = $input.response.headers['X-RateLimit-Reset'];
        $json.rate_limit = { remaining, reset };
        return $json;
```

## Cross-References

| Related Spec | Relationship | Key Integration Points |
|--------------|--------------|------------------------|
| [phase-io-contracts.md](../io-contracts/phase-io-contracts.md) | depends on | Phase output schemas define checkpoint contents |
| [error-handling-v2.1.md](../reliability/error-handling-v2.1.md) | related | Errors are included in checkpoint state |
| [circuit-breakers-v2.1.md](../reliability/circuit-breakers-v2.1.md) | related | Breaker state is checkpointed; trips trigger checkpoints |
| [gate-logic-v2.1.md](../gates/gate-logic-v2.1.md) | related | Gate decisions trigger checkpoints |
| [health-monitoring-v2.1.md](../reliability/health-monitoring-v2.1.md) | related | Health history included in checkpoints |

## Validation Criteria

1. **Write Test:** Create checkpoint, verify it appears in GitHub
2. **Verify Test:** Write checkpoint, verify content matches after read-back
3. **Rate Limit Test:** Simulate low rate limit, verify batching activates
4. **Resume Test:** Create checkpoint, simulate failure, verify resume works
5. **Retention Test:** Create 60 runs, verify cleanup follows retention policy
6. **Conflict Test:** Simulate concurrent writes, verify conflict resolution

## Changelog

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 2.1.1 | 2025-12-05 | Devin | Initial creation with GitHub rate limit awareness |
