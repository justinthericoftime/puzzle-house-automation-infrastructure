# Gate Logic Specification

## Metadata
- **Version:** 2.1.1
- **Status:** Draft
- **Last Updated:** 2025-12-05
- **Dependencies:** health-monitoring-v2.1.md, circuit-breakers-v2.1.md
- **Dependents:** phase-io-contracts.md, checkpoint-spec.md
- **Owner:** Infrastructure Team
- **Stress Test Findings Addressed:** FINDING-005 (LLM self-critique unreliable for escalation decisions)

## Overview

This specification defines the gate logic system for the Composable Development Infrastructure. Gates are quality checkpoints between phases that evaluate outputs and determine whether to proceed, iterate, or escalate. The design explicitly addresses the finding that LLM self-critique is unreliable for making autonomous escalation decisions.

## Platform Constraints

- **LLM Limitation:** Self-critique scores are not reliable enough for autonomous proceed/escalate decisions
- **LLM Limitation:** Adversarial pass findings should inform humans, not trigger automatic escalation
- **n8n Limitation:** Complex decision trees must be implemented as IF/Switch nodes
- **Human Availability:** Operators may not be available 24/7; time-aware escalation needed

## Scope

### In Scope
- Gate classification (critical vs standard)
- Uncertainty assessment schema
- Escalation decision rules
- Adversarial pass (informational only)
- Time-aware escalation
- Gate decision logging
- n8n implementation patterns

### Out of Scope
- Health monitoring thresholds (see health-monitoring-v2.1.md)
- Circuit breaker triggers (see circuit-breakers-v2.1.md)
- Phase I/O schemas (see phase-io-contracts.md)

## Specification

### Gate Classification

```yaml
gate_classification:
  critical_gates:
    gates: [gate_2, gate_4, gate_6]
    description: "High-impact decisions that affect architecture or significant resources"
    automation_level: "human_required"
    rationale: |
      FINDING-005: LLM self-critique is unreliable for escalation decisions.
      Critical gates ALWAYS require human review, regardless of scores.
      This is a hard constraint, not a threshold-based decision.

    gate_2:
      name: "Architecture Validation"
      phase_before: "Phase 1 (Research)"
      phase_after: "Phase 2 (Specification)"
      impact: "Architectural decisions are expensive to reverse"
      human_review: "ALWAYS REQUIRED"

    gate_4:
      name: "Specification Approval"
      phase_before: "Phase 3 (Specification)"
      phase_after: "Phase 4 (Implementation Planning)"
      impact: "Specifications drive all downstream implementation"
      human_review: "ALWAYS REQUIRED"

    gate_6:
      name: "Pre-Build Validation"
      phase_before: "Phase 5 (Implementation)"
      phase_after: "Phase 6 (Build)"
      impact: "Build phase consumes significant resources (Devin time)"
      human_review: "ALWAYS REQUIRED"

  standard_gates:
    gates: [gate_1, gate_3, gate_5, gate_7, gate_8]
    description: "Lower-impact checkpoints that can proceed with scoring"
    automation_level: "score_based_with_escalation"
    rationale: "These gates can auto-proceed if scores meet threshold, but escalate on uncertainty"

    thresholds:
      auto_proceed: "score >= 80 AND no critical uncertainties"
      iterate: "60 <= score < 80"
      escalate: "score < 60 OR critical uncertainties present"
```

### Uncertainty Assessment Schema (CRITICAL)

```yaml
uncertainty_assessment:
  purpose: |
    Capture what the LLM doesn't know or is uncertain about.
    This information is for HUMAN review, not for automatic escalation.

  required_output:
    unknowns:
      type: "array"
      description: "List of information that is missing or uncertain"
      example:
        - "Exact API rate limits for production environment"
        - "User preference for error notification channel"

    assumptions:
      type: "array"
      description: "List of assumptions this output relies on"
      example:
        - "Assuming PostgreSQL 14+ is available"
        - "Assuming team uses Slack for notifications"

    reversibility:
      type: "string"
      enum: ["yes", "no", "partially"]
      description: "Is this decision easily reversible?"
      guidance:
        yes: "Can be changed with minimal effort (config change, feature flag)"
        no: "Requires significant rework (database schema, API contract)"
        partially: "Some aspects reversible, others not"

    impact_scope:
      type: "array"
      description: "What downstream phases does this affect?"
      example:
        - "Phase 4: Implementation planning depends on this architecture"
        - "Phase 7: Build will use these specifications"

  usage_in_gates:
    critical_gates: "Uncertainty assessment is SHOWN to human reviewer"
    standard_gates: "Uncertainty assessment is LOGGED but doesn't auto-escalate"
    rationale: |
      FINDING-005: LLM uncertainty self-assessment is not reliable enough
      to trigger automatic escalation. Humans must interpret uncertainty.
```

### Escalation Decision Rules

```yaml
escalation_rules:
  rule_priority: "Rules are evaluated in order; first matching rule applies"

  RULE-0:
    name: "Critical Gate Override"
    condition: "gate_type == 'critical'"
    action: "ESCALATE"
    rationale: "Critical gates (2, 4, 6) always require human review"
    message: "Critical gate requires human approval"

  RULE-1:
    name: "Circuit Breaker Active"
    condition: "any_circuit_breaker_open == true"
    action: "ESCALATE"
    rationale: "System is in degraded state; human should decide"
    message: "Circuit breaker {breaker_id} is open"

  RULE-2:
    name: "Health Critical"
    condition: "system_health == 'critical'"
    action: "ESCALATE"
    rationale: "API failures may affect output quality"
    message: "System health is critical: {health_summary}"

  RULE-3:
    name: "Score Below Minimum"
    condition: "gate_score < 60"
    action: "ESCALATE"
    rationale: "Output quality is below acceptable threshold"
    message: "Gate score {score} is below minimum (60)"

  RULE-4:
    name: "Score In Iteration Range"
    condition: "60 <= gate_score < 80"
    action: "ITERATE"
    max_iterations: 3
    on_max_iterations: "ESCALATE"
    rationale: "Output may improve with iteration"
    message: "Gate score {score} in iteration range. Iteration {n} of 3."

  RULE-5:
    name: "Score Acceptable"
    condition: "gate_score >= 80"
    action: "PROCEED"
    rationale: "Output meets quality threshold"
    message: "Gate passed with score {score}"

  RULE-6:
    name: "Oscillation Detected"
    condition: "oscillation_detected == true"
    action: "ESCALATE"
    rationale: "Iteration is not converging; human decision needed"
    message: "Gate scores are oscillating: {score_history}"

  evaluation_order: [RULE-0, RULE-1, RULE-2, RULE-6, RULE-3, RULE-4, RULE-5]
```

### Adversarial Pass (INFORMATIONAL ONLY)

```yaml
adversarial_pass:
  purpose: |
    Prompt the LLM to identify potential failure modes that the
    scoring rubric might miss. This is INFORMATIONAL for human
    reviewers, NOT an automatic escalation trigger.

  applies_to: [gate_2, gate_4, gate_6]
  note: "Only critical gates; standard gates skip adversarial pass"

  prompt_template: |
    Review this output and list 3 specific ways it could fail or cause
    problems downstream that the scoring rubric might miss.

    Consider:
    - Edge cases not covered
    - Assumptions that might be wrong
    - Integration points that could break
    - Scale or performance issues
    - Security or data integrity risks

  output_schema:
    failure_modes:
      type: "array"
      min_items: 3
      max_items: 5
      items:
        description:
          type: "string"
          description: "What could go wrong"
        likelihood:
          type: "string"
          enum: ["low", "medium", "high"]
        impact:
          type: "string"
          enum: ["low", "medium", "high"]
        mitigation:
          type: "string"
          description: "How to prevent or detect this failure"

  usage:
    automatic_escalation: false
    rationale: |
      FINDING-005: LLM self-critique is unreliable.
      Adversarial findings are SHOWN to human reviewer but do NOT
      automatically trigger escalation. Human decides if findings
      are significant.

    display_to_human: true
    display_format: |
      ## Adversarial Review (Informational)
      The following potential issues were identified:
      {formatted_failure_modes}

      These findings are for your consideration and do not
      automatically affect the gate decision.
```

### Time-Aware Escalation

```yaml
time_aware_escalation:
  purpose: "Route escalations appropriately based on operator availability"

  operator_hours:
    timezone: "America/New_York"
    available_hours: "08:00-22:00"
    note: "Adjust based on actual operator availability"

  escalation_routing:
    during_operator_hours:
      channel: "primary"
      method: "webhook_high_priority"
      expected_response_time: "30 minutes"
      message_prefix: "🔔 GATE REVIEW NEEDED"

    outside_operator_hours:
      channel: "deferred"
      method: "queue_for_morning"
      storage: "GitHub /state/pending-escalations.json"
      message_prefix: "📋 QUEUED FOR REVIEW"

  morning_batch:
    trigger_time: "08:00 ET"
    action: "Send summary of all queued escalations"
    format: |
      ## Morning Gate Review Queue
      {count} gates pending review from overnight runs:
      {formatted_list}

  urgent_override:
    condition: "cost_threshold_exceeded OR meta_breaker_tripped"
    action: "Escalate immediately regardless of time"
    channel: "emergency"
    note: "Some issues can't wait for morning"
```

### Gate Decision Logging

```yaml
gate_decision_logging:
  purpose: "Maintain audit trail of all gate decisions for analysis"
  storage: "GitHub /logs/{run_id}/gates/{gate_id}.json"

  log_schema:
    gate_id:
      type: "string"
      pattern: "gate_{number}"

    run_id:
      type: "string"
      format: "UUID"

    timestamp:
      type: "string"
      format: "ISO8601"

    phase_before:
      type: "string"

    phase_after:
      type: "string"

    gate_type:
      type: "string"
      enum: ["critical", "standard"]

    input_hash:
      type: "string"
      description: "Hash of gate input for consistency detection"

    scores:
      rubric_score:
        type: "number"
        description: "Score from rubric evaluation"
      iteration_number:
        type: "integer"
      score_history:
        type: "array"
        items: "number"

    uncertainty_assessment:
      unknowns:
        type: "array"
      assumptions:
        type: "array"
      reversibility:
        type: "string"
      impact_scope:
        type: "array"

    adversarial_pass:
      performed:
        type: "boolean"
      failure_modes:
        type: "array or null"

    decision:
      action:
        type: "string"
        enum: ["proceed", "iterate", "escalate"]
      rule_applied:
        type: "string"
        description: "Which rule triggered this decision"
      message:
        type: "string"

    escalation_details:
      escalated:
        type: "boolean"
      channel:
        type: "string or null"
      response_received:
        type: "boolean or null"
      response_timestamp:
        type: "string or null"
      human_decision:
        type: "string or null"
        enum: ["approve", "reject", "iterate", null]

    health_context:
      system_status:
        type: "string"
      api_health_summary:
        type: "object"
      circuit_breakers_open:
        type: "array"
```

### Gate Rubric Structure

```yaml
gate_rubric:
  purpose: "Standardized scoring criteria for gate evaluation"

  rubric_schema:
    gate_id:
      type: "string"

    criteria:
      type: "array"
      items:
        name:
          type: "string"
          description: "Criterion name"
        weight:
          type: "number"
          description: "Weight (0.0 to 1.0, sum to 1.0)"
        description:
          type: "string"
          description: "What this criterion evaluates"
        scoring_guide:
          excellent:
            score: 100
            description: "Criteria for excellent"
          good:
            score: 80
            description: "Criteria for good"
          acceptable:
            score: 60
            description: "Criteria for acceptable"
          poor:
            score: 40
            description: "Criteria for poor"
          fail:
            score: 0
            description: "Criteria for fail"

  example_rubric:
    gate_id: "gate_2"
    criteria:
      - name: "Completeness"
        weight: 0.3
        description: "All required sections present and filled"
      - name: "Technical Accuracy"
        weight: 0.3
        description: "Technical details are correct and feasible"
      - name: "Clarity"
        weight: 0.2
        description: "Document is clear and unambiguous"
      - name: "Alignment"
        weight: 0.2
        description: "Output aligns with project goals and constraints"

  score_calculation: |
    total_score = sum(criterion.score * criterion.weight for criterion in criteria)
```

## n8n Implementation Pattern

```yaml
n8n_implementation:
  gate_evaluation_workflow:
    description: "Reusable sub-workflow for gate evaluation"

    inputs:
      - gate_id: "Which gate (gate_1, gate_2, etc.)"
      - phase_output: "Output from the preceding phase"
      - run_context: "run_id, health_summary, circuit_breaker_state"

    nodes:
      1_determine_gate_type:
        type: "Code"
        purpose: "Classify gate as critical or standard"
        code_pattern: |
          const criticalGates = ['gate_2', 'gate_4', 'gate_6'];
          const gateType = criticalGates.includes($input.gate_id)
            ? 'critical'
            : 'standard';
          return { gate_type: gateType };

      2_check_circuit_breakers:
        type: "Code"
        purpose: "Check if any breakers are open"
        code_pattern: |
          const openBreakers = Object.entries($input.circuit_breaker_state.breakers)
            .filter(([id, state]) => state.state === 'open')
            .map(([id]) => id);
          return { open_breakers: openBreakers, any_open: openBreakers.length > 0 };

      3_evaluate_rubric:
        type: "HTTP Request (Claude)"
        purpose: "Score output against rubric"
        prompt_includes:
          - "Phase output"
          - "Rubric criteria"
          - "Scoring guide"
        output: "{ scores: {...}, total_score: number }"

      4_generate_uncertainty_assessment:
        type: "HTTP Request (Claude)"
        purpose: "Generate uncertainty assessment"
        prompt_includes:
          - "Phase output"
          - "Uncertainty schema"
        output: "{ unknowns: [], assumptions: [], reversibility: string, impact_scope: [] }"

      5_adversarial_pass:
        type: "IF + HTTP Request"
        condition: "gate_type == 'critical'"
        true_branch:
          type: "HTTP Request (Claude)"
          purpose: "Generate adversarial findings"
          prompt: "{{adversarial_prompt_template}}"
        false_branch: "Skip (return null)"

      6_apply_escalation_rules:
        type: "Code"
        purpose: "Evaluate rules in order, determine action"
        code_pattern: |
          const rules = [
            { id: 'RULE-0', condition: () => $input.gate_type === 'critical', action: 'escalate' },
            { id: 'RULE-1', condition: () => $input.any_breaker_open, action: 'escalate' },
            { id: 'RULE-2', condition: () => $input.health_status === 'critical', action: 'escalate' },
            { id: 'RULE-6', condition: () => $input.oscillation_detected, action: 'escalate' },
            { id: 'RULE-3', condition: () => $input.score < 60, action: 'escalate' },
            { id: 'RULE-4', condition: () => $input.score >= 60 && $input.score < 80, action: 'iterate' },
            { id: 'RULE-5', condition: () => $input.score >= 80, action: 'proceed' }
          ];

          for (const rule of rules) {
            if (rule.condition()) {
              return { action: rule.action, rule_applied: rule.id };
            }
          }
          return { action: 'escalate', rule_applied: 'FALLBACK' };

      7_route_action:
        type: "Switch"
        routing_field: "action"
        routes:
          proceed: "Continue to next phase"
          iterate: "Loop back to phase with feedback"
          escalate: "Send to escalation handler"

      8_log_decision:
        type: "HTTP Request"
        purpose: "Write gate decision to GitHub"
        method: "PUT"
        url: "https://api.github.com/repos/{owner}/{repo}/contents/logs/{run_id}/gates/{gate_id}.json"

  escalation_handler:
    description: "Handle escalation routing based on time"

    nodes:
      1_check_operator_hours:
        type: "Code"
        purpose: "Determine if within operator hours"
        code_pattern: |
          const now = new Date();
          const etHour = new Date(now.toLocaleString('en-US', { timeZone: 'America/New_York' })).getHours();
          const isOperatorHours = etHour >= 8 && etHour < 22;
          return { is_operator_hours: isOperatorHours };

      2_route_escalation:
        type: "IF"
        condition: "{{ $json.is_operator_hours == true }}"
        true_branch: "Send immediate notification"
        false_branch: "Queue for morning batch"

      3_immediate_notification:
        type: "HTTP Request"
        purpose: "Send webhook notification"
        url: "{{$env.ESCALATION_WEBHOOK_URL}}"

      4_queue_for_morning:
        type: "HTTP Request"
        purpose: "Append to pending-escalations.json in GitHub"
        method: "PUT"
        url: "https://api.github.com/repos/{owner}/{repo}/contents/state/pending-escalations.json"

  iteration_handler:
    description: "Handle iteration with oscillation detection"

    nodes:
      1_record_score:
        type: "Code"
        purpose: "Add score to history, check for oscillation"
        code_pattern: |
          const history = $input.score_history || [];
          history.push($input.current_score);

          let oscillating = false;
          if (history.length >= 4) {
            const recent = history.slice(-4);
            const deltas = [];
            for (let i = 0; i < recent.length - 1; i++) {
              deltas.push(recent[i + 1] - recent[i]);
            }
            const signs = deltas.map(d => Math.sign(d));
            oscillating = (signs[0] !== signs[1]) && (signs[1] !== signs[2]);
          }

          return {
            score_history: history,
            iteration_count: history.length,
            oscillation_detected: oscillating
          };

      2_check_max_iterations:
        type: "IF"
        condition: "{{ $json.iteration_count >= 3 || $json.oscillation_detected }}"
        true_branch: "Escalate"
        false_branch: "Continue iteration"
```

## Cross-References

| Related Spec | Relationship | Key Integration Points |
|--------------|--------------|------------------------|
| [health-monitoring-v2.1.md](../reliability/health-monitoring-v2.1.md) | depends on | Health status feeds RULE-2 |
| [circuit-breakers-v2.1.md](../reliability/circuit-breakers-v2.1.md) | depends on | Breaker state feeds RULE-1, oscillation feeds CB-4 |
| [error-handling-v2.1.md](../reliability/error-handling-v2.1.md) | related | Escalation channels align |
| [phase-io-contracts.md](../io-contracts/phase-io-contracts.md) | depended by | Gate decision schema used in phase outputs |
| [checkpoint-spec.md](../checkpoints/checkpoint-spec.md) | depended by | Gate decisions trigger checkpoints |

## Validation Criteria

1. **Critical Gate Test:** Verify gates 2, 4, 6 always escalate regardless of score
2. **Rule Priority Test:** Trigger multiple rules, verify highest priority applies
3. **Oscillation Test:** Feed alternating scores, verify RULE-6 triggers
4. **Time-Aware Test:** Test escalation routing during and outside operator hours
5. **Adversarial Pass Test:** Verify adversarial findings are logged but don't auto-escalate
6. **Decision Logging Test:** Verify all gate decisions are logged to GitHub

## Changelog

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 2.1.1 | 2025-12-05 | Devin | Initial creation addressing FINDING-005 (adversarial pass informational only, critical gates always human) |
