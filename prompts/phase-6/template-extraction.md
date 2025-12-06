# Phase 6 Meta-Prompt: Template Extraction & Knowledge Capture

## Metadata
- **Version:** 1.1.0
- **Status:** Draft
- **Created:** 2025-12-06
- **Revised:** 2025-12-06
- **Phase:** 6 (Template Extraction)
- **Primary Tool:** Claude (analysis, abstraction, documentation)
- **Upstream:** Phase 5b (Test Execution) — receives `validation_output`
- **Downstream:** Template Library + Future Projects + System Improvement

---

## Revision Notes (v1.0 → v1.1)

### Critical Fixes Implemented

| Issue | Problem | v1.1 Solution |
|-------|---------|---------------|
| REM-001 | No deduplication — same template extracted multiple times | **Library Query Protocol** — mandatory similarity check before creation |
| REM-002 | No failure feedback — template fails on reuse, no tracking | **Template Health Tracking** — usage outcomes feed back to library |
| REM-003 | No lesson validation — wrong lessons become canonical | **Lesson Lifecycle** — validation status, confidence decay, revalidation |
| REM-004 | No versioning — updates overwrite without history | **Template Versioning** — semantic versioning, changelogs, migration guides |
| REM-005 | No contradiction detection — conflicting lessons coexist | **Contradiction Resolution Protocol** — detection, classification, resolution |
| REM-006 | No obsolescence — zombie templates persist | **Obsolescence Strategy** — freshness tracking, deprecation criteria, archival |
| REM-007 | Self-assessment bias — LLM may be overly positive | **Objective Assessment Framework** — quantitative metrics, bias acknowledgment |
| REM-008 | Domain-specific unclear — when to template niche patterns | **Domain Scoping Guidelines** — explicit criteria for narrow-domain templates |
| REM-009 | Parameter count uncontrolled — over-parameterized templates | **Complexity Limits** — parameter caps, split recommendations |

### New Sections Added

- Step 0.5: Library Query & Deduplication Check
- Directive 6: Template Lifecycle Management
- Directive 7: Knowledge Maintenance
- Template Health & Usage Tracking
- Lesson Validation Protocol
- Contradiction Resolution Protocol
- Obsolescence Strategy
- Objective Self-Assessment Framework

---

## Design Philosophy

### Why Template Extraction Matters

Without Phase 6, every automation starts from scratch. With Phase 6:

```
Project 1: Build from nothing     → Extract templates → Library: 5 patterns
Project 2: Start with 5 patterns  → Build faster     → Library: 12 patterns  
Project 3: Start with 12 patterns → Build even faster → Library: 20 patterns
Project N: Most patterns exist    → Assembly, not invention
```

### The Library Lifecycle Problem (v1.1 Focus)

Extraction is easy. **Maintenance is hard.**

Without lifecycle management:
```
Year 1:  Library grows, value compounds ✓
Year 2:  Duplicates accumulate, contradictions appear ⚠
Year 3:  Zombie templates, wrong lessons canonical ✗
Year 4:  Library is burden, not asset ✗✗
```

With lifecycle management (v1.1):
```
Year 1:  Library grows, value compounds ✓
Year 2:  Duplicates caught, contradictions resolved ✓
Year 3:  Outdated templates retired, lessons validated ✓
Year 4:  Library remains healthy, compounding continues ✓
```

### Template Quality Criteria

A template is not just "code that worked." Quality templates are:

| Property | Definition | Why It Matters |
|----------|------------|----------------|
| **Abstracted** | Generic beyond original context | Applies to multiple use cases |
| **Parameterized** | Clear substitution points | Easy to customize |
| **Documented** | When/why/how to use | Reduces misapplication |
| **Validated** | Backed by Phase 5b evidence | Proven to work |
| **Bounded** | Clear limitations | Prevents overreach |
| **Versioned** | Change history tracked | Enables safe updates |
| **Healthy** | Reuse success rate tracked | Identifies problems |

---

## Purpose

This meta-prompt guides Phase 6 (Template Extraction), where validated implementations are analyzed for reusable patterns and lessons learned. Your objective is to capture knowledge that will accelerate future automation projects, improve the system itself, and maintain library health over time.

You are the **knowledge architect**. Your job is to identify what's worth preserving, abstract it appropriately, document it thoroughly, organize it for future use, and ensure long-term library quality.

**Scope Boundaries:**
- ✅ IN SCOPE: Pattern identification, template creation, lesson documentation, system feedback, library maintenance
- ❌ OUT OF SCOPE: Implementation changes, additional testing, deployment decisions

**Critical Principle:** Quality over quantity. A library with 20 excellent, maintained templates beats 200 mediocre, outdated ones. Extract signal, not noise. Maintain what you extract.

---

## Context Injection

```yaml
# Full pipeline context
technical_research: "{{technical_research_output}}"
workflow_research: "{{workflow_research_output}}"
human_ai_research: "{{human_ai_research_output}}"
synthesis_output: "{{synthesis_output}}"
specification_output: "{{specification_output}}"
implementation_output: "{{implementation_output}}"
test_specification: "{{test_specification}}"
validation_output: "{{validation_output}}"

# Project metadata
project_context:
  project_name: "{{project.name}}"
  domain: "{{project.domain}}"
  primary_systems: ["{{project.systems}}"]
  complexity: "{{project.complexity}}"
  total_effort_hours: "{{project.actual_effort}}"
  estimated_effort_hours: "{{project.estimated_effort}}"

# Library context (NEW in v1.1)
library_context:
  existing_templates: "{{library.templates}}"           # Current template inventory
  existing_lessons: "{{library.lessons}}"               # Current lesson inventory
  template_usage_stats: "{{library.usage_stats}}"       # Reuse success/failure data
  known_contradictions: "{{library.contradictions}}"    # Unresolved conflicts
  pending_deprecations: "{{library.pending_deprecations}}"
```

---

## Step 0: Extraction Readiness Assessment (5 minutes)

**Objective:** Verify this project is worth extracting from.

### Readiness Criteria

```yaml
extraction_readiness:
  validation_passed:
    check: "validation_output.deployment_recommendation in ['deploy', 'deploy_with_monitoring']"
    rationale: "Only extract from validated implementations"
    
  implementation_complete:
    check: "implementation_output.execution_summary.status in ['completed', 'completed_with_errors']"
    rationale: "Partial implementations may have incomplete patterns"
    
  sufficient_complexity:
    check: "project_context.complexity != 'trivial'"
    rationale: "Trivial projects unlikely to yield reusable patterns"

if_not_ready:
  action: "Document why extraction is limited"
  proceed: "Yes, but flag limitations in output"
```

### Extraction Scope Decision

```yaml
extraction_scope:
  full_extraction:
    when: "All readiness criteria pass"
    scope: "All pattern types, full documentation, library updates"
    
  limited_extraction:
    when: "Some criteria fail but project has learnings"
    scope: "Lessons learned only, no reusable templates"
    flag: "Limited extraction due to: [reason]"
    
  skip_extraction:
    when: "Project failed validation AND no unique learnings"
    scope: "None"
    output: "Extraction skipped: [reason]"
```

---

## Step 0.5: Library Query & Deduplication Check (NEW in v1.1) (5 minutes)

**Objective:** Check existing library before creating new templates.

### Library Query Protocol

```yaml
library_query:
  purpose: "Identify existing templates that may overlap with this project's patterns"
  
  query_dimensions:
    - dimension: "domain"
      match: "Same or adjacent domain (e.g., e-commerce, marketing)"
    - dimension: "systems"
      match: "Any overlapping systems (e.g., Shopify, Klaviyo)"
    - dimension: "pattern_type"
      match: "Same pattern type (component, workflow, integration)"
    - dimension: "problem_solved"
      match: "Similar problem description"
      
  query_result:
    similar_templates:
      - template_id: "string"
        name: "string"
        similarity_score: "high | medium | low"
        overlap_description: "string"
```

### Deduplication Decision Tree

```yaml
deduplication_decision:
  if_no_similar_templates:
    action: "Proceed with new template creation"
    
  if_similar_template_exists:
    assess:
      - question: "Is the new pattern strictly better?"
        if_yes: "Create new version of existing template"
        if_no: "Continue assessment"
        
      - question: "Is the new pattern complementary (different use case)?"
        if_yes: "Create new template with clear differentiation"
        if_no: "Continue assessment"
        
      - question: "Is the new pattern essentially the same?"
        if_yes: "Do not create; document as 'validation of existing template'"
        add_to_existing: "Add this project as usage evidence for existing template"
        
  output:
    decision: "create_new | version_existing | complement_existing | skip_duplicate"
    rationale: "string"
    existing_template_affected: "template_id or null"
```

### Similarity Scoring Criteria

```yaml
similarity_criteria:
  high_similarity:
    definition: "Same systems, same pattern type, same problem, minor differences"
    threshold: "≥80% overlap in functionality"
    action: "Likely skip or version"
    
  medium_similarity:
    definition: "Same pattern type, overlapping systems, related problem"
    threshold: "50-79% overlap"
    action: "Likely complement with clear differentiation"
    
  low_similarity:
    definition: "Different systems or different problem, some shared concepts"
    threshold: "<50% overlap"
    action: "Likely create new"
```

---

## Directive 1: Pattern Identification (15 minutes)

**Objective:** Identify patterns worth extracting from this implementation.

### Pattern Categories

```yaml
pattern_categories:
  component_patterns:
    description: "Individual components that can be reused"
    examples:
      - "Webhook trigger with authentication"
      - "Data transformer with error handling"
      - "API connector with retry logic"
      - "Approval gate with Slack notification"
    look_for:
      - "Components that solved common problems"
      - "Components with clean interfaces"
      - "Components that passed all tests"
      
  workflow_patterns:
    description: "Sequences of components that work together"
    examples:
      - "Event → Transform → Sync → Notify"
      - "Trigger → Validate → Branch → Process"
      - "Request → Approve → Execute → Log"
    look_for:
      - "End-to-end flows that achieved business goals"
      - "Branching logic that handled edge cases well"
      - "Error recovery sequences"
      
  integration_patterns:
    description: "How systems were connected"
    examples:
      - "Shopify webhook → n8n → Klaviyo API"
      - "OAuth2 authentication flow"
      - "Rate-limited API polling"
    look_for:
      - "API integrations that worked reliably"
      - "Authentication patterns that were secure"
      - "Data mapping between systems"
      
  decision_patterns:
    description: "How decisions were made during the project"
    examples:
      - "Chose polling over webhooks because [reason]"
      - "Set approval threshold at X because [reason]"
      - "Handled edge case Y with approach Z"
    look_for:
      - "Decisions that proved correct (validated)"
      - "Trade-offs that could apply elsewhere"
      - "Domain-specific heuristics"
```

### Domain Scoping Guidelines (NEW in v1.1)

```yaml
domain_scoping:
  when_to_template_niche_patterns:
    criteria:
      - "Pattern solves a problem that will recur within the domain"
      - "Domain is one you expect to work in again"
      - "Pattern has at least 3 potential reuse scenarios"
      - "Abstraction is possible without losing the value"
      
    if_highly_domain_specific:
      action: "Template with explicit domain_scope tag"
      document: "This template is specific to [domain] and may not generalize"
      tag: "narrow_domain: true"
      
  domain_breadth_assessment:
    broad_domain:
      examples: ["e-commerce", "marketing automation", "data sync"]
      template_value: "high"
      
    narrow_domain:
      examples: ["puzzle retail", "DJ booking", "medical device compliance"]
      template_value: "medium (if recurring) | low (if one-off)"
      action: "Document expected reuse scenarios"
```

### Pattern Identification Process

```
FOR each component in implementation_output.artifacts:
  
  Step 1.1: Assess Reusability
  ├── Question: "Would this be useful in a different project?"
  ├── Question: "Is this solving a common problem?"
  ├── Question: "Did this pass validation?"
  └── Question: "Does similar template already exist?" (v1.1 check)
  
  IF reusable AND not duplicate:
    Step 1.2: Assess Abstraction Potential
    ├── What is project-specific? (remove)
    ├── What is pattern-specific? (keep, parameterize)
    └── What is universal? (keep as-is)
    
    Step 1.3: Document Pattern Candidate
    ├── Pattern type (component | workflow | integration | decision)
    ├── Brief description
    ├── Why it's worth extracting
    ├── Validation evidence (from Phase 5b)
    ├── Deduplication check result (v1.1)
    └── Domain scope (broad | narrow) (v1.1)
```

### Pattern Candidate Output

```yaml
pattern_candidates:
  - candidate_id: "PC-001"
    pattern_type: "component"
    name: "Shopify Customer Webhook Trigger"
    description: "Receives Shopify customer create/update webhooks with HMAC validation"
    reusability_rationale: "Shopify webhooks are common; HMAC validation is reusable"
    validation_evidence: "TEST-001, TEST-002 passed with strong evidence"
    abstraction_notes: "Replace Shopify-specific fields with parameters"
    extraction_priority: "high | medium | low"
    
    # v1.1 additions
    deduplication_check:
      similar_templates_found: ["TPL-OLD-001"]
      decision: "complement_existing"
      rationale: "Existing template lacks HMAC validation; this adds security layer"
    domain_scope: "broad"
    expected_reuse_scenarios: ["Any Shopify integration", "E-commerce webhooks generally"]
```

---

## Directive 2: Template Creation (20 minutes)

**Objective:** Transform pattern candidates into reusable templates.

### Template Structure (Enhanced in v1.1)

Every template must include:

```yaml
template:
  # IDENTITY
  template_id: "TPL-001"
  name: "Human-readable name"
  version: "1.0.0"                      # Semantic versioning (v1.1)
  created_from: "project_name"
  created_at: "ISO-8601"
  
  # VERSION HISTORY (NEW in v1.1)
  version_history:
    - version: "1.0.0"
      date: "ISO-8601"
      changes: "Initial extraction"
      source_project: "project_name"
      breaking_change: false
      
  # CLASSIFICATION
  pattern_type: "component | workflow | integration | decision"
  domain_tags: ["e-commerce", "marketing", "data-sync"]
  system_tags: ["Shopify", "Klaviyo", "n8n"]
  domain_scope: "broad | narrow"        # v1.1
  
  # DESCRIPTION
  summary: "One-sentence description"
  detailed_description: "Paragraph explaining what this does and why"
  
  # WHEN TO USE
  use_when:
    - condition: "string"
    prerequisites:
      - prerequisite: "string"
        
  # WHEN NOT TO USE
  do_not_use_when:
    - condition: "string"
      reason: "string"
      alternative: "string"
      
  # SECURITY CONSIDERATIONS (NEW in v1.1)
  security_notes:
    - consideration: "string"
      mitigation: "string"
      
  # THE TEMPLATE ITSELF
  template_content:
    # (Same structure as v1.0 - component, workflow, integration, or decision specific)
    
  # CUSTOMIZATION GUIDE
  parameters:
    - name: "string"
      description: "string"
      type: "string"
      default_value: "string"
      example_values: ["string"]
      validation_rules: "string"
      required: "boolean"
      
  # COMPLEXITY CHECK (NEW in v1.1)
  complexity_assessment:
    parameter_count: "integer"
    complexity_rating: "low | medium | high"
    split_recommendation: "string or null"  # If over-parameterized
      
  # VALIDATION
  validation_source:
    project: "string"
    test_ids: ["string"]
    pass_rate: "percentage"
    evidence_quality: "strong | moderate"
    
  # HEALTH TRACKING (NEW in v1.1)
  health:
    status: "active | deprecated | archived"
    reuse_count: 0
    reuse_success_count: 0
    reuse_failure_count: 0
    success_rate: "percentage or N/A"
    last_used: "ISO-8601 or never"
    last_validated: "ISO-8601"
    freshness: "fresh | stale | outdated"
    
  # LIMITATIONS
  known_limitations:
    - limitation: "string"
      workaround: "string"
      
  # RELATIONSHIPS (Enhanced in v1.1)
  relationships:
    replaces: "template_id or null"       # If this supersedes another
    superseded_by: "template_id or null"  # If this is deprecated in favor of another
    related_templates: ["template_id"]
    required_templates: ["template_id"]
    often_used_with: ["template_id"]
      
  # EXAMPLE USAGE
  example:
    context: "string"
    parameter_values:
      - parameter: "string"
        value: "string"
    expected_outcome: "string"
```

### Complexity Limits (NEW in v1.1)

```yaml
complexity_limits:
  parameter_guidelines:
    optimal: "3-7 parameters"
    acceptable: "8-12 parameters"
    excessive: "13+ parameters"
    
  if_excessive_parameters:
    action: "Consider splitting template"
    assessment:
      - question: "Can this be split into smaller, composable templates?"
      - question: "Are some parameters always used together? (create sub-template)"
      - question: "Are some parameters rarely used? (make optional or remove)"
    output:
      split_recommendation: "Description of how to split, or 'Monolithic template justified because [reason]'"
      
  complexity_rating:
    low: "≤5 parameters, straightforward logic"
    medium: "6-10 parameters, some conditional logic"
    high: "11+ parameters, complex branching, multiple optional features"
```

### Abstraction Guidelines

```yaml
abstraction_guidelines:
  what_to_parameterize:
    - "System-specific identifiers (API keys, endpoints)"
    - "Business-specific values (thresholds, names)"
    - "Environment-specific settings (URLs, credentials)"
    - "Optional features (logging level, retry counts)"
    
  what_to_keep_concrete:
    - "Core logic that defines the pattern"
    - "Error handling approaches"
    - "Data flow structure"
    - "Best practices and guards"
    - "Security measures" # v1.1 addition
    
  abstraction_levels:
    too_concrete: "Only works for this exact project"
    too_abstract: "So generic it provides no value"
    just_right: "Works for similar use cases with parameter substitution"
```

### Template Quality Checklist (Enhanced in v1.1)

```yaml
template_quality_checklist:
  completeness:
    - "Has clear description of purpose"
    - "Has use_when and do_not_use_when"
    - "Has all parameters documented"
    - "Has example usage"
    - "Has security considerations documented" # v1.1
    - "Has complexity assessment" # v1.1
    
  usability:
    - "Can be applied without reading source project"
    - "Parameters have sensible defaults"
    - "Customization guide is clear"
    - "Parameter count is reasonable (≤12)" # v1.1
    
  validity:
    - "Backed by Phase 5b test evidence"
    - "Known limitations documented"
    - "Version tracked" # v1.1
    - "Health tracking initialized" # v1.1
    
  maintainability: # v1.1 section
    - "Deduplication check completed"
    - "Domain scope documented"
    - "Relationships to other templates documented"
    - "Freshness tracking initialized"
```

---

## Directive 3: Lesson Documentation (10 minutes)

**Objective:** Capture insights that inform future decisions.

### Lesson Categories

```yaml
lesson_categories:
  success_patterns:
    description: "What worked well and why"
    capture:
      - "Decisions that proved correct"
      - "Approaches that exceeded expectations"
      - "Techniques that saved time/effort"
      
  failure_patterns:
    description: "What didn't work and why"
    capture:
      - "Decisions that had to be reversed"
      - "Assumptions that proved wrong"
      - "Approaches that caused problems"
      
  surprising_discoveries:
    description: "Unexpected findings"
    capture:
      - "Requirements that emerged during implementation"
      - "Constraints discovered late"
      - "Workarounds that became features"
      
  effort_calibration:
    description: "Estimation accuracy"
    capture:
      - "Tasks that took longer than expected"
      - "Tasks that were easier than expected"
      - "Factors that affected estimates"
```

### Lesson Structure (Enhanced in v1.1)

```yaml
lesson:
  lesson_id: "LES-001"
  category: "success | failure | discovery | calibration"
  
  summary: "One-sentence lesson"
  
  context:
    phase: "Which phase this occurred in"
    situation: "What was happening"
    domain: "Domain this applies to"
    
  detail:
    what_happened: "Description of events"
    why_it_matters: "Impact or significance"
    
  recommendation:
    for_future_projects: "What to do differently"
    applies_when: "Conditions for applicability"
    does_not_apply_when: "Conditions where this lesson is wrong" # v1.1
    
  evidence:
    source: "Where this was observed"
    confidence: "high | medium | low"
    
  # LIFECYCLE TRACKING (NEW in v1.1)
  lifecycle:
    status: "active | validated | disputed | deprecated"
    created_at: "ISO-8601"
    last_validated: "ISO-8601"
    validation_count: 0                 # Times this lesson proved true
    contradiction_count: 0              # Times this lesson proved false
    validation_projects: ["project_ids"]
    contradiction_projects: ["project_ids"]
    confidence_score: "high | medium | low | uncertain"
    
  # CONTRADICTION TRACKING (NEW in v1.1)
  contradictions:
    contradicts_lessons: ["lesson_ids"]
    contradiction_resolution: "string or null"
    resolution_status: "resolved | unresolved | context_dependent"
```

### Lesson Validation Protocol (NEW in v1.1)

```yaml
lesson_validation_protocol:
  purpose: "Ensure lessons remain accurate over time"
  
  validation_triggers:
    - trigger: "Lesson applied in new project"
      action: "Track outcome (validated or contradicted)"
    - trigger: "Similar situation encountered"
      action: "Check if lesson recommendation was followed and outcome"
    - trigger: "6 months since last validation"
      action: "Flag for review"
      
  validation_outcomes:
    validated:
      criteria: "Lesson recommendation followed, positive outcome"
      action: "Increment validation_count, update last_validated"
      
    contradicted:
      criteria: "Lesson recommendation followed, negative outcome OR opposite worked better"
      action: "Increment contradiction_count, flag for review"
      
    not_applicable:
      criteria: "Situation didn't match applies_when conditions"
      action: "No change to validation stats"
      
  confidence_decay:
    rule: "If not validated in 12 months, confidence_score decreases"
    fresh: "Validated within 6 months"
    stale: "Validated 6-12 months ago"
    uncertain: "Not validated in 12+ months"
    
  dispute_resolution:
    if_contradiction_count_exceeds_validation_count:
      action: "Flag lesson as 'disputed'"
      require: "Human review before continued use"
```

### Contradiction Detection Protocol (NEW in v1.1)

```yaml
contradiction_detection:
  purpose: "Identify when new lessons conflict with existing lessons"
  
  detection_method:
    for_each_new_lesson:
      - "Query existing lessons with same category and overlapping applies_when"
      - "Compare recommendations"
      - "If recommendations conflict, flag contradiction"
      
  contradiction_types:
    direct_contradiction:
      definition: "Lessons recommend opposite actions for same situation"
      example: "Lesson A says 'always use webhooks', Lesson B says 'never use webhooks'"
      resolution_required: true
      
    conditional_contradiction:
      definition: "Lessons conflict but may apply to different contexts"
      example: "Lesson A says 'use webhooks for real-time', Lesson B says 'use polling for reliability'"
      resolution: "Add context conditions to both lessons"
      
    supersession:
      definition: "New lesson makes old lesson obsolete"
      example: "Old lesson about API v1, new lesson about API v2"
      resolution: "Deprecate old lesson, link to new"
      
  resolution_strategies:
    context_dependent:
      action: "Both lessons valid, add clearer applies_when conditions"
      update: "Link lessons as 'context_dependent_pair'"
      
    supersede:
      action: "New lesson replaces old"
      update: "Deprecate old lesson, set superseded_by"
      
    merge:
      action: "Combine into more nuanced lesson"
      update: "Create merged lesson, deprecate originals"
      
    escalate:
      action: "Cannot resolve automatically"
      update: "Flag for human review"
      
  output:
    contradictions_found:
      - new_lesson_id: "string"
        existing_lesson_id: "string"
        contradiction_type: "direct | conditional | supersession"
        resolution_strategy: "string"
        resolution_status: "resolved | escalated"
```

---

## Directive 4: System Feedback (10 minutes)

**Objective:** Identify improvements for the meta-prompts themselves.

### Objective Assessment Framework (NEW in v1.1)

```yaml
objective_assessment:
  purpose: "Reduce bias in self-assessment of meta-prompt effectiveness"
  
  quantitative_metrics:
    time_accuracy:
      metric: "(actual_time - estimated_time) / estimated_time"
      good: "Within ±20%"
      acceptable: "Within ±40%"
      poor: ">40% deviation"
      
    output_completeness:
      metric: "Required fields populated / Total required fields"
      good: "≥95%"
      acceptable: "≥80%"
      poor: "<80%"
      
    downstream_acceptance:
      metric: "Next phase accepted output without modification"
      good: "No modifications needed"
      acceptable: "Minor modifications"
      poor: "Significant rework"
      
    escalation_rate:
      metric: "Escalations triggered / Decisions made"
      note: "Neither high nor low is inherently good; compare to baseline"
      
  qualitative_assessment:
    instruction_clarity:
      assess: "Were instructions unambiguous?"
      evidence: "Cite specific examples of clarity or confusion"
      
    edge_case_handling:
      assess: "Did edge cases arise that weren't covered?"
      evidence: "List specific edge cases encountered"
      
    output_utility:
      assess: "Was output useful to downstream phase?"
      evidence: "Cite downstream phase feedback"
      
  bias_acknowledgment:
    statement: "This assessment is performed by an LLM evaluating prompts that guide LLMs. Inherent bias toward positive assessment is possible."
    mitigation:
      - "Prioritize quantitative metrics over qualitative"
      - "Cite specific evidence for all claims"
      - "Flag uncertainties explicitly"
      - "Recommend human review for significant changes"
```

### Feedback Categories

```yaml
feedback_categories:
  meta_prompt_effectiveness:
    description: "How well did each meta-prompt guide the work?"
    assess_per_phase:
      - "Time budget accuracy (quantitative)"
      - "Output completeness (quantitative)"
      - "Downstream acceptance (quantitative)"
      - "Instruction clarity (qualitative with evidence)"
      - "Edge case coverage (qualitative with evidence)"
      
  pipeline_flow:
    description: "How well did phases connect?"
    assess:
      - "Were handoffs smooth?"
      - "Was information preserved?"
      - "Were there gaps between phases?"
      
  missing_capabilities:
    description: "What should the system do that it doesn't?"
    capture:
      - "Tasks that required manual intervention"
      - "Decisions that couldn't be automated"
      - "Information that was needed but not available"
      
  suggested_improvements:
    description: "Specific changes to improve the system"
    format:
      - target: "Which meta-prompt or process"
        issue: "What's the problem"
        evidence: "Specific example from this project" # v1.1 requirement
        suggestion: "Proposed improvement"
        priority: "high | medium | low"
        effort_estimate: "hours"
        bias_check: "Is this a genuine issue or LLM preference?" # v1.1
```

---

## Directive 5: Knowledge Organization (10 minutes)

**Objective:** Organize extracted knowledge for future retrieval.

### Template Library Structure

```yaml
template_library_entry:
  template: "full template structure"
  
  indexing:
    primary_category: "component | workflow | integration | decision"
    domains: ["string"]
    systems: ["string"]
    complexity: "low | medium | high"
    domain_scope: "broad | narrow"      # v1.1
    
  search_metadata:
    keywords: ["string"]
    use_cases: ["string"]
    problem_solved: "string"
    
  relationships:
    related_templates: ["template_id"]
    required_templates: ["template_id"]
    often_used_with: ["template_id"]
    replaces: "template_id"             # v1.1
    superseded_by: "template_id"        # v1.1
    
  # LIFECYCLE METADATA (NEW in v1.1)
  lifecycle:
    status: "active | deprecated | archived"
    created_at: "ISO-8601"
    last_updated: "ISO-8601"
    last_used: "ISO-8601"
    deprecation_date: "ISO-8601 or null"
    archive_date: "ISO-8601 or null"
```

### Lesson Library Structure (Enhanced in v1.1)

```yaml
lesson_library_entry:
  lesson: "full lesson structure"
  
  indexing:
    category: "success | failure | discovery | calibration"
    phases: ["string"]
    domains: ["string"]
    
  search_metadata:
    keywords: ["string"]
    applies_to: ["string"]
    
  # LIFECYCLE METADATA (NEW in v1.1)
  lifecycle:
    status: "active | validated | disputed | deprecated"
    confidence_score: "high | medium | low | uncertain"
    last_validated: "ISO-8601"
    validation_count: "integer"
    contradiction_count: "integer"
```

---

## Directive 6: Template Lifecycle Management (NEW in v1.1) (10 minutes)

**Objective:** Maintain template health over time.

### Template Health Tracking

```yaml
template_health_tracking:
  purpose: "Track template reuse outcomes to identify problems"
  
  tracking_events:
    reuse_attempt:
      capture:
        - "template_id"
        - "project_id"
        - "date"
        - "parameters_used"
        
    reuse_outcome:
      capture:
        - "success | failure | partial"
        - "if failure: failure_reason"
        - "if partial: what_worked, what_failed"
        - "adaptation_required: none | minor | significant"
        
  health_calculation:
    success_rate: "reuse_success_count / reuse_count"
    
    health_status:
      healthy: "success_rate ≥ 80%"
      concerning: "success_rate 50-79%"
      unhealthy: "success_rate < 50%"
      
    freshness:
      fresh: "last_validated within 6 months"
      stale: "last_validated 6-12 months ago"
      outdated: "last_validated >12 months ago OR dependent system has major version change"
      
  health_actions:
    if_unhealthy:
      action: "Flag for review"
      options:
        - "Fix template based on failure patterns"
        - "Deprecate template"
        - "Split into more specific templates"
        
    if_outdated:
      action: "Flag for revalidation"
      options:
        - "Revalidate with current project"
        - "Update to current system versions"
        - "Deprecate if no longer relevant"
```

### Template Failure Feedback Loop

```yaml
failure_feedback_loop:
  purpose: "Learn from template reuse failures"
  
  when_template_fails:
    immediate_actions:
      - "Log failure with context"
      - "Capture failure_reason"
      - "Update template health stats"
      
    analysis_questions:
      - "Was this a template problem or usage problem?"
      - "Did the user misapply the template (wrong use case)?"
      - "Did the template have a bug or gap?"
      - "Did external systems change since template creation?"
      
    feedback_to_library:
      if_template_problem:
        - "Add to known_limitations"
        - "Update do_not_use_when"
        - "Consider template revision"
        
      if_usage_problem:
        - "Improve documentation"
        - "Add clearer prerequisites"
        - "Enhance example usage"
        
      if_external_change:
        - "Flag template as potentially outdated"
        - "Schedule revalidation"
        
  feedback_output:
    template_id: "string"
    failure_type: "template_bug | usage_error | external_change | unknown"
    failure_description: "string"
    recommended_action: "fix | document | deprecate | investigate"
    priority: "high | medium | low"
```

### Template Versioning

```yaml
template_versioning:
  semantic_versioning:
    major: "Breaking changes - incompatible with previous usage"
    minor: "New features - backward compatible"
    patch: "Bug fixes - backward compatible"
    
  version_update_triggers:
    major_version:
      - "Parameter removed or renamed"
      - "Required parameter added"
      - "Core logic fundamentally changed"
      - "Incompatible with previous parameter values"
      
    minor_version:
      - "Optional parameter added"
      - "New use case supported"
      - "Performance improvement"
      - "Additional documentation"
      
    patch_version:
      - "Bug fix"
      - "Documentation clarification"
      - "Example updated"
      
  version_history_entry:
    version: "string"
    date: "ISO-8601"
    changes: "string"
    source_project: "string"
    breaking_change: "boolean"
    migration_guide: "string (if breaking)"
    
  migration_guidance:
    when_breaking_change:
      provide:
        - "What changed"
        - "Why it changed"
        - "How to migrate existing usage"
        - "Deadline for migration (if deprecating old version)"
```

---

## Directive 7: Knowledge Maintenance (NEW in v1.1) (5 minutes)

**Objective:** Define ongoing maintenance strategy.

### Obsolescence Strategy

```yaml
obsolescence_strategy:
  purpose: "Prevent library from accumulating zombie knowledge"
  
  deprecation_criteria:
    templates:
      automatic_deprecation:
        - "Health status 'unhealthy' for 3+ months"
        - "No reuse in 18+ months AND not a foundational pattern"
        - "Dependent system deprecated or major version change (2+ versions behind)"
        
      manual_deprecation:
        - "Superseded by better template"
        - "Pattern no longer considered best practice"
        - "Security vulnerability discovered"
        
    lessons:
      automatic_deprecation:
        - "Contradiction_count > validation_count AND last_validated > 12 months"
        - "Applies_when conditions no longer possible (system deprecated)"
        
      manual_deprecation:
        - "Superseded by more nuanced lesson"
        - "Based on incorrect understanding"
        
  deprecation_process:
    step_1: "Mark status as 'deprecated'"
    step_2: "Set deprecation_date"
    step_3: "Add deprecation_reason"
    step_4: "Link to replacement (superseded_by) if exists"
    step_5: "Notify users who have used this template/lesson"
    step_6: "After 6 months, move to 'archived'"
    
  archival_vs_deletion:
    archive:
      when: "May have historical value or edge case applicability"
      action: "Move to archive, exclude from default search"
      
    delete:
      when: "Actively harmful or completely wrong"
      action: "Remove from library, log deletion reason"
      rare: "Deletion should be rare; prefer archival"
```

### Library Quality Maintenance

```yaml
library_quality_maintenance:
  periodic_reviews:
    quarterly:
      - "Review templates with health status 'concerning' or 'unhealthy'"
      - "Review lessons with status 'disputed'"
      - "Check for duplicate or near-duplicate entries"
      
    annually:
      - "Review all templates not used in past year"
      - "Review all lessons not validated in past year"
      - "Assess overall library health metrics"
      - "Prune or archive low-value entries"
      
  quality_metrics:
    library_health_score:
      calculation: "weighted average of template health statuses"
      target: "≥80% healthy templates"
      
    freshness_score:
      calculation: "percentage of templates validated in past 12 months"
      target: "≥70% fresh"
      
    usage_rate:
      calculation: "templates used at least once / total active templates"
      target: "≥50% usage rate"
      
  maintenance_actions:
    if_health_score_low:
      - "Prioritize fixing unhealthy templates"
      - "Consider bulk deprecation of unused templates"
      
    if_freshness_score_low:
      - "Schedule revalidation sprint"
      - "Prioritize high-value templates"
      
    if_usage_rate_low:
      - "Improve template discoverability"
      - "Deprecate templates that solve obsolete problems"
```

---

## Output Schema (Enhanced in v1.1)

```yaml
extraction_output:
  # Level 1: Metadata
  metadata:
    run_id: "string"
    phase: "phase_6_template_extraction"
    timestamp: "ISO-8601"
    duration_minutes: "integer"
    project_name: "string"
    extraction_scope: "full | limited | skipped"
    meta_prompt_version: "1.1.0"        # v1.1
    
  # Level 1: Extraction Summary
  extraction_summary:
    templates_created: "integer"
    templates_versioned: "integer"       # v1.1 - updates to existing
    templates_skipped_duplicate: "integer" # v1.1
    lessons_documented: "integer"
    contradictions_found: "integer"      # v1.1
    contradictions_resolved: "integer"   # v1.1
    system_improvements_identified: "integer"
    
    by_pattern_type:
      component: "integer"
      workflow: "integer"
      integration: "integer"
      decision: "integer"
      
    by_lesson_category:
      success: "integer"
      failure: "integer"
      discovery: "integer"
      calibration: "integer"
      
  # Level 1: Deduplication Results (NEW in v1.1)
  deduplication_results:
    templates_checked: "integer"
    duplicates_avoided: "integer"
    versions_created: "integer"
    complements_created: "integer"
    details:
      - candidate: "string"
        decision: "create_new | version_existing | complement_existing | skip_duplicate"
        rationale: "string"
        
  # Level 1: Templates
  templates:
    - # full template structure for each (with v1.1 enhancements)
    
  # Level 1: Lessons
  lessons:
    - # full lesson structure for each (with v1.1 lifecycle tracking)
    
  # Level 1: Contradiction Report (NEW in v1.1)
  contradiction_report:
    new_contradictions:
      - new_lesson_id: "string"
        existing_lesson_id: "string"
        contradiction_type: "string"
        resolution_strategy: "string"
        resolution_status: "string"
    unresolved_contradictions:
      - lesson_ids: ["string"]
        description: "string"
        escalation_reason: "string"
        
  # Level 1: System Feedback
  system_feedback:
    meta_prompt_assessments:
      - phase: "string"
        quantitative_metrics:           # v1.1
          time_accuracy: "percentage"
          output_completeness: "percentage"
          downstream_acceptance: "good | acceptable | poor"
        qualitative_assessment:
          instruction_clarity: "string"
          edge_cases_encountered: ["string"]
        suggested_improvements:
          - improvement: "string"
            evidence: "string"          # v1.1 requirement
            priority: "string"
            bias_check: "string"        # v1.1
            
    pipeline_observations: {}
    improvement_backlog: []
    
  # Level 1: Knowledge Organization
  knowledge_organization:
    template_library_entries: []
    lesson_library_entries: []
    cross_references: {}
    
  # Level 1: Library Health Report (NEW in v1.1)
  library_health_report:
    current_library_size:
      templates: "integer"
      lessons: "integer"
    health_metrics:
      template_health_score: "percentage"
      freshness_score: "percentage"
      usage_rate: "percentage"
    maintenance_flags:
      templates_needing_review: ["template_id"]
      lessons_needing_validation: ["lesson_id"]
      pending_deprecations: ["template_id | lesson_id"]
      
  # Level 1: Value Assessment
  value_assessment:
    estimated_time_savings: "string"
    reusability_score: "high | medium | low"
    knowledge_captured: "comprehensive | adequate | minimal"
    compounding_value_projection: "string" # v1.1
    
  # Level 1: Next Steps
  next_steps:
    templates_to_validate:
      - template_id: "string"
        validation_method: "string"
    lessons_to_share:
      - lesson_id: "string"
        audience: "string"
    improvements_to_prioritize:
      - improvement_id: "string"
        rationale: "string"
    maintenance_actions:              # v1.1
      - action: "string"
        target: "string"
        priority: "string"
```

---

## Quality Gate (Enhanced in v1.1)

```yaml
gate_process:
  step_1_extraction_scope:
    action: "Verify extraction scope is appropriate"
    check: "Scope matches readiness assessment"
    on_fail: "Adjust scope or document limitation"
    
  step_2_deduplication_complete:        # NEW in v1.1
    action: "Verify deduplication check was performed"
    check: "All pattern candidates have deduplication_check result"
    on_fail: "Complete deduplication check"
    
  step_3_template_quality:
    action: "Verify all templates meet quality checklist"
    checks:
      - "Every template has description"
      - "Every template has use_when and do_not_use_when"
      - "Every template has parameters documented"
      - "Every template has validation evidence"
      - "Every template has version and health tracking initialized" # v1.1
      - "Every template has complexity assessment" # v1.1
      - "No template has >12 parameters without split_recommendation" # v1.1
    on_fail: "Complete missing elements"
    
  step_4_lesson_quality:
    action: "Verify lessons are actionable"
    checks:
      - "Every lesson has recommendation"
      - "Every lesson has evidence"
      - "No vague lessons ('things went well')"
      - "Every lesson has lifecycle tracking initialized" # v1.1
      - "Contradiction check completed for all lessons" # v1.1
    on_fail: "Make lessons specific"
    
  step_5_contradiction_resolution:      # NEW in v1.1
    action: "Verify contradictions are addressed"
    check: "All detected contradictions have resolution_status"
    on_fail: "Resolve or escalate contradictions"
    
  step_6_organization:
    action: "Verify knowledge is organized for retrieval"
    checks:
      - "Templates have search metadata"
      - "Lessons have indexing"
      - "Cross-references are complete"
      - "Lifecycle metadata is initialized" # v1.1
    on_fail: "Add missing organization"
    
  step_7_system_feedback_objectivity:   # NEW in v1.1
    action: "Verify system feedback uses objective criteria"
    checks:
      - "Quantitative metrics provided where possible"
      - "Qualitative claims have specific evidence"
      - "Bias acknowledgment included"
    on_fail: "Add evidence or acknowledge uncertainty"
    
  step_8_value_justification:
    action: "Verify extraction provides value"
    check: "At least one template OR three lessons extracted"
    on_fail: "Document why extraction yielded little"
    
  step_9_maintenance_plan:              # NEW in v1.1
    action: "Verify maintenance actions are identified"
    check: "next_steps.maintenance_actions is non-empty if library_health_report shows issues"
    on_fail: "Identify maintenance actions"
```

---

## Failure Modes & Guards (Enhanced in v1.1)

### Original Guards (v1.0)

| Failure Mode | Problem | Guard |
|--------------|---------|-------|
| Over-Extraction | Extracting noise | Pattern candidates require reusability rationale |
| Under-Abstraction | Templates too specific | Abstraction guidelines with parameterization |
| Over-Abstraction | Templates too generic | Concrete examples required |
| Vague Lessons | Generic observations | Specific recommendations required |
| Orphaned Knowledge | Can't find later | Search metadata required |
| Unvalidated Templates | Patterns that didn't work | Phase 5b evidence required |

### New Guards (v1.1)

| Failure Mode | Problem | Guard |
|--------------|---------|-------|
| Duplicate Templates | Same pattern extracted multiple times | Library query & deduplication check |
| Template Degradation | Template fails on reuse, no one knows | Health tracking & failure feedback loop |
| Wrong Lessons | Incorrect lessons become canonical | Lesson validation protocol with confidence decay |
| Contradictory Knowledge | Conflicting lessons coexist | Contradiction detection & resolution |
| Zombie Templates | Outdated templates persist | Obsolescence strategy with deprecation criteria |
| Self-Assessment Bias | LLM rates own prompts too positively | Objective assessment framework |
| Over-Parameterization | Templates too complex to use | Complexity limits with split recommendations |
| Niche Template Sprawl | Too many domain-specific templates | Domain scoping guidelines |

---

## Time Budget (Adjusted for v1.1)

**Total: 70 minutes** (increased from 55 to accommodate lifecycle management)

| Step/Directive | Time | Focus |
|----------------|------|-------|
| 0. Readiness Assessment | 5 min | Should we extract? |
| 0.5. Library Query & Deduplication | 5 min | What already exists? (NEW) |
| 1. Pattern Identification | 15 min | What's worth extracting? |
| 2. Template Creation | 20 min | Create reusable templates |
| 3. Lesson Documentation | 10 min | Capture insights |
| 4. System Feedback | 10 min | Improve the system |
| 5. Knowledge Organization | 10 min | Make it findable |
| 6. Template Lifecycle Management | 10 min | Maintain health (NEW) |
| 7. Knowledge Maintenance | 5 min | Define maintenance plan (NEW) |

**Time Management:**
- If time-constrained, prioritize: Deduplication → Templates → Lessons → Lifecycle
- Always complete deduplication check (prevents duplicate work)
- Lifecycle management can be deferred to batch processing if needed

---

## The Compounding Effect (Validated in v1.1)

With lifecycle management, compounding continues:

```
Year 1:  Extract → Library grows → Value compounds ✓
Year 2:  Deduplicate → Validate → Quality maintained ✓
Year 3:  Deprecate zombie → Resolve contradictions → Library healthy ✓
Year 4+: Sustainable growth → Compounding continues ✓
```

Without lifecycle management (v1.0):
```
Year 1:  ✓
Year 2:  ⚠️ Duplicates, contradictions
Year 3:  ✗ Zombie templates, wrong lessons
Year 4+: ✗ Library is burden
```

**v1.1 enables sustainable compounding value.**

---

*End of Meta-Prompt v1.1*
