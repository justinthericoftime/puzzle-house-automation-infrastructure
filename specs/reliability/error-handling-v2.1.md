# Error Handling Specification

## Metadata
- **Version:** 2.1.1
- **Status:** Draft
- **Last Updated:** 2025-12-05
- **Dependencies:** retry-logic-v2.1.md
- **Dependents:** circuit-breakers-v2.1.md, health-monitoring-v2.1.md
- **Owner:** Infrastructure Team
- **Stress Test Findings Addressed:** FINDING-001 (parallel sinks infeasible), FINDING-002 (independent heartbeat thread infeasible)

## Overview

This specification defines the error handling strategy for the Composable Development Infrastructure. It establishes error classification, sequential error sink writes, halt conditions, and integration with external heartbeat services. The design explicitly addresses n8n platform constraints discovered during stress testing.

## Platform Constraints

- **n8n Limitation:** Cannot execute parallel operations within a single workflow execution
- **n8n Limitation:** Cannot spawn independent threads for background tasks
- **GitHub API:** Rate limits apply to all write operations (5000/hour authenticated)
- **LLM APIs:** Various timeout and rate limit behaviors per provider

## Scope

### In Scope
- Error classification schema (transient, persistent, unknown)
- API-specific error handling overrides
- Sequential error sink configuration
- Halt condition logic
- External heartbeat integration
- Error entry JSON schema
- Escalation rules

### Out of Scope
- Retry logic (see retry-logic-v2.1.md)
- Circuit breaker state management (see circuit-breakers-v2.1.md)
- Health monitoring thresholds (see health-monitoring-v2.1.md)

## Specification

### Error Classification

```yaml
error_classification:
  transient:
    description: "Temporary failures that may resolve on retry"
    http_codes: [429, 500, 502, 503, 504]
    default_action: "retry_with_backoff"
    examples:
      - "Rate limit exceeded"
      - "Service temporarily unavailable"
      - "Gateway timeout"

  persistent:
    description: "Failures that will not resolve on retry"
    http_codes: [400, 401, 403, 404, 422]
    default_action: "log_and_escalate"
    examples:
      - "Invalid request format"
      - "Authentication failed"
      - "Resource not found"
      - "Validation error"

  unknown:
    description: "Unclassified errors requiring investigation"
    http_codes: []
    default_action: "single_retry_then_escalate"
    examples:
      - "Unexpected response format"
      - "Network errors without HTTP code"
      - "Timeout without response"
```

### API-Specific Overrides

```yaml
api_overrides:
  github:
    rate_limit_429:
      action: "wait_for_reset"
      behavior: |
        Read X-RateLimit-Reset header
        Wait until reset time + 5 second buffer
        Then retry
      max_wait_seconds: 300

    secondary_rate_limit:
      detection: "403 with 'secondary rate limit' in message"
      action: "exponential_backoff"
      initial_wait_seconds: 60

  perplexity:
    timeout_handling:
      timeout_ms: 180000
      on_timeout: "classify_as_transient"
      note: "Long research queries may take 2-3 minutes"

  claude:
    overloaded_error:
      detection: "529 or 'overloaded' in response"
      action: "retry_with_longer_backoff"
      base_delay_ms: 5000

  devin:
    build_timeout:
      timeout_ms: 300000
      on_timeout: "checkpoint_and_escalate"
      note: "Build operations are expensive; don't retry blindly"

  chatprd:
    fallback_behavior:
      on_any_error: "switch_to_claude_fallback"
      note: "ChatPRD is optional; Claude can generate specs"
```

### Sequential Error Sinks (CRITICAL)

```yaml
error_sinks:
  architecture: "sequential"
  rationale: |
    FINDING-001: n8n cannot execute parallel operations.
    Error sinks MUST be written sequentially, not in parallel.
    Each sink is attempted only if the previous sink fails.

  execution_order: [primary, secondary, tertiary]

  primary:
    name: "GitHub Error Log"
    type: "github_file"
    path: "/logs/{run_id}/errors/{timestamp}_{error_id}.json"
    timeout_ms: 10000
    on_success: "stop_sink_chain"
    on_failure: "continue_to_secondary"

  secondary:
    name: "n8n Internal Storage"
    type: "n8n_execution_data"
    storage: "$execution.customData.errors[]"
    timeout_ms: 1000
    on_success: "stop_sink_chain"
    on_failure: "continue_to_tertiary"

  tertiary:
    name: "External Webhook"
    type: "http_webhook"
    url: "{{$env.ERROR_WEBHOOK_URL}}"
    timeout_ms: 5000
    on_success: "stop_sink_chain"
    on_failure: "all_sinks_failed"

  all_sinks_failed:
    action: "halt_workflow"
    notification: "immediate_escalation"
    message: |
      CRITICAL: All error sinks failed. Workflow halted.
      Run: {run_id}
      Phase: {phase}
      Original error: {error_message}
      Sink failures: {sink_failure_details}
```

### Halt Condition Logic

```yaml
halt_conditions:
  immediate_halt:
    triggers:
      - "all_error_sinks_failed"
      - "authentication_failure_on_critical_api"
      - "cost_threshold_exceeded"
      - "meta_breaker_tripped"
    action: |
      1. Attempt final checkpoint (best effort)
      2. Send immediate escalation notification
      3. Stop workflow with error state
      4. Log halt reason to all available sinks

  graceful_halt:
    triggers:
      - "persistent_error_on_non_critical_path"
      - "optional_api_unavailable"
    action: |
      1. Log error
      2. Set fallback flag
      3. Continue with degraded functionality
      4. Include in phase health summary
```

### External Heartbeat Integration

```yaml
external_heartbeat:
  architecture: "external_service"
  rationale: |
    FINDING-002: n8n cannot run independent background threads.
    Heartbeat monitoring MUST use an external service that
    detects missing pings, not an n8n-internal watchdog.

  recommended_services:
    - name: "Healthchecks.io"
      free_tier: "20 checks"
      setup_url: "https://healthchecks.io"
    - name: "Dead Man's Snitch"
      free_tier: "1 snitch"
    - name: "Cronitor"
      free_tier: "5 monitors"

  integration_pattern:
    workflow_responsibility: "Send pings at defined points"
    external_responsibility: "Detect missing pings and alert"

  ping_points:
    - event: "workflow_start"
      endpoint: "{{HEARTBEAT_URL}}"
    - event: "phase_complete"
      endpoint: "{{HEARTBEAT_URL}}?phase={phase}"
    - event: "workflow_complete"
      endpoint: "{{HEARTBEAT_URL}}"
    - event: "error_occurred"
      endpoint: "{{HEARTBEAT_URL}}/fail"

  error_sink_integration:
    on_all_sinks_failed: "ping_heartbeat_fail_endpoint"
    purpose: "External service alerts operator even if workflow is stuck"
```

### Error Entry Schema

```yaml
error_entry_schema:
  required_fields:
    error_id:
      type: "string"
      format: "UUID v4"
      description: "Unique identifier for this error instance"

    run_id:
      type: "string"
      format: "UUID v4"
      description: "Run this error occurred in"

    timestamp:
      type: "string"
      format: "ISO8601"
      description: "When the error occurred"

    phase:
      type: "string"
      pattern: "phase_{number}_{name}"
      description: "Which phase was executing"

    node:
      type: "string"
      description: "n8n node that encountered the error"

    api:
      type: "string"
      enum: ["claude", "perplexity", "chatprd", "devin", "github", "internal"]
      description: "Which API or service failed"

    classification:
      type: "string"
      enum: ["transient", "persistent", "unknown"]
      description: "Error classification"

    http_status:
      type: "integer or null"
      description: "HTTP status code if applicable"

    error_message:
      type: "string"
      description: "Error message (sanitized, no PII)"
      pii_note: "Ensure no PII in this field"

    stack_trace:
      type: "string or null"
      description: "If available and relevant"

    retry_count:
      type: "integer"
      description: "0 if first attempt, increments with retries"

  resolution_fields:
    action_taken:
      type: "string"
      enum: ["retry_scheduled", "escalated", "halted", "logged_only"]

    sink_results:
      type: "object"
      description: "Which sinks succeeded/failed"
      example:
        github: "success"
        n8n_internal: "success"
        webhook: "failed_timeout"
```

### Escalation Rules

```yaml
escalation_rules:
  immediate_escalation:
    triggers:
      - "persistent error on critical path"
      - "all error sinks failed (workflow halted)"
      - "authentication_failure (credentials may be revoked)"
      - "cost threshold exceeded"
    channel: "webhook (high priority)"
    message_template: |
      🚨 IMMEDIATE: {error_type} in {phase}
      Run: {run_id}
      Error: {error_message}
      Action required: {recommended_action}

  deferred_escalation:
    triggers:
      - "transient error exceeded retry limit"
      - "unknown error after single retry"
    channel: "error log only (reviewed in morning batch)"
    note: "These are logged but don't wake up the operator"

  silent_logging:
    triggers:
      - "transient error resolved by retry"
      - "warning-level issues"
    channel: "error log only"
    note: "No notification; available for post-mortem analysis"
```

## n8n Implementation Pattern

```yaml
n8n_implementation:
  error_handler_workflow:
    description: "Create a reusable sub-workflow for error handling"

    inputs:
      - error_object: "The caught error"
      - context: "run_id, phase, node, api"

    nodes:
      1_classify_error:
        type: "Code"
        purpose: "Classify error as transient/persistent/unknown"
        code_pattern: |
          const httpCode = $input.error.httpCode;
          const transientCodes = [429, 500, 502, 503, 504];
          const persistentCodes = [400, 401, 403, 404, 422];

          if (transientCodes.includes(httpCode)) {
            return { classification: 'transient' };
          } else if (persistentCodes.includes(httpCode)) {
            return { classification: 'persistent' };
          } else {
            return { classification: 'unknown' };
          }

      2_write_github:
        type: "HTTP Request"
        purpose: "POST to GitHub API to create error log file"
        on_error: "Continue (don't fail workflow)"
        settings:
          continueOnFail: true

      3_check_github_result:
        type: "IF"
        condition: "{{ $json.success == true }}"
        true_branch: "Return success"
        false_branch: "Continue to n8n internal"

      4_write_n8n_internal:
        type: "Code"
        purpose: "Append to $execution.customData.errors"
        code_pattern: |
          const errors = $execution.customData.errors || [];
          errors.push($input.errorEntry);
          $execution.customData.errors = errors;
          return { success: true };

      5_check_n8n_result:
        type: "IF"
        condition: "{{ $json.success == true }}"
        true_branch: "Return success"
        false_branch: "Continue to webhook"

      6_write_webhook:
        type: "HTTP Request"
        purpose: "POST to error webhook"
        on_error: "Continue"
        settings:
          continueOnFail: true

      7_evaluate_halt:
        type: "IF"
        condition: "{{ $json.successful_sinks == 0 }}"
        true_branch: "Stop and Error node"
        false_branch: "Return to main workflow"

    usage: |
      In main workflow, wrap API calls in try/catch:
      - On error: Call error handler sub-workflow
      - Pass error + context
      - Error handler returns: {should_retry, should_halt, error_logged}
```

## Cross-References

| Related Spec | Relationship | Key Integration Points |
|--------------|--------------|------------------------|
| [retry-logic-v2.1.md](./retry-logic-v2.1.md) | depends on | Retry behavior for transient errors, per-API timeout values |
| [circuit-breakers-v2.1.md](./circuit-breakers-v2.1.md) | depended by | Error counts feed circuit breaker trip conditions |
| [health-monitoring-v2.1.md](./health-monitoring-v2.1.md) | depended by | Error rates contribute to health status |
| [gate-logic-v2.1.md](../gates/gate-logic-v2.1.md) | depended by | Escalation rules align with gate escalation |

## Validation Criteria

1. **Error Classification Test:** Submit errors with various HTTP codes and verify correct classification
2. **Sequential Sink Test:** Simulate primary sink failure and verify secondary is attempted
3. **Halt Condition Test:** Simulate all sinks failing and verify workflow halts
4. **Heartbeat Integration Test:** Verify heartbeat fail endpoint is called on critical errors
5. **Schema Validation Test:** Verify all error entries conform to the defined schema

## Changelog

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 2.1.1 | 2025-12-05 | Devin | Initial creation addressing FINDING-001 (sequential sinks) and FINDING-002 (external heartbeat) |
