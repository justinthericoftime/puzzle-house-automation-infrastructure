# Health Monitoring Specification

## Metadata
- **Version:** 2.1.1
- **Status:** Draft
- **Last Updated:** 2025-12-05
- **Dependencies:** error-handling-v2.1.md, retry-logic-v2.1.md
- **Dependents:** gate-logic-v2.1.md
- **Owner:** Infrastructure Team
- **Stress Test Findings Addressed:** FINDING-002 (independent heartbeat thread infeasible in n8n)

## Overview

This specification defines the health monitoring strategy for the Composable Development Infrastructure. It covers pre-flight health checks, continuous lightweight monitoring, external heartbeat configuration, and mid-run health summaries. The design explicitly addresses the n8n constraint that independent background threads cannot be spawned.

## Platform Constraints

- **n8n Limitation:** Cannot spawn independent threads for background monitoring
- **n8n Limitation:** No built-in health check primitives
- **External Dependency:** Heartbeat monitoring requires external service (Healthchecks.io, etc.)
- **API Variability:** Different APIs have different health check requirements

## Scope

### In Scope
- Pre-flight health checks (required vs optional APIs)
- Continuous monitoring (lightweight, per-call tracking)
- External heartbeat configuration
- Threshold definitions (degraded, critical)
- Mid-run health summary schema
- n8n implementation patterns

### Out of Scope
- Error handling logic (see error-handling-v2.1.md)
- Retry logic (see retry-logic-v2.1.md)
- Circuit breaker triggers (see circuit-breakers-v2.1.md)

## Specification

### Pre-Flight Health Checks

```yaml
pre_flight_checks:
  timing: "Execute once at Phase 0, before any real work"
  purpose: "Verify all required services are available before starting expensive operations"

  checks:
    - api: "claude"
      test_type: "simple_completion"
      test_prompt: "Reply with exactly: OK"
      expected: "Response contains 'OK'"
      timeout_ms: 10000
      required: true
      failure_impact: "Cannot proceed - Claude is primary LLM"

    - api: "perplexity"
      test_type: "simple_search"
      test_query: "test query health check"
      expected: "Any valid response"
      timeout_ms: 15000
      required: true
      failure_impact: "Cannot proceed - Perplexity is primary research tool"

    - api: "github"
      test_type: "file_read"
      test_path: "/knowledge/meta-prompts/templates/meta-prompt-template-v1.0.md"
      expected: "File content returned"
      timeout_ms: 5000
      required: true
      failure_impact: "Cannot proceed - GitHub is state storage"

    - api: "chatprd"
      test_type: "ping"
      expected: "200 OK or valid error response"
      timeout_ms: 10000
      required: false
      fallback_note: "Will use Claude for spec generation"
      fallback_flag: "fallback_required.chatprd = true"

    - api: "devin"
      test_type: "ping"
      expected: "200 OK or valid error response"
      timeout_ms: 10000
      required: false
      fallback_note: "Only needed for Phase 7"
      fallback_flag: "fallback_required.devin = true"

    - api: "heartbeat_service"
      test_type: "ping"
      test_url: "{{$env.HEARTBEAT_URL}}/ping"
      expected: "200 OK"
      timeout_ms: 5000
      required: false
      note: "External heartbeat service (Healthchecks.io etc)"
      fallback_note: "Workflow will run without external monitoring"

  failure_handling:
    required_api_fails:
      action: "abort_workflow"
      notification: "immediate"
      message: "PRE-FLIGHT FAILED: Required API '{api}' is unavailable. Error: {error}"
      log_to: "error_sinks"

    optional_api_fails:
      action: "log_warning_continue"
      notification: "none"
      message: "PRE-FLIGHT WARNING: Optional API '{api}' unavailable. Will use fallback."
      set_flag: "fallback_required.{api} = true"

  output_schema:
    pre_flight_complete: "boolean"
    timestamp: "ISO8601"
    checks_passed: "integer"
    checks_failed: "integer"
    required_failures: "array"
    optional_failures: "array"
    fallback_flags: "object"
```

### Continuous Monitoring (Lightweight)

```yaml
continuous_monitoring:
  approach: "Lightweight tracking without separate threads"
  rationale: |
    FINDING-002: n8n cannot spawn independent monitoring threads.
    Instead, we track health metrics inline with each API call.
    This adds minimal overhead while providing visibility.

  per_call_tracking:
    what_to_track:
      - api: "Which API was called"
      - timestamp: "When the call was made"
      - latency_ms: "How long it took"
      - success: "true/false"
      - error_type: "If failed, what type (transient/persistent/unknown)"
      - http_status: "HTTP status code if applicable"

    storage: "Workflow variable: $json.api_metrics[api].calls[]"
    max_calls_stored: 10
    storage_note: "Rolling window per API; oldest calls dropped when limit reached"

  threshold_evaluation:
    when: "After each API call, before proceeding"
    purpose: "Detect degradation early and take action"

    thresholds:
      degraded:
        condition: "Last 5 calls: success_rate < 60%"
        action: "Log warning, set degraded flag"
        flag: "api_status.{api} = 'degraded'"
        notification: "none (logged only)"

      critical:
        condition: "Last 5 calls: success_rate < 20%"
        action: "Switch to fallback OR pause phase"
        flag: "api_status.{api} = 'critical'"
        notification: "deferred escalation"

    evaluation_logic: |
      const calls = $json.api_metrics[api].calls.slice(-5);
      const successCount = calls.filter(c => c.success).length;
      const successRate = successCount / calls.length;

      if (successRate < 0.20) return 'critical';
      if (successRate < 0.60) return 'degraded';
      return 'healthy';

  baseline_latency:
    establishment: "Average of first 3 successful calls per API"
    storage: "$json.api_metrics[api].baseline_latency_ms"
    comparison: "If current_latency > 2 * baseline: log warning"
    purpose: "Detect performance degradation even when calls succeed"
```

### External Heartbeat Configuration

```yaml
external_heartbeat:
  architecture: "external_service"
  rationale: |
    FINDING-002: n8n cannot run independent background threads.
    Heartbeat monitoring MUST use an external service that
    detects missing pings, not an n8n-internal watchdog.

  recommended_service: "Healthchecks.io (free tier: 20 checks)"

  alternative_services:
    - name: "Dead Man's Snitch"
      free_tier: "1 snitch"
      url: "https://deadmanssnitch.com"
    - name: "Cronitor"
      free_tier: "5 monitors"
      url: "https://cronitor.io"
    - name: "Better Uptime"
      free_tier: "10 monitors"
      url: "https://betteruptime.com"

  setup_steps:
    step_1: "Create account at healthchecks.io"
    step_2: "Create new check with period=60min, grace=30min"
    step_3: "Copy ping URL (https://hc-ping.com/your-uuid)"
    step_4: "Store in n8n credentials as HEARTBEAT_URL"
    step_5: "Configure alert channels (email, Slack, SMS)"

  workflow_integration:
    ping_points:
      - event: "phase_0_start"
        purpose: "Signal workflow beginning"
        endpoint: "{{HEARTBEAT_URL}}"

      - event: "phase_0_5_complete"
        purpose: "Signal hardening complete"
        endpoint: "{{HEARTBEAT_URL}}?phase=0.5"

      - event: "each_phase_complete"
        purpose: "Signal progress"
        endpoint: "{{HEARTBEAT_URL}}?rid={{run_id}}&phase={{phase}}"

      - event: "workflow_complete"
        purpose: "Signal successful completion"
        endpoint: "{{HEARTBEAT_URL}}"

      - event: "on_error"
        purpose: "Signal error occurred"
        endpoint: "{{HEARTBEAT_URL}}/fail"

    ping_format:
      success: "GET {{HEARTBEAT_URL}}"
      with_context: "GET {{HEARTBEAT_URL}}?rid={{run_id}}&phase={{phase}}"
      failure: "GET {{HEARTBEAT_URL}}/fail"

  detection_behavior:
    normal: "Pings received within grace period"
    alert: "No ping received for grace_period (30 min)"
    alert_message: "Workflow may be stuck or crashed"

  error_handling:
    on_ping_failure: "Continue workflow (heartbeat failure shouldn't halt work)"
    logging: "Log ping failures to error log for debugging"
```

### Mid-Run Health Summary

```yaml
mid_run_health_summary:
  trigger: "At each phase boundary (after phase completion, before next phase)"
  storage: "Include in phase output metadata AND checkpoint"
  purpose: "Provide visibility into system health for debugging and monitoring"

  summary_schema:
    run_id:
      type: "string"
      format: "UUID"

    phase_completed:
      type: "string"
      pattern: "phase_{number}_{name}"

    timestamp:
      type: "string"
      format: "ISO8601"

    api_health:
      type: "array"
      items:
        api:
          type: "string"
          enum: ["claude", "perplexity", "chatprd", "devin", "github"]
        calls_made:
          type: "integer"
        successful:
          type: "integer"
        failed:
          type: "integer"
        success_rate:
          type: "number"
          format: "0.0 to 1.0"
        avg_latency_ms:
          type: "integer"
        last_status:
          type: "string"
          enum: ["healthy", "degraded", "critical"]
        fallback_active:
          type: "boolean"

    overall_status:
      healthy_apis:
        type: "integer"
      degraded_apis:
        type: "integer"
      critical_apis:
        type: "integer"

    recommendation:
      value:
        type: "string"
        enum: ["proceed", "proceed_with_caution", "pause_for_review"]
      reason:
        type: "string"

  recommendation_logic: |
    if (critical_apis > 0) return { value: 'pause_for_review', reason: 'Critical API failure' };
    if (degraded_apis > 1) return { value: 'proceed_with_caution', reason: 'Multiple degraded APIs' };
    return { value: 'proceed', reason: 'All systems healthy' };
```

### Health Status Aggregation

```yaml
health_aggregation:
  purpose: "Combine individual API health into overall system health"

  aggregation_rules:
    system_healthy:
      condition: "All required APIs healthy, no critical flags"
      action: "Continue normal operation"

    system_degraded:
      condition: "Any API degraded OR any fallback active"
      action: "Continue with increased monitoring"
      notification: "Log warning"

    system_critical:
      condition: "Any required API critical OR multiple APIs degraded"
      action: "Pause for review OR switch to safe mode"
      notification: "Deferred escalation"

  safe_mode:
    description: "Reduced functionality mode when system is degraded"
    behaviors:
      - "Skip optional API calls"
      - "Use cached data where possible"
      - "Increase checkpoint frequency"
      - "Reduce batch sizes"
```

## n8n Implementation Pattern

```yaml
n8n_implementation:
  pre_flight_workflow:
    description: "Execute health checks at workflow start"

    nodes:
      1_check_claude:
        type: "HTTP Request"
        method: "POST"
        url: "{{$env.CLAUDE_API_URL}}"
        body: { prompt: "Reply with exactly: OK" }
        timeout: 10000
        continueOnFail: true

      2_check_perplexity:
        type: "HTTP Request"
        method: "POST"
        url: "{{$env.PERPLEXITY_API_URL}}"
        body: { query: "test query health check" }
        timeout: 15000
        continueOnFail: true

      3_check_github:
        type: "HTTP Request"
        method: "GET"
        url: "https://api.github.com/repos/{owner}/{repo}/contents/knowledge/meta-prompts/templates/meta-prompt-template-v1.0.md"
        timeout: 5000
        continueOnFail: true

      4_check_optional_apis:
        type: "HTTP Request"
        description: "Check ChatPRD and Devin (optional)"
        continueOnFail: true

      5_aggregate_results:
        type: "Code"
        purpose: "Combine results, set fallback flags"
        code_pattern: |
          const results = {
            claude: $('1_check_claude').json.success,
            perplexity: $('2_check_perplexity').json.success,
            github: $('3_check_github').json.success,
            chatprd: $('4_check_optional_apis').json.chatprd_success,
            devin: $('4_check_optional_apis').json.devin_success
          };

          const requiredFailed = ['claude', 'perplexity', 'github']
            .filter(api => !results[api]);

          return {
            pre_flight_complete: requiredFailed.length === 0,
            required_failures: requiredFailed,
            fallback_flags: {
              chatprd: !results.chatprd,
              devin: !results.devin
            }
          };

      6_evaluate_proceed:
        type: "IF"
        condition: "{{ $json.pre_flight_complete == true }}"
        true_branch: "Continue to Phase 0.5"
        false_branch: "Stop and Error"

  per_call_tracking:
    description: "Wrap each API call with metrics tracking"

    pattern: |
      Before API call:
        const startTime = Date.now();

      After API call:
        const latency = Date.now() - startTime;
        const metrics = $json.api_metrics || {};
        const apiMetrics = metrics[api] || { calls: [], baseline_latency_ms: null };

        apiMetrics.calls.push({
          timestamp: new Date().toISOString(),
          latency_ms: latency,
          success: response.success,
          error_type: response.error?.type || null,
          http_status: response.status || null
        });

        if (apiMetrics.calls.length > 10) {
          apiMetrics.calls = apiMetrics.calls.slice(-10);
        }

        metrics[api] = apiMetrics;
        $json.api_metrics = metrics;

  heartbeat_ping:
    node_type: "HTTP Request"
    method: "GET"
    url: "{{$env.HEARTBEAT_URL}}?rid={{$json.run_id}}&phase={{$json.phase}}"
    continueOnFail: true
    placement: "After each phase completion node"

  health_summary_generation:
    node_type: "Code"
    trigger: "At phase boundary"
    code_pattern: |
      const metrics = $json.api_metrics || {};
      const apiHealth = Object.entries(metrics).map(([api, data]) => {
        const calls = data.calls || [];
        const successful = calls.filter(c => c.success).length;
        const successRate = calls.length > 0 ? successful / calls.length : 1;
        const avgLatency = calls.length > 0
          ? Math.round(calls.reduce((sum, c) => sum + c.latency_ms, 0) / calls.length)
          : 0;

        let status = 'healthy';
        if (successRate < 0.20) status = 'critical';
        else if (successRate < 0.60) status = 'degraded';

        return {
          api,
          calls_made: calls.length,
          successful,
          failed: calls.length - successful,
          success_rate: successRate,
          avg_latency_ms: avgLatency,
          last_status: status,
          fallback_active: $json.fallback_flags?.[api] || false
        };
      });

      const healthy = apiHealth.filter(a => a.last_status === 'healthy').length;
      const degraded = apiHealth.filter(a => a.last_status === 'degraded').length;
      const critical = apiHealth.filter(a => a.last_status === 'critical').length;

      let recommendation = { value: 'proceed', reason: 'All systems healthy' };
      if (critical > 0) recommendation = { value: 'pause_for_review', reason: 'Critical API failure' };
      else if (degraded > 1) recommendation = { value: 'proceed_with_caution', reason: 'Multiple degraded APIs' };

      return {
        run_id: $json.run_id,
        phase_completed: $json.phase,
        timestamp: new Date().toISOString(),
        api_health: apiHealth,
        overall_status: { healthy_apis: healthy, degraded_apis: degraded, critical_apis: critical },
        recommendation
      };
```

## Cross-References

| Related Spec | Relationship | Key Integration Points |
|--------------|--------------|------------------------|
| [error-handling-v2.1.md](./error-handling-v2.1.md) | depends on | Health status feeds error classification |
| [retry-logic-v2.1.md](./retry-logic-v2.1.md) | depends on | Per-API timeout values for health checks |
| [circuit-breakers-v2.1.md](./circuit-breakers-v2.1.md) | related | API health informs CB-3 (consecutive failures) |
| [gate-logic-v2.1.md](../gates/gate-logic-v2.1.md) | depended by | Health summary included in gate context |
| [phase-io-contracts.md](../io-contracts/phase-io-contracts.md) | depended by | Health summary schema used in phase outputs |

## Validation Criteria

1. **Pre-Flight Test:** Disable a required API, verify workflow aborts with correct message
2. **Fallback Test:** Disable optional API, verify fallback flag is set and workflow continues
3. **Threshold Test:** Simulate 3/5 failed calls, verify degraded status is detected
4. **Heartbeat Test:** Verify pings are sent at each phase boundary
5. **Health Summary Test:** Verify summary schema is complete and accurate at phase boundaries

## Changelog

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 2.1.1 | 2025-12-05 | Devin | Initial creation addressing FINDING-002 (external heartbeat architecture) |
