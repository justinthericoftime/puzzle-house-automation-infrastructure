# Phase 1 Meta-Prompt: Workflow Research Track

## Metadata
- **Version:** 1.3.0
- **Status:** Final (post-validation review)
- **Created:** 2025-12-05
- **Revised:** 2025-12-05
- **Track:** Workflow (2 of 3 parallel tracks)
- **Primary Tool:** Claude (reasoning about provided context)
- **Secondary Tool:** Perplexity (research on similar workflows)
- **Upstream:** Phase 0 (Initialization) — receives `task_parsed`
- **Downstream:** Phase 2 (Synthesis V1) — produces `workflow_research_findings`

---

## Revision Notes (v1.3)

**Fixes from v1.2 validation review:**

1. **Citation Faithfulness (WF2-001):** Citations must now reference specific input elements. Generic "common pattern" justifications are banned unless marked low-confidence with explicit acknowledgment.

2. **Confidence Calibration (WF2-002):** Added mandatory reasoning step before confidence assignment. Conservative labeling required when derivation basis is weak.

3. **Moderate Input Thresholds (WF2-003):** Tightened proceed_with_gaps criteria. Requires ≥2 explicit items across different categories. If both stakeholders AND success_metrics are empty, must escalate.

4. **Sparse Precedence (WF2-004):** Explicitly stated that sparse_input_rules override proceed_decision logic. Sparse input always escalates.

5. **Threshold Edge Cases (WF2-008):** Clarified ">60%" means 61%+. Threshold check skipped when <3 derived items exist.

**Previous fixes (v1.2) retained:**
- Two-category classification (EXPLICIT/DERIVED)
- Input Richness Assessment (Step 0)
- Adaptive gate requirements
- Empty arrays valid for sparse/moderate input

---

## Purpose

This meta-prompt guides the Workflow Research track of Phase 1 (Discovery). Your objective is to understand the current state — existing processes, pain points, and business context — so that Phase 2 can design automation that solves real problems.

You are the **business analyst**. Your job is to understand what exists today and what needs to change. While Technical Research asks "What's possible?", you ask "What's needed?" The architecture phase will design; you diagnose.

**Scope Boundaries:**
- ✅ IN SCOPE: Current workflow, pain points, stakeholders, success metrics
- ❌ OUT OF SCOPE: Detailed volume analysis, comprehensive edge case enumeration (Phase 2 handles these)

**Critical Principle:** Report what you know, flag what you don't. Never fabricate detail to fill gaps.

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

---

## Step 0: Input Richness Assessment (2 minutes)

**Before any research, assess the input.**

```yaml
input_assessment:
  check_1: "Does task description contain workflow trigger? (event/schedule/manual)"
  check_2: "Does task description mention any current process steps?"
  check_3: "Does task description state any problems or pain points?"
  check_4: "Does task description mention any stakeholders or roles?"
  check_5: "Is business context provided? (why this matters)"
```

**Scoring:**
- **Rich Input (4-5 checks pass):** Proceed with full analysis. Standard gate requirements apply.
- **Moderate Input (2-3 checks pass):** Proceed with caution. Relaxed gate requirements. Flag gaps prominently.
- **Sparse Input (0-1 checks pass):** Enter Sparse Input Mode. Minimal analysis. Escalate with clarifying questions.

**Record the assessment in output metadata.**

---

## Research Directives

### Directive 1: Current State & Pain Points (20 minutes)

**Objective:** Understand how the process works today and what's broken.

**Research Process:**
```
Step 1.1: Extract Current State (8 min)
├── Read: Task description for explicit workflow details
├── Quote: Direct text for anything marked EXPLICIT
├── Derive: Steps that logically follow (mark as DERIVED)
├── Reason: State WHY each confidence level is assigned (see Confidence Protocol)
└── Flag: Information gaps (what's not clear)

Step 1.2: Identify Pain Points (7 min)
├── Extract: Explicitly stated problems (quote the text)
├── Derive: Likely problems based on workflow structure
├── Reason: Justify confidence for each derived pain point
└── Ban: Do NOT use generic "common pattern" claims as high/medium confidence

Step 1.3: Assess Automation Readiness (5 min)
├── Evaluate: How much is already automated vs manual?
├── Identify: What parts are automatable?
├── Flag: What might be difficult to automate?
└── Note: Dependencies on other systems
```

**For current state, capture:**
```yaml
current_state:
  workflow_name: "string"                    # REQUIRED
  trigger: "event | schedule | manual | unknown"  # REQUIRED
  automation_level: "manual | semi_automated | fully_automated | unknown"  # REQUIRED
  
  steps:                                     # REQUIRED: array of step objects
    - number: "integer"
      description: "string"
      actor: "human | system | unknown"
      tool: "string or null"
      evidence_type: "explicit | derived"
      evidence_citation: "string"            # See Citation Requirements below
      confidence: "high | medium | low"
      confidence_reasoning: "string"         # NEW: Why this confidence level?
      
  existing_tools: ["string"]
  outputs: ["string"]
```

**For each pain point, capture:**
```yaml
pain_point:
  id: "string"                               # PP-001, PP-002, etc.
  description: "string"                      # REQUIRED
  category: "speed | accuracy | cost | reliability | user_experience"
  severity: "critical | high | medium | low"
  evidence_type: "explicit | derived"
  evidence_citation: "string"                # See Citation Requirements below
  confidence: "high | medium | low"
  confidence_reasoning: "string"             # NEW: Why this confidence level?
  addressable_by_automation: "yes | partially | no | unknown"
```

---

### Directive 2: Stakeholders & Success Metrics (15 minutes)

**Objective:** Identify who's involved and how success will be measured.

**Research Process:**
```
Step 2.1: Stakeholder Identification (7 min)
├── Extract: Roles explicitly mentioned in task description
├── Derive: Standard roles IF workflow type is clear AND input is rich
├── Reason: Justify confidence for each stakeholder
└── Note: Primary user of automation (if identifiable)

Step 2.2: Success Metrics Definition (8 min)
├── Extract: Stated success criteria (quote directly)
├── Derive: Metrics that would indicate improvement
├── Allow: "unknown" for baseline if not stated
└── Define: Target based on stated or derived goals
```

**For each stakeholder, capture:**
```yaml
stakeholder:
  id: "string"                               # S-001, S-002, etc.
  role: "string"                             # REQUIRED
  type: "initiator | performer | recipient | approver | affected"
  needs: ["string"]                          # What they need (or ["unknown"])
  evidence_type: "explicit | derived"
  evidence_citation: "string"                # See Citation Requirements below
  confidence: "high | medium | low"
  confidence_reasoning: "string"             # NEW: Why this confidence level?
  primary_user: "boolean"
```

**For each success metric, capture:**
```yaml
metric:
  id: "string"                               # M-001, M-002, etc.
  name: "string"                             # REQUIRED
  category: "efficiency | accuracy | cost | reliability"
  current_baseline: "string"                 # Current state OR "unknown"
  target: "string"                           # Desired state OR "to be defined"
  how_measured: "string"                     # How to measure (or "to be defined")
  evidence_type: "explicit | derived"
  evidence_citation: "string"                # See Citation Requirements below
  confidence: "high | medium | low"
  confidence_reasoning: "string"             # NEW: Why this confidence level?
  priority: "primary | secondary"
```

---

### Directive 3: Synthesis & Handoff (10 minutes)

**Objective:** Organize findings for Phase 2 consumption.

**Process:**
```
Step 3.1: Compile Findings (3 min)
├── Count: Explicit vs derived items
├── Calculate: Overall confidence score
├── Organize: Pain points by priority (if any identified)
└── Ensure: All required fields populated

Step 3.2: Identify Gaps (4 min)
├── List: What information is missing?
├── Generate: Clarifying questions for critical gaps
├── Assess: Can Phase 2 proceed with current info?
└── Determine: Escalate or proceed with gaps

Step 3.3: Prepare Handoff (3 min)
├── Summarize: Key findings for Phase 2
├── Identify: Automation opportunities (if clear)
├── Flag: Risks and concerns
└── State: What Phase 2 needs to decide or clarify
```

---

## Citation Requirements (NEW in v1.3)

**Citations must be faithful to the input, not generic domain knowledge.**

### For EXPLICIT Items

```yaml
requirement: "Must quote or closely paraphrase actual text from task_context"
format: "Source: [field_name] — '[quoted text]'"
example:
  good: "Source: original_description — 'we manually export new emails each week'"
  bad: "Task mentions manual export"  # Too vague, no quote
```

### For DERIVED Items

```yaml
requirement: "Must reference specific input elements that support the derivation"
format: "Derived from [field_name]: '[relevant text]' → [logical step]"

ALLOWED derivations:
  - Logical necessity: "Derived from original_description: 'email capture on Shopify' → emails must be stored somewhere"
  - Stated implication: "Derived from original_description: 'manual weekly export' → implies delay between capture and availability"
  
BANNED derivations (unless marked LOW confidence):
  - Generic patterns: "Manual CSV workflows commonly have errors"
  - Industry norms: "This is typical for e-commerce"
  - Assumed behavior: "Users usually expect..."
  
If using generic pattern:
  - MUST mark confidence as "low"
  - MUST include confidence_reasoning: "Based on generic pattern, not input-specific evidence"
```

### Citation Validation

Before finalizing output, verify each citation:
1. Does it reference a specific field from task_context?
2. Can the quoted text actually be found in the input?
3. Does the derivation logically follow from the cited text?

If any answer is "no," either fix the citation or downgrade to low confidence.

---

## Confidence Assignment Protocol (NEW in v1.3)

**Confidence scores must be reasoned, not guessed.**

### Before Assigning Confidence

For each derived item, answer these questions:

```yaml
confidence_checklist:
  q1: "Does my citation reference specific text from the input?"
  q2: "Would someone else reading the input reach the same conclusion?"
  q3: "Am I relying on generic domain knowledge or input-specific evidence?"
  q4: "If this derivation is wrong, how would I know?"
```

### Confidence Levels (Revised)

```yaml
HIGH:
  criteria:
    - Citation references specific input text
    - Derivation is logically necessary (not just plausible)
    - Another analyst would reach the same conclusion
  example: "'email capture on Shopify' necessarily implies email storage"
  confidence_reasoning: "Logically necessary — cannot capture without storing"

MEDIUM:
  criteria:
    - Citation references specific input text
    - Derivation is reasonable but not certain
    - Alternative interpretations exist but are less likely
  example: "'manual export' likely implies some delay, though frequency unknown"
  confidence_reasoning: "Reasonable inference from 'manual' — but frequency not stated"

LOW:
  criteria:
    - Derivation relies on generic patterns, not input
    - OR citation is weak/indirect
    - OR significant uncertainty about interpretation
  example: "CSV workflows often have errors (industry pattern)"
  confidence_reasoning: "Generic pattern — task doesn't mention errors specifically"
```

### Conservative Labeling Rule

**When uncertain between two confidence levels, choose the lower one.**

Rationale: Overconfident outputs mislead Phase 2. Underconfident outputs trigger appropriate caution.

---

## Information Classification Protocol

**v1.3 uses TWO categories with mandatory reasoning.**

### Classification Levels

```yaml
EXPLICIT:
  definition: "Directly stated in task description"
  confidence: "high (always)"
  citation_requirement: "MUST quote source field and text"
  confidence_reasoning: "Required: 'Directly stated in [field]'"

DERIVED:
  definition: "Not directly stated; logically follows from input"
  confidence: "varies — must be reasoned per item"
  citation_requirement: "MUST reference input elements supporting derivation"
  confidence_reasoning: "Required: explain why this confidence level"
  
  BANNED_PATTERNS:
    - "commonly" without low confidence
    - "typically" without low confidence
    - "usually" without low confidence
    - "industry standard" without low confidence
```

### Why Mandatory Reasoning?

Research shows LLMs miscalibrate self-reported confidence. Requiring explicit reasoning:
- Forces consideration of evidence quality
- Makes weak derivations visible
- Enables downstream review of confidence decisions

---

## Output Schema

Your output MUST conform to this schema. Maximum 3 levels of nesting.

```yaml
workflow_research_output:
  # Level 1: Metadata
  metadata:
    run_id: "string"
    phase: "phase_1_discovery"
    track: "workflow"
    timestamp: "ISO-8601"
    duration_minutes: "integer"
    input_richness: "rich | moderate | sparse"
    explicit_item_count: "integer"
    derived_item_count: "integer"
    overall_confidence: "high | medium | low"
    
  # Level 1: Summary
  summary:
    workflow_name: "string"
    domain: "string"
    complexity: "simple | moderate | complex | unknown"
    automation_readiness: "ready | needs_clarification | insufficient_information"
    key_insight: "string (one sentence)"
    
  # Level 1: Current State
  current_state:
    workflow_name: "string"
    trigger: "event | schedule | manual | unknown"
    automation_level: "manual | semi_automated | fully_automated | unknown"
    steps:
      - number: "integer"
        description: "string"
        actor: "human | system | unknown"
        tool: "string or null"
        evidence_type: "explicit | derived"
        evidence_citation: "string"
        confidence: "high | medium | low"
        confidence_reasoning: "string"           # NEW in v1.3
    existing_tools: ["string"]
    outputs: ["string"]
    
  # Level 1: Pain Points (may be empty array if none identified)
  pain_points:
    - id: "string"
      description: "string"
      category: "string"
      severity: "critical | high | medium | low"
      evidence_type: "explicit | derived"
      evidence_citation: "string"
      confidence: "high | medium | low"
      confidence_reasoning: "string"             # NEW in v1.3
      addressable_by_automation: "yes | partially | no | unknown"
      
  # Level 1: Stakeholders (may be empty array if none identified)
  stakeholders:
    - id: "string"
      role: "string"
      type: "initiator | performer | recipient | approver | affected"
      needs: ["string"]
      evidence_type: "explicit | derived"
      evidence_citation: "string"
      confidence: "high | medium | low"
      confidence_reasoning: "string"             # NEW in v1.3
      primary_user: "boolean"
      
  # Level 1: Success Metrics (may be empty array if none identified)
  success_metrics:
    - id: "string"
      name: "string"
      category: "efficiency | accuracy | cost | reliability"
      current_baseline: "string"
      target: "string"
      how_measured: "string"
      evidence_type: "explicit | derived"
      evidence_citation: "string"
      confidence: "high | medium | low"
      confidence_reasoning: "string"             # NEW in v1.3
      priority: "primary | secondary"
      
  # Level 1: Information Gaps (REQUIRED — at least document what's unknown)
  information_gaps:
    - gap: "string"
      impact: "HIGH | MEDIUM | LOW"
      clarifying_question: "string"
      blocks_phase_2: "boolean"
      
  # Level 1: Recommendations for Phase 2
  phase_2_handoff:
    automation_opportunities:
      - opportunity: "string"
        pain_points_addressed: ["pain point IDs"]
        complexity: "low | medium | high"
        confidence: "high | medium | low"
    automation_risks:
      - risk: "string"
        likelihood: "high | medium | low"
        mitigation: "string"
    key_decisions_needed: ["string"]
    questions_requiring_human_input: ["string"]
    proceed_recommendation: "proceed | proceed_with_gaps | escalate"
```

**Schema Rules:**
- Arrays MAY be empty if no items can be identified with confidence
- Every EXPLICIT item MUST have evidence_citation with quoted text and source field
- Every DERIVED item MUST have evidence_citation referencing input elements
- Every item MUST have confidence_reasoning explaining the confidence level
- information_gaps is REQUIRED and must not be empty

---

## Quality Gate

### Gate Process (Adapts to Input Richness)

```yaml
gate_process:
  step_1_schema_validation:
    action: "Validate output against schema"
    check: "All required fields present and correct types"
    on_fail: "Return to Directive 3, fix schema issues, retry (max 2 retries)"
    
  step_2_citation_faithfulness:                  # ENHANCED in v1.3
    action: "Verify citations are faithful to input"
    checks:
      - "Every EXPLICIT citation quotes actual input text with source field"
      - "Every DERIVED citation references specific input elements"
      - "No generic pattern citations marked high/medium confidence"
    on_fail: "Fix citations OR downgrade confidence to low"
    
  step_3_confidence_reasoning:                   # NEW in v1.3
    action: "Verify confidence reasoning exists"
    check: "Every item has confidence_reasoning field populated"
    on_fail: "Add reasoning; if reasoning is weak, downgrade confidence"
    
  step_4_confidence_distribution:
    action: "Assess overall confidence"
    compute: "Count derived items by confidence level"
    threshold: "If ≥61% of derived items are low-confidence, flag"  # CLARIFIED in v1.3
    small_sample_rule: "If <3 derived items, skip threshold check"  # NEW in v1.3
    on_warning: "Note in summary, may affect proceed_recommendation"
    
  step_5_quality_assessment:
    action: "Evaluate against input-appropriate criteria"
    criteria_by_richness:
      rich_input:
        - current_state: "Trigger identified + ≥2 steps with faithful citations"
        - pain_points: "≥1 pain point with evidence"
        - stakeholders: "≥1 stakeholder identified"
        - success_metrics: "≥1 metric defined"
      moderate_input:
        - current_state: "Trigger identified OR ≥1 step with evidence"
        - pain_points: "≥1 pain point OR documented 'none identified'"
        - stakeholders: "≥1 stakeholder OR documented 'none identified'"
        - success_metrics: "Optional; gaps documented if missing"
        - explicit_minimum: "≥2 explicit items across different sections"  # NEW in v1.3
      sparse_input:
        - current_state: "Best effort with available information"
        - pain_points: "Optional; gaps documented"
        - stakeholders: "Optional; gaps documented"
        - success_metrics: "Optional; gaps documented"
        - required: "≥3 clarifying questions in information_gaps"
    on_fail: "Iterate on weak areas OR escalate if sparse"
    
  step_6_proceed_decision:
    action: "Determine recommendation for Phase 2"
    
    # CRITICAL: Sparse input rules take precedence (v1.3 clarification)
    sparse_override: |
      IF input_richness = "sparse":
        proceed_recommendation = "escalate"  # ALWAYS, regardless of other factors
    
    moderate_rules: |                            # TIGHTENED in v1.3
      IF input_richness = "moderate":
        IF explicit_item_count < 2:
          proceed_recommendation = "escalate"
        ELSE IF stakeholders = [] AND success_metrics = []:
          proceed_recommendation = "escalate"    # NEW: can't proceed without either
        ELSE:
          proceed_recommendation = "proceed_with_gaps"
    
    rich_rules: |
      IF input_richness = "rich":
        IF overall_confidence = "low" OR >2 blocking_gaps:
          proceed_recommendation = "proceed_with_gaps"
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
Citations Faithful? ─── NO ──→ Fix citations or downgrade confidence ──→ Retry
  │
  YES
  │
  ▼
Confidence Reasoning Present? ─── NO ──→ Add reasoning ──→ Retry
  │
  YES
  │
  ▼
Input Richness? ──┬── SPARSE ──→ ≥3 clarifying questions? ─── NO ──→ Add questions
                  │                      │
                  │                     YES
                  │                      │
                  │                      ▼
                  │              ESCALATE (always for sparse)
                  │
                  ├── MODERATE ──→ ≥2 explicit items? ─── NO ──→ ESCALATE
                  │                      │
                  │                     YES
                  │                      │
                  │                      ▼
                  │              Both stakeholders AND metrics empty? ─── YES ──→ ESCALATE
                  │                      │
                  │                      NO
                  │                      │
                  │                      ▼
                  │              PASS (proceed_with_gaps)
                  │
                  └── RICH ──→ Quality criteria met? ─── NO ──→ ITERATE
                                       │
                                      YES
                                       │
                                       ▼
                               ≥3 derived items? ─── NO ──→ PASS (skip threshold)
                                       │
                                      YES
                                       │
                                       ▼
                               ≥61% low confidence? ─── YES ──→ PASS (proceed_with_gaps)
                                       │
                                      NO
                                       │
                                       ▼
                                   PASS (proceed)
```

---

## Sparse Input Mode

**When input_richness = "sparse", special rules apply.**

**CRITICAL: Sparse input rules ALWAYS override proceed_decision logic. Sparse input ALWAYS escalates.**

### What Triggers Sparse Input Mode

Input is sparse if ≤1 of these is true:
- Task description contains workflow trigger
- Task description mentions process steps
- Task description states problems
- Task description mentions stakeholders
- Business context is provided

### Sparse Input Behavior

```yaml
sparse_input_rules:
  PRECEDENCE: "These rules override ALL other proceed_decision logic"  # v1.3 clarification
  
  DO:
    - Extract every explicit detail, however minimal
    - Note workflow_name as "Unknown" or best guess with low confidence
    - Set trigger to "unknown" if not stated
    - Document ≥3 clarifying questions in information_gaps
    - Set proceed_recommendation to "escalate"
    - Set automation_readiness to "insufficient_information"
    
  DO_NOT:
    - Fabricate workflow steps to fill gaps
    - Invent stakeholders not implied by task
    - Create pain points from generic domain research
    - Assume success metrics without any basis
    - Use generic pattern citations
    
  OUTPUT_REQUIREMENTS:
    - pain_points: May be empty array []
    - stakeholders: May be empty array []
    - success_metrics: May be empty array []
    - information_gaps: MUST have ≥3 questions
    - phase_2_handoff.proceed_recommendation: MUST be "escalate" (no exceptions)
```

### Sparse Input Example

```yaml
# Input: "Automate something with Shopify"

workflow_research_output:
  metadata:
    run_id: "run-sparse-001"
    phase: "phase_1_discovery"
    track: "workflow"
    timestamp: "2025-12-05T16:00:00Z"
    duration_minutes: 15
    input_richness: "sparse"
    explicit_item_count: 1
    derived_item_count: 0
    overall_confidence: "low"
    
  summary:
    workflow_name: "Unknown Shopify workflow"
    domain: "e-commerce"
    complexity: "unknown"
    automation_readiness: "insufficient_information"
    key_insight: "Task description lacks workflow details; clarification required"
    
  current_state:
    workflow_name: "Unknown"
    trigger: "unknown"
    automation_level: "unknown"
    steps: []  # Empty — cannot fabricate
    existing_tools: ["Shopify"]
    outputs: []
    
  pain_points: []  # Empty — no evidence to derive pain points
  
  stakeholders: []  # Empty — no roles mentioned
  
  success_metrics: []  # Empty — no criteria stated
  
  information_gaps:
    - gap: "What specific workflow or process needs automation?"
      impact: "HIGH"
      clarifying_question: "Can you describe the current manual process step by step?"
      blocks_phase_2: true
    - gap: "What problem is this automation solving?"
      impact: "HIGH"
      clarifying_question: "What pain points or inefficiencies are you experiencing?"
      blocks_phase_2: true
    - gap: "Who performs this workflow currently?"
      impact: "MEDIUM"
      clarifying_question: "Which team members or roles are involved in this process?"
      blocks_phase_2: false
    - gap: "What does success look like?"
      impact: "MEDIUM"
      clarifying_question: "How would you measure if this automation is working well?"
      blocks_phase_2: false
      
  phase_2_handoff:
    automation_opportunities: []
    automation_risks:
      - risk: "Automation scope unclear"
        likelihood: "high"
        mitigation: "Obtain workflow details before proceeding"
    key_decisions_needed:
      - "Define the actual workflow to automate"
    questions_requiring_human_input:
      - "What specific Shopify workflow needs automation?"
      - "What is the current process and what's broken?"
    proceed_recommendation: "escalate"  # ALWAYS for sparse input
```

---

## Failure Modes & Guards

### 1. Fabricated Workflows
**Problem:** Inventing detailed workflows not supported by input.
**Guard:** Every step requires faithful evidence_citation.
**Enforcement:** Citations must reference specific input text. Generic patterns = low confidence only.

### 2. Confidence Inflation
**Problem:** Marking low-confidence items as high-confidence.
**Guard:** Mandatory confidence_reasoning field.
**Enforcement:** Reasoning reviewed in gate step 3. Weak reasoning = downgrade confidence.

### 3. Unfaithful Citations
**Problem:** Citations that don't actually support the claim.
**Guard:** Citation faithfulness check in gate step 2.
**Enforcement:** Must quote actual input text (EXPLICIT) or reference specific elements (DERIVED).

### 4. Generic Pain Points
**Problem:** Listing problems from domain knowledge, not this workflow.
**Guard:** Generic patterns ("commonly," "typically") banned at high/medium confidence.
**Enforcement:** Must be marked low confidence with appropriate reasoning.

### 5. Forced Fabrication
**Problem:** Gate requirements forcing invention to meet minimums.
**Guard:** Requirements adapt to input richness; empty arrays allowed.
**Enforcement:** Sparse = escalate always. Moderate = escalate if <2 explicit items.

### 6. Hidden Low-Confidence
**Problem:** Proceeding confidently when most content is uncertain.
**Guard:** Confidence distribution check with ≥61% threshold.
**Enforcement:** Skipped for <3 items; triggers proceed_with_gaps otherwise.

### 7. Sparse Input Override Failure
**Problem:** Sparse input not escalating due to logic conflicts.
**Guard:** Explicit precedence rule in gate process.
**Enforcement:** Sparse input rules override ALL other proceed_decision logic.

---

## Time Budget

**Total: 45 minutes**

| Phase | Time | Focus |
|-------|------|-------|
| 0. Input Assessment | 2 min | Assess richness, determine mode |
| 1. Current State & Pain Points | 18 min | Workflow, problems, readiness |
| 2. Stakeholders & Success Metrics | 15 min | Who cares, how to measure |
| 3. Synthesis & Handoff | 10 min | Compile, gaps, prepare for Phase 2 |

**Time Management Rules:**
- If input is sparse, spend more time on clarifying questions, less on derivation
- Reserve final 10 min for Directive 3 (non-negotiable)
- If total approaches 45 min: Complete current step and synthesize

---

## Example Output (Rich Input)

```yaml
workflow_research_output:
  metadata:
    run_id: "run-rich-001"
    phase: "phase_1_discovery"
    track: "workflow"
    timestamp: "2025-12-05T15:30:00Z"
    duration_minutes: 38
    input_richness: "rich"
    explicit_item_count: 4
    derived_item_count: 5
    overall_confidence: "medium"
    
  summary:
    workflow_name: "Email capture and sync to Klaviyo"
    domain: "e-commerce"
    complexity: "simple"
    automation_readiness: "ready"
    key_insight: "Manual weekly email export to Klaviyo causes campaign delays"
    
  current_state:
    workflow_name: "Email capture and sync to Klaviyo"
    trigger: "manual"
    automation_level: "semi_automated"
    steps:
      - number: 1
        description: "Customer enters email on Shopify checkout or popup"
        actor: "human"
        tool: "Shopify"
        evidence_type: "derived"
        evidence_citation: "Derived from original_description: 'email capture on Shopify' → customer must enter email somewhere"
        confidence: "high"
        confidence_reasoning: "Logically necessary — cannot capture email without entry point"
      - number: 2
        description: "Email stored in Shopify customer database"
        actor: "system"
        tool: "Shopify"
        evidence_type: "derived"
        evidence_citation: "Derived from original_description: 'email capture on Shopify' → Shopify stores customer data automatically"
        confidence: "high"
        confidence_reasoning: "Logically necessary — captured data must be stored"
      - number: 3
        description: "Weekly manual export of new customers from Shopify"
        actor: "human"
        tool: "Shopify Admin"
        evidence_type: "explicit"
        evidence_citation: "Source: original_description — 'we manually export new emails each week'"
        confidence: "high"
        confidence_reasoning: "Directly stated in input"
      - number: 4
        description: "CSV upload to Klaviyo"
        actor: "human"
        tool: "Klaviyo"
        evidence_type: "derived"
        evidence_citation: "Derived from original_description: 'sync to Klaviyo' + 'manual export' → manual sync typically uses CSV"
        confidence: "medium"
        confidence_reasoning: "Reasonable inference — CSV is common for manual sync, but method not explicitly stated"
    existing_tools: ["Shopify", "Klaviyo"]
    outputs: ["Customer profile in Klaviyo"]
    
  pain_points:
    - id: "PP-001"
      description: "Weekly manual sync causes 1-7 day delay in email campaigns"
      category: "speed"
      severity: "high"
      evidence_type: "explicit"
      evidence_citation: "Source: original_description — 'delays our welcome emails by up to a week'"
      confidence: "high"
      confidence_reasoning: "Directly stated in input"
      addressable_by_automation: "yes"
      
  stakeholders:
    - id: "S-001"
      role: "Marketing Manager"
      type: "initiator"
      needs: ["Timely customer data in Klaviyo", "Reduced manual work"]
      evidence_type: "explicit"
      evidence_citation: "Source: original_description — 'our marketing team exports'"
      confidence: "high"
      confidence_reasoning: "Directly stated in input"
      primary_user: true
      
  success_metrics:
    - id: "M-001"
      name: "Sync delay"
      category: "efficiency"
      current_baseline: "1-7 days"
      target: "<1 hour"
      how_measured: "Time between Shopify signup and Klaviyo profile creation"
      evidence_type: "derived"
      evidence_citation: "Derived from original_description: 'delays by up to a week' → baseline is 1-7 days"
      confidence: "medium"
      confidence_reasoning: "Baseline derived from stated delay; target is reasonable goal but not specified"
      priority: "primary"
      
  information_gaps:
    - gap: "Exact current sync frequency"
      impact: "LOW"
      clarifying_question: "Is the export exactly weekly, or does it vary?"
      blocks_phase_2: false
    - gap: "Volume of new signups"
      impact: "LOW"
      clarifying_question: "Approximately how many new email signups per week?"
      blocks_phase_2: false
      
  phase_2_handoff:
    automation_opportunities:
      - opportunity: "Real-time sync via Shopify webhooks to Klaviyo API"
        pain_points_addressed: ["PP-001"]
        complexity: "low"
        confidence: "high"
      - opportunity: "Scheduled automated sync (hourly/daily)"
        pain_points_addressed: ["PP-001"]
        complexity: "low"
        confidence: "high"
    automation_risks:
      - risk: "Duplicate profiles in Klaviyo if sync logic incorrect"
        likelihood: "medium"
        mitigation: "Use Shopify customer ID as external identifier"
    key_decisions_needed:
      - "Real-time (webhook) vs scheduled sync"
      - "How to handle existing customers already in Klaviyo"
    questions_requiring_human_input:
      - "Is there an existing Klaviyo integration that needs to be replaced?"
    proceed_recommendation: "proceed"
```

---

## Handoff to Phase 2

Phase 2 (Synthesis V1) will receive this output and use it to:
1. Understand what problem needs solving
2. Design automation that addresses prioritized pain points
3. Account for stakeholder needs
4. Define success criteria based on metrics
5. Investigate flagged gaps if critical

**Your job is complete when:**
- Schema is valid with all citations faithful to input
- Confidence reasoning is present for all items
- Input richness is assessed and recorded
- Current state is documented with evidence types and confidence
- Pain points, stakeholders, metrics have evidence (or arrays are empty with gaps documented)
- Information gaps have clarifying questions
- phase_2_handoff.proceed_recommendation reflects actual confidence and follows gate rules

You are not designing. You are diagnosing — and honestly reporting what you know vs. don't know.

---

*End of Meta-Prompt v1.3*
