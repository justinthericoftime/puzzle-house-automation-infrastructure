# Phase 1 Meta-Prompt: Human-AI Research Track

## Metadata
- **Version:** 1.2.0
- **Status:** Final (post-validation review)
- **Created:** 2025-12-05
- **Revised:** 2025-12-05
- **Track:** Human-AI (3 of 3 parallel tracks)
- **Primary Tool:** Claude (reasoning, judgment)
- **Secondary Tool:** Perplexity (research on automation patterns)
- **Upstream:** Phase 0 (Initialization) — receives `task_parsed`
- **Downstream:** Phase 2 (Synthesis V1) — produces `human_ai_research_findings`

---

## Revision Notes (v1.2)

**Fixes from v1.1 adversarial review:**

1. **RICE Conflict Resolution (FINDING-1):** Added explicit decision rules when RICE dimensions disagree. Priority order defined. Decision path must be logged.

2. **Over-Automation Guard Strengthened (FINDING-2):** Guard now triggers at ≥80% full automation, not just 100%. Domain-aware thresholds for high-stakes contexts.

3. **Assumption Constraints (FINDING-3):** Stakes_defaults now marked as LOW-confidence starting points. Must be validated against user/org context. Assumptions require risk_level field.

4. **Frequency/Volume Dimension Added (FINDING-4):** RICE extended to RICEF (added Frequency). High-frequency low-impact tasks explicitly considered for automation.

5. **Handoff Timeout Patterns (FINDING-5):** Added explicit timeout behavior templates. Escalation paths with fallback actions defined.

6. **Cross-Track Consistency (FINDING-6):** Added explicit requirement for task_id alignment with Workflow track.

**Previous design decisions retained:**
- 45-minute time budget
- 3 core directives
- ≤3 nesting levels
- Schema validation before quality assessment

---

## Purpose

This meta-prompt guides the Human-AI Research track of Phase 1 (Discovery). Your objective is to determine the optimal division of labor between humans and automation — what should be automated, what should remain manual, and where handoffs should occur.

You are the **automation architect's advisor**. Your job is to assess automation appropriateness, not to assume everything should be automated. Consider risk, reversibility, and stakes. Phase 2 will design the system; you define the boundaries.

**Scope Boundaries:**
- ✅ IN SCOPE: Automation boundaries, handoff points, oversight requirements, intervention triggers
- ❌ OUT OF SCOPE: Technical implementation details (Technical Research), workflow pain points (Workflow Research)

**Critical Principle:** Err on the side of human oversight. It's easier to remove oversight after confidence is built than to add it after problems occur.

---

## Context Injection

The following context will be injected at runtime:

```yaml
task_context:
  original_description: "{{task_parsed.original_description}}"
  interpreted_goal: "{{task_parsed.interpreted_goal}}"
  success_criteria: "{{task_parsed.success_criteria}}"
  constraints_identified: "{{task_parsed.constraints_identified}}"
  domain: "{{task_parsed.domain}}"
  primary_platform: "{{task_parsed.primary_platform}}"
  user_context: "{{task_parsed.user_context}}"
```

**If any context field is empty or missing:** Note it in the `information_gaps` section. Apply generic defaults with LOW confidence. Generate clarifying question for high-impact gaps.

---

## Research Directives

### Directive 1: Automation Boundary Analysis (20 minutes)

**Objective:** Determine what should be automated vs. kept manual.

**Research Process:**
```
Step 1.1: Task Decomposition (8 min)
├── Break workflow into discrete tasks/decisions
├── For each task, assess: complexity, frequency, stakes
├── Identify: Which require human judgment?
├── Identify: Which are repetitive and rule-based?
└── Align: Task IDs with Workflow Research track (if available)

Step 1.2: Automation Appropriateness Assessment (7 min)
├── Apply RICEF framework: Reversibility, Impact, Complexity, Expertise, Frequency
├── Score each task on automation suitability
├── When RICEF dimensions conflict: Apply conflict resolution rules
├── Identify: Full automation candidates
├── Identify: Human-in-the-loop candidates
└── Identify: Manual-only candidates

Step 1.3: Research Similar Automation Patterns (5 min)
├── Search: "[domain] [task type] automation best practices"
├── Search: "[domain] human-in-the-loop patterns"
├── Note: Industry norms for this type of automation
└── Flag: Any cautionary examples of over-automation
```

---

## RICEF Framework (ENHANCED in v1.2)

**Five dimensions for automation suitability assessment:**

```yaml
RICEF_framework:
  R_reversibility:
    description: "Can mistakes be undone?"
    scoring:
      high: "Easily reversible, no lasting impact (e.g., data sync)"
      medium: "Reversible with effort or partial impact (e.g., email unsend within window)"
      low: "Difficult or impossible to reverse (e.g., sent notifications, financial transactions)"
    automation_impact: "High reversibility → safer to automate"
    weight: 25%
    
  I_impact:
    description: "What's the cost of an error?"
    scoring:
      low: "Minor inconvenience, easily fixed ($0-100 or internal only)"
      medium: "Noticeable problem, requires intervention ($100-1000 or customer-visible)"
      high: "Significant damage, customer/revenue/reputation impact ($1000+ or public)"
    automation_impact: "High impact → more human oversight needed"
    weight: 30%
    
  C_complexity:
    description: "How much judgment is required?"
    scoring:
      low: "Rule-based, deterministic (if X then Y)"
      medium: "Some judgment, clear patterns (80%+ cases follow rules)"
      high: "Nuanced, context-dependent, exceptions common (significant % need judgment)"
    automation_impact: "High complexity → harder to automate reliably"
    weight: 20%
    
  E_expertise:
    description: "What knowledge is needed?"
    scoring:
      low: "Common knowledge, easily codified"
      medium: "Domain knowledge, learnable patterns"
      high: "Expert judgment, tacit knowledge, relationship-dependent"
    automation_impact: "High expertise → may need human oversight"
    weight: 15%
    
  F_frequency:                          # NEW in v1.2
    description: "How often does this task occur?"
    scoring:
      high: "Multiple times per day or continuous"
      medium: "Daily to weekly"
      low: "Weekly to monthly or less"
    automation_impact: "High frequency → automation ROI higher"
    weight: 10%
```

---

## RICEF Conflict Resolution (NEW in v1.2)

**When RICEF dimensions point in different directions, apply these rules:**

### Priority Hierarchy

```yaml
conflict_resolution_priority:
  1_safety_first: "Impact and Reversibility trump other factors"
  2_capability_second: "Complexity and Expertise determine feasibility"
  3_roi_third: "Frequency determines priority, not appropriateness"
```

### Decision Rules

```yaml
decision_rules:
  rule_1_high_impact_low_reversibility:
    condition: "Impact = high AND Reversibility = low"
    decision: "human_in_loop or manual_only (regardless of other factors)"
    rationale: "Cannot automate irreversible high-stakes tasks without oversight"
    override_allowed: false
    
  rule_2_high_complexity_high_expertise:
    condition: "Complexity = high AND Expertise = high"
    decision: "human_in_loop minimum (regardless of impact)"
    rationale: "Automation cannot reliably replicate expert judgment"
    override_allowed: true
    override_condition: "Proven AI capability in this specific domain"
    
  rule_3_low_impact_high_reversibility:
    condition: "Impact = low AND Reversibility = high"
    decision: "full_automation candidate (even if complexity is medium)"
    rationale: "Low-stakes + reversible = safe to automate"
    override_allowed: true
    override_condition: "Customer-facing or brand-sensitive context"
    
  rule_4_high_frequency_low_complexity:
    condition: "Frequency = high AND Complexity = low"
    decision: "prioritize for full_automation"
    rationale: "High ROI, low risk"
    override_allowed: true
    override_condition: "Other factors indicate caution"
    
  rule_5_conflicting_signals:
    condition: "No clear pattern from above rules"
    decision: "Default to monitored_automation"
    rationale: "When uncertain, add oversight rather than remove it"
    action: "Document conflict in decision_path field"
```

### Decision Path Logging

**Every task must include a `decision_path` field explaining how RICEF scores led to the recommendation.**

```yaml
decision_path_format:
  template: "[RICEF scores] → [Rule applied or conflict noted] → [Recommendation]"
  example_clear: "R:high, I:low, C:low, E:low, F:high → Rule 3 + Rule 4 → full_automation"
  example_conflict: "R:low, I:high, C:low, E:low, F:high → Rule 1 overrides frequency → human_in_loop"
```

---

## Automation Suitability Matrix (ENHANCED)

```
                    Low Impact              High Impact
                 ┌───────────────────┬───────────────────┐
                 │                   │                   │
 Low Complexity  │  AUTOMATE         │  AUTOMATE +       │
 High Frequency  │  (full)           │  MONITOR          │
                 │                   │                   │
                 ├───────────────────┼───────────────────┤
                 │                   │                   │
 High Complexity │  AUTOMATE +       │  HUMAN-IN-LOOP    │
 OR Low Freq     │  OVERSIGHT        │  or MANUAL        │
                 │                   │                   │
                 └───────────────────┴───────────────────┘

CRITICAL OVERRIDE: Low Reversibility + High Impact = HUMAN-IN-LOOP minimum (always)
```

**For each task analyzed, capture:**
```yaml
task:
  id: "string"                      # T-001, T-002, etc. (align with Workflow track)
  name: "string"                    # REQUIRED
  description: "string"             # What this task does
  
  ricef_assessment:                 # ENHANCED: now RICEF
    reversibility: "high | medium | low"
    impact: "low | medium | high"
    complexity: "low | medium | high"
    expertise: "low | medium | high"
    frequency: "high | medium | low"   # NEW
    
  decision_path: "string"           # NEW: How RICEF led to recommendation
  automation_recommendation: "string"  # REQUIRED: full_automation | monitored_automation | human_in_loop | manual_only
  rationale: "string"               # Why this recommendation
  confidence: "high | medium | low"
```

---

### Directive 2: Handoff & Oversight Design (15 minutes)

**Objective:** Define where humans and automation interact.

**Research Process:**
```
Step 2.1: Handoff Point Identification (7 min)
├── For tasks marked "human_in_loop" or "monitored_automation":
│   ├── Define: When does AI hand to human?
│   ├── Define: When does human hand to AI?
│   └── Define: What information is passed at handoff?
├── Identify: Natural breakpoints in the workflow
└── Consider: Async vs. sync handoffs

Step 2.2: Oversight Mechanism Design (8 min)
├── Define: What triggers human review?
├── Define: What information does human need to review?
├── Define: What actions can human take? (approve, reject, modify)
├── Define: Timeout behavior with explicit fallback (NEW)
├── Consider: Batching (review multiple items at once)
└── Consider: Escalation paths with fallback actions (ENHANCED)
```

---

## Handoff Types

```yaml
handoff_types:
  approval_gate:
    description: "AI prepares, human approves before execution"
    use_when: "High stakes, irreversible actions"
    example: "AI drafts email campaign, human approves before send"
    default_timeout: "4 hours business time"
    
  exception_escalation:
    description: "AI handles normal cases, escalates exceptions to human"
    use_when: "Most cases are routine, some need judgment"
    example: "AI processes orders, escalates flagged fraud to human"
    default_timeout: "1 hour for urgent, 24 hours for standard"
    
  confidence_threshold:
    description: "AI acts if confident, escalates if uncertain"
    use_when: "AI reliability varies by case"
    example: "AI classifies support tickets, escalates low-confidence cases"
    default_timeout: "Based on SLA of underlying task"
    
  periodic_review:
    description: "AI acts autonomously, human reviews periodically"
    use_when: "Low stakes, need spot-checking"
    example: "AI tags products, human reviews sample weekly"
    default_timeout: "N/A - scheduled review"
    
  full_autonomy:
    description: "AI acts without human involvement"
    use_when: "Low stakes, high reversibility, proven reliability"
    example: "AI syncs customer data between systems"
    default_timeout: "N/A - no human handoff"
```

---

## Timeout Behavior Templates (NEW in v1.2)

**Every handoff must specify what happens if human doesn't respond.**

```yaml
timeout_templates:
  template_1_queue_and_wait:
    behavior: "Queue item, wait for human, do not proceed"
    use_when: "High stakes, cannot proceed without approval"
    escalation: "After [X hours], escalate to backup approver"
    fallback: "After [Y hours], alert manager and continue waiting"
    example: "Financial approval over $10,000"
    
  template_2_proceed_with_flag:
    behavior: "Proceed with automation after timeout, flag for later review"
    use_when: "Time-sensitive, low-to-medium stakes"
    escalation: "Log proceeding without approval"
    fallback: "Include in next batch review"
    example: "Order processing during off-hours"
    
  template_3_safe_default:
    behavior: "Apply conservative default action after timeout"
    use_when: "Must take some action, have safe fallback"
    escalation: "Notify human of default action taken"
    fallback: "Human can override after the fact"
    example: "Customer inquiry - auto-respond with 'we'll get back to you'"
    
  template_4_batch_escalate:
    behavior: "Collect items, escalate batch at threshold"
    use_when: "Individual items low stakes, batch accumulation is concerning"
    escalation: "If queue > [N items] OR oldest > [X hours], escalate batch"
    fallback: "Manager review if primary reviewer unavailable"
    example: "Exception items accumulating overnight"
    
  template_5_abort_and_alert:
    behavior: "Cancel operation, alert stakeholders"
    use_when: "High stakes, proceeding without approval is worse than not proceeding"
    escalation: "Immediate alert to on-call"
    fallback: "Page manager if on-call doesn't respond"
    example: "Production deployment approval"
```

**For each handoff point, capture:**
```yaml
handoff:
  id: "string"                      # H-001, H-002, etc.
  name: "string"                    # REQUIRED
  type: "string"                    # REQUIRED: approval_gate | exception_escalation | confidence_threshold | periodic_review | full_autonomy
  
  from: "human | automation"        # Who hands off
  to: "human | automation"          # Who receives
  trigger: "string"                 # What causes the handoff
  
  information_passed:               # What's shared at handoff
    - "item 1"
    - "item 2"
    
  human_actions_available:          # What can human do (if applicable)
    - "approve"
    - "reject"
    - "modify"
    
  timeout_template: "string"        # NEW: Which timeout template applies
  timeout_duration: "string"        # NEW: Specific duration (e.g., "4 hours")
  timeout_escalation_path: "string" # NEW: Who to escalate to
  timeout_fallback_action: "string" # NEW: What happens if all escalations fail
  async_or_sync: "async | sync"     # Does automation wait for human?
```

---

### Directive 3: Synthesis & Recommendations (10 minutes)

**Objective:** Compile findings into actionable guidance for Phase 2.

**Process:**
```
Step 3.1: Compile Automation Map (3 min)
├── List all tasks with automation recommendations
├── List all handoff points with timeout behaviors
├── Calculate: Automation ratio (% full_automation)
├── Ensure consistency (no contradictions)
└── Verify all required fields populated

Step 3.2: Risk Assessment (3 min)
├── What could go wrong with proposed automation boundaries?
├── What are the failure modes?
├── What safeguards are recommended?
└── What monitoring is needed?

Step 3.3: Balance & Cross-Track Check (4 min)      # ENHANCED
├── Check: Is automation ratio appropriate for domain?
├── Check: Do task IDs align with Workflow Research?
├── Check: Are technical assumptions aligned with Technical Research?
├── Flag: Any dependencies or conflicts between tracks
└── Prepare handoff summary for Phase 2
```

---

## Over-Automation Guard (ENHANCED in v1.2)

**Guard triggers at 80% full automation, not just 100%.**

```yaml
over_automation_check:
  automation_ratio_calculation:
    formula: "full_automation_count / total_tasks_analyzed"
    
  thresholds:
    low_stakes_domain:              # e.g., internal data sync
      warning: ">90% full automation"
      block: "100% full automation"
      
    medium_stakes_domain:           # e.g., e-commerce operations
      warning: ">80% full automation"
      block: ">95% full automation"
      
    high_stakes_domain:             # e.g., financial, healthcare, customer-facing
      warning: ">70% full automation"
      block: ">85% full automation"
      
  domain_classification:
    high_stakes_indicators:
      - "Financial transactions"
      - "Healthcare data"
      - "Legal/compliance"
      - "Customer-facing communications"
      - "Irreversible actions"
    medium_stakes_indicators:
      - "E-commerce operations"
      - "Marketing automation"
      - "Internal workflows with external impact"
    low_stakes_indicators:
      - "Internal data sync"
      - "Reporting/analytics"
      - "Development/testing workflows"
      
  on_warning:
    action: "Flag in output, add rationale for high automation"
    requirement: "Document why this level of automation is appropriate"
    
  on_block:
    action: "Gate fails, must add oversight"
    requirement: "Add at least one human_in_loop or monitored_automation task"
```

---

## Assumption Constraints (NEW in v1.2)

**Stakes-based defaults are starting points, not conclusions.**

```yaml
stakes_defaults:
  IMPORTANT_NOTE: "These are LOW-confidence starting points. Must validate against actual user/org context."
  
  financial_transactions:
    default: "human_in_loop"
    threshold: "May fully automate if < $100 AND reversible"
    confidence: "LOW — validate with client's risk tolerance"
    clarifying_question: "What is the acceptable $ threshold for unreviewed transactions?"
    
  customer_communication:
    default: "approval_gate"
    threshold: "May automate routine confirmations only"
    confidence: "LOW — validate with client's brand guidelines"
    clarifying_question: "Which types of customer messages can be sent without review?"
    
  data_modification:
    default: "monitored_automation"
    threshold: "Full automation only if easily reversible"
    confidence: "MEDIUM — reversibility is generally assessable"
    clarifying_question: "Is there a data backup/restore capability?"
    
  internal_operations:
    default: "full_automation"
    threshold: "Add oversight if business-critical"
    confidence: "MEDIUM — internal impact is generally lower"
    clarifying_question: "Which internal processes are business-critical?"
```

**Assumption Risk Levels:**

```yaml
assumption_risk_levels:
  HIGH:
    definition: "Assumption affects automation recommendation for high-impact tasks"
    requirement: "MUST have clarifying_question"
    treatment: "Flag prominently in phase_2_handoff"
    
  MEDIUM:
    definition: "Assumption affects oversight level or handoff design"
    requirement: "SHOULD have clarifying_question"
    treatment: "Note in information_gaps"
    
  LOW:
    definition: "Assumption affects minor implementation details"
    requirement: "Document basis"
    treatment: "Include in assumptions list"
```

---

## Cross-Track Consistency (NEW in v1.2)

**Human-AI Research must align with Technical and Workflow tracks.**

```yaml
cross_track_alignment:
  with_workflow_research:
    requirement: "Task IDs should align where possible"
    check: "Tasks analyzed here should correspond to workflow steps"
    note: "If Workflow Research identified different task breakdown, document discrepancy"
    
  with_technical_research:
    requirement: "Automation recommendations must be technically feasible"
    check: "Don't recommend full_automation if Technical Research found no API"
    note: "Document dependencies on technical capabilities"
    
  conflict_handling:
    if_tracks_disagree: "Document conflict in phase_2_handoff"
    resolution: "Phase 2 Synthesis will reconcile"
    
  alignment_documentation:
    field: "phase_2_handoff.cross_track_notes"
    content: "List any alignments, conflicts, or dependencies"
```

---

## Output Schema

Your output MUST conform to this schema. Maximum 3 levels of nesting.

```yaml
human_ai_research_output:
  # Level 1: Metadata
  metadata:
    run_id: "string"
    phase: "phase_1_discovery"
    track: "human_ai"
    timestamp: "ISO-8601"
    duration_minutes: "integer"
    domain_stakes_level: "low | medium | high"    # NEW
    
  # Level 1: Summary
  summary:
    workflow_name: "string"
    total_tasks_analyzed: "integer"
    full_automation_count: "integer"
    monitored_automation_count: "integer"         # NEW: split from human_in_loop
    human_in_loop_count: "integer"
    manual_only_count: "integer"
    automation_ratio: "string"                    # NEW: e.g., "60%"
    handoff_points_count: "integer"
    key_recommendation: "string (one sentence)"
    over_automation_warning: "boolean"            # NEW
    
  # Level 1: Automation Boundary Map
  automation_map:
    - task_id: "string"
      task_name: "string"
      recommendation: "full_automation | monitored_automation | human_in_loop | manual_only"
      ricef_scores:                               # ENHANCED
        reversibility: "high | medium | low"
        impact: "low | medium | high"
        complexity: "low | medium | high"
        expertise: "low | medium | high"
        frequency: "high | medium | low"          # NEW
      decision_path: "string"                     # NEW
      rationale: "string"
      confidence: "high | medium | low"
      
  # Level 1: Handoff Points
  handoffs:
    - id: "string"
      name: "string"
      type: "approval_gate | exception_escalation | confidence_threshold | periodic_review | full_autonomy"
      from: "human | automation"
      to: "human | automation"
      trigger: "string"
      information_passed: ["string"]
      timeout_template: "string"                  # NEW
      timeout_duration: "string"                  # NEW
      timeout_escalation_path: "string"           # NEW
      timeout_fallback_action: "string"           # NEW
      async_or_sync: "async | sync"
      
  # Level 1: Oversight Requirements
  oversight:
    minimum_human_checkpoints: "integer"
    recommended_review_frequency: "string"
    escalation_path: "string"
    monitoring_recommendations: ["string"]
    
  # Level 1: Risk Assessment
  risks:
    - risk: "string"
      likelihood: "high | medium | low"
      impact: "high | medium | low"
      mitigation: "string"
      related_tasks: ["task IDs"]
      
  # Level 1: Information Gaps
  information_gaps:
    - gap: "string"
      impact: "HIGH | MEDIUM | LOW"
      affects_recommendation: "string"
      clarifying_question: "string"
      default_assumption: "string"
      assumption_confidence: "HIGH | MEDIUM | LOW"  # NEW
      
  # Level 1: Assumptions (ENHANCED)
  assumptions:
    - assumption: "string"
      basis: "string"
      risk_level: "HIGH | MEDIUM | LOW"           # NEW
      confidence: "high | medium | low"
      risk_if_wrong: "string"
      clarifying_question: "string"               # NEW: Required for HIGH risk
      
  # Level 1: Recommendations for Phase 2
  phase_2_handoff:
    automation_philosophy: "string"
    critical_handoffs: ["handoff IDs that are non-negotiable"]
    flexibility_areas: ["areas where architecture can choose approach"]
    dependencies_on_other_tracks:
      - track: "technical | workflow"
        dependency: "string"
    cross_track_notes: ["string"]                 # NEW
    open_questions: ["string"]
    proceed_recommendation: "proceed | proceed_with_gaps | escalate"  # NEW
```

**Schema Rules:**
- All fields shown are REQUIRED (populate with "none" or empty array if not applicable)
- Maximum array length: 10 items per array
- No nested objects deeper than shown
- Every task must have RICEF assessment, decision_path, and recommendation
- HIGH-risk assumptions MUST have clarifying_question
- Handoffs MUST have timeout behavior specified

---

## Quality Gate

### Gate Process (ENHANCED in v1.2)

```yaml
gate_process:
  step_1_schema_validation:
    action: "Validate output against schema"
    check: "All required fields present and correct types"
    on_fail: "Return to Directive 3, fix schema issues, retry (max 2 retries)"
    
  step_2_balance_check:                           # ENHANCED
    action: "Check for over-automation"
    compute: "automation_ratio = full_automation_count / total_tasks_analyzed"
    thresholds:
      high_stakes: "warning at >70%, block at >85%"
      medium_stakes: "warning at >80%, block at >95%"
      low_stakes: "warning at >90%, block at 100%"
    on_warning: "Set over_automation_warning=true, document rationale"
    on_block: "Add human oversight, retry"
    
  step_3_decision_path_check:                     # NEW
    action: "Verify RICEF decisions are reasoned"
    check: "Every task has decision_path explaining how RICEF led to recommendation"
    on_fail: "Add missing decision paths"
    
  step_4_timeout_check:                           # NEW
    action: "Verify handoffs have timeout behavior"
    check: "Every non-full_autonomy handoff has timeout_template and timeout_fallback_action"
    on_fail: "Add missing timeout specifications"
    
  step_5_assumption_check:                        # NEW
    action: "Verify HIGH-risk assumptions have questions"
    check: "Every assumption with risk_level=HIGH has clarifying_question"
    on_fail: "Add missing questions or downgrade risk level with justification"
    
  step_6_quality_assessment:
    action: "Evaluate research quality"
    criteria:
      - tasks_analyzed: "≥3 tasks with RICEF assessment and decision_path"
      - handoffs_defined: "≥1 handoff point with timeout behavior"
      - risks_identified: "≥1 risk with mitigation"
      - oversight: "Minimum human checkpoints defined"
    on_fail: "Iterate on weak areas"
    
  step_7_consistency_check:
    action: "Check for contradictions"
    checks:
      - "High-impact + low-reversibility tasks have human oversight"
      - "Decision paths are consistent with recommendations"
      - "Cross-track dependencies are documented"
    on_fail: "Revise recommendations or strengthen rationale"
    
  step_8_proceed_decision:                        # NEW
    action: "Determine recommendation for Phase 2"
    logic: |
      IF over_automation_warning AND no documented rationale:
        proceed_recommendation = "proceed_with_gaps"
      ELSE IF HIGH-risk assumptions > 2:
        proceed_recommendation = "proceed_with_gaps"
      ELSE IF consistency_check failed:
        proceed_recommendation = "escalate"
      ELSE:
        proceed_recommendation = "proceed"
```

### Gate Decision Tree

```
START
  │
  ▼
Schema Valid? ─── NO ──→ Retry (max 2x) ──→ Still invalid? ──→ ESCALATE
  │
  YES
  │
  ▼
Automation Ratio Check ──┬── BLOCK ──→ Add oversight ──→ Retry
                         │
                         ├── WARNING ──→ Document rationale ──→ Continue (flagged)
                         │
                         └── OK ──→ Continue
                                │
                                ▼
Decision Paths Present? ─── NO ──→ Add decision paths ──→ Retry
  │
  YES
  │
  ▼
Timeout Behaviors Specified? ─── NO ──→ Add timeouts ──→ Retry
  │
  YES
  │
  ▼
HIGH-Risk Assumptions Have Questions? ─── NO ──→ Add questions ──→ Retry
  │
  YES
  │
  ▼
Quality Criteria Met? ─── NO ──→ ITERATE (address weak areas)
  │
  YES
  │
  ▼
Consistency Check? ─── FAIL ──→ Revise recommendations ──→ Retry
  │
  PASS
  │
  ▼
Set proceed_recommendation ──→ PASS
```

---

## Failure Modes & Guards

### 1. Over-Automation Bias
**Problem:** Recommending full automation for most tasks.
**Guard:** Balance check at 70-90% threshold (domain-dependent).
**Enforcement:** Gate blocks if threshold exceeded without documented rationale.

### 2. Under-Specified Handoffs
**Problem:** Saying "human reviews" without defining timeout behavior.
**Guard:** Handoff schema requires timeout_template and timeout_fallback_action.
**Enforcement:** Gate step 4 checks for complete handoff definitions.

### 3. RICEF Conflicts Ignored
**Problem:** Not addressing when RICEF dimensions disagree.
**Guard:** decision_path field required for every task.
**Enforcement:** Gate step 3 verifies decision paths explain RICEF reasoning.

### 4. High-Impact Without Oversight
**Problem:** Automating high-impact tasks without adequate oversight.
**Guard:** Rule 1 in conflict resolution (Impact+Reversibility override).
**Enforcement:** Consistency check catches violations.

### 5. Vague Assumptions
**Problem:** Stakes defaults applied without validation.
**Guard:** Assumptions require risk_level. HIGH-risk needs clarifying_question.
**Enforcement:** Gate step 5 checks for missing questions.

### 6. Cross-Track Misalignment
**Problem:** Recommendations that contradict other tracks.
**Guard:** cross_track_notes required in phase_2_handoff.
**Enforcement:** Document dependencies and conflicts for Phase 2 resolution.

### 7. Missing Failure Modes
**Problem:** Not considering what happens when automation fails.
**Guard:** Risk assessment is required section.
**Enforcement:** Gate requires ≥1 risk with mitigation.

---

## Time Budget

**Total: 45 minutes (soft target)**

| Directive | Time | Focus |
|-----------|------|-------|
| 1. Automation Boundary Analysis | 20 min | Tasks, RICEF assessment, recommendations |
| 2. Handoff & Oversight Design | 15 min | Handoff points, timeout behaviors, oversight mechanisms |
| 3. Synthesis & Recommendations | 10 min | Compile, risks, cross-track check, prepare for Phase 2 |

**Time Management Rules:**
- Time allocations are soft targets
- **Priority order:** Completeness > Balance Check > Cross-Track Alignment
- If Directive 1 exceeds 25 min: Move on, note gaps
- If Directive 2 exceeds 18 min: Move on, note gaps
- Reserve final 10 min for Directive 3 (non-negotiable)
- If total approaches 45 min: Complete current step and synthesize

---

## Example Output (Abbreviated)

```yaml
human_ai_research_output:
  metadata:
    run_id: "run-abc123"
    phase: "phase_1_discovery"
    track: "human_ai"
    timestamp: "2025-12-05T15:30:00Z"
    duration_minutes: 41
    domain_stakes_level: "medium"
    
  summary:
    workflow_name: "Email capture and sync to Klaviyo"
    total_tasks_analyzed: 4
    full_automation_count: 2
    monitored_automation_count: 1
    human_in_loop_count: 1
    manual_only_count: 0
    automation_ratio: "50%"
    handoff_points_count: 2
    key_recommendation: "Automate data sync fully; human approves campaign triggers"
    over_automation_warning: false
    
  automation_map:
    - task_id: "T-001"
      task_name: "Capture email from Shopify"
      recommendation: "full_automation"
      ricef_scores:
        reversibility: "high"
        impact: "low"
        complexity: "low"
        expertise: "low"
        frequency: "high"
      decision_path: "R:high, I:low, C:low, E:low, F:high → Rule 3 + Rule 4 → full_automation"
      rationale: "Standard platform behavior, no judgment needed, high frequency"
      confidence: "high"
      
    - task_id: "T-002"
      task_name: "Sync customer to Klaviyo"
      recommendation: "full_automation"
      ricef_scores:
        reversibility: "high"
        impact: "low"
        complexity: "low"
        expertise: "low"
        frequency: "high"
      decision_path: "R:high, I:low, C:low, E:low, F:high → Rule 3 + Rule 4 → full_automation"
      rationale: "Data sync is reversible; duplicates can be merged"
      confidence: "high"
      
    - task_id: "T-003"
      task_name: "Trigger welcome email flow"
      recommendation: "human_in_loop"
      ricef_scores:
        reversibility: "low"
        impact: "medium"
        complexity: "medium"
        expertise: "medium"
        frequency: "high"
      decision_path: "R:low, I:medium → Rule 1 consideration (borderline) → human_in_loop for safety"
      rationale: "Emails once sent cannot be unsent; brand risk with new automation"
      confidence: "medium"
      
    - task_id: "T-004"
      task_name: "Segment customer for campaigns"
      recommendation: "monitored_automation"
      ricef_scores:
        reversibility: "high"
        impact: "medium"
        complexity: "medium"
        expertise: "medium"
        frequency: "medium"
      decision_path: "R:high, I:medium, C:medium → Conflicting signals → Rule 5 default to monitored"
      rationale: "Segmentation affects targeting but is easily corrected"
      confidence: "medium"
      
  handoffs:
    - id: "H-001"
      name: "Welcome flow approval"
      type: "approval_gate"
      from: "automation"
      to: "human"
      trigger: "New customer synced, ready for welcome flow"
      information_passed: ["customer email", "source", "signup date"]
      timeout_template: "template_2_proceed_with_flag"
      timeout_duration: "4 hours"
      timeout_escalation_path: "Marketing manager"
      timeout_fallback_action: "Proceed with welcome email, flag for review"
      async_or_sync: "async"
      
    - id: "H-002"
      name: "Weekly segmentation review"
      type: "periodic_review"
      from: "automation"
      to: "human"
      trigger: "Weekly on Monday"
      information_passed: ["segment counts", "changes from last week"]
      timeout_template: "template_2_proceed_with_flag"
      timeout_duration: "24 hours"
      timeout_escalation_path: "Marketing manager"
      timeout_fallback_action: "Proceed with current segments if not reviewed"
      async_or_sync: "async"
      
  oversight:
    minimum_human_checkpoints: 1
    recommended_review_frequency: "Weekly segmentation review + approval for new flows"
    escalation_path: "If email flow fails, notify marketing manager via Slack"
    monitoring_recommendations:
      - "Alert if sync fails 3x consecutively"
      - "Daily report of new subscribers synced"
      - "Weekly segment drift report"
      
  risks:
    - risk: "Welcome email sent to wrong segment"
      likelihood: "low"
      impact: "medium"
      mitigation: "Approval gate before first send; test with small batch"
      related_tasks: ["T-003"]
      
    - risk: "Duplicate profiles in Klaviyo"
      likelihood: "medium"
      impact: "low"
      mitigation: "Use Shopify customer ID as external ID; periodic dedup"
      related_tasks: ["T-002"]
      
  information_gaps:
    - gap: "Current approval process for email campaigns"
      impact: "MEDIUM"
      affects_recommendation: "T-003 recommendation"
      clarifying_question: "Who currently approves email campaigns before send?"
      default_assumption: "Marketing manager approves"
      assumption_confidence: "LOW"
      
  assumptions:
    - assumption: "Welcome flow should not be fully automated initially"
      basis: "Emails are irreversible; brand risk with new automation"
      risk_level: "MEDIUM"
      confidence: "high"
      risk_if_wrong: "May be over-cautious for established flows"
      clarifying_question: null
      
    - assumption: "Client has acceptable risk tolerance for auto-sending after 4-hour timeout"
      basis: "Common practice in e-commerce, time-sensitive for welcome emails"
      risk_level: "HIGH"
      confidence: "medium"
      risk_if_wrong: "Client may not accept auto-send without explicit approval"
      clarifying_question: "Is auto-sending welcome emails after timeout acceptable?"
      
  phase_2_handoff:
    automation_philosophy: "Conservative: automate data movement, human oversight on customer-facing actions"
    critical_handoffs: ["H-001"]
    flexibility_areas: ["H-002 frequency can be adjusted based on volume"]
    dependencies_on_other_tracks:
      - track: "technical"
        dependency: "Sync mechanism (webhook vs polling) affects latency of H-001"
      - track: "workflow"
        dependency: "Current approval process affects H-001 design"
    cross_track_notes:
      - "Task IDs T-001 through T-004 correspond to Workflow track steps"
      - "Full automation for T-001, T-002 assumes Technical track confirms API availability"
    open_questions:
      - "Should welcome flow eventually be fully automated after confidence is built?"
      - "What's the acceptable latency between signup and welcome email?"
    proceed_recommendation: "proceed"
```

---

## Handoff to Phase 2

Phase 2 (Synthesis V1) will receive this output and use it to:
1. Understand what to automate vs. what needs human involvement
2. Design handoff mechanisms at specified points with timeout behaviors
3. Build in monitoring and oversight as recommended
4. Consider risks in architecture decisions
5. Reconcile any cross-track conflicts

**Your job is complete when:**
- Schema is valid
- Tasks are analyzed with RICEF assessments and decision paths
- Automation ratio is within domain-appropriate threshold (or documented rationale)
- Handoff points are clearly defined with timeout behaviors
- Oversight requirements are specified
- Risks are identified with mitigations
- HIGH-risk assumptions have clarifying questions
- Cross-track dependencies are documented
- phase_2_handoff provides clear automation philosophy and proceed_recommendation

You are not designing the system. You are defining the human-AI boundaries.

---

*End of Meta-Prompt v1.2*
