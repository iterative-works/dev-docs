---
status: draft
last updated: {{DATE}}
version: "0.1"
---

# Feature Implementation Plan: {{FEATURE_NAME}}

## Feature Overview
- **Description**: {{BRIEF_DESCRIPTION}}
- **Change Request**: [CR-{{CR_NUMBER}}](../change-requests/{{CR_NUMBER}}.md)
- **Feature File**: [{{FEATURE_FILE_NAME}}](../features/{{FEATURE_FILE_NAME}}.feature)
- **Business Value**: {{BUSINESS_VALUE}}

## Domain Analysis

### Domain Concepts
| Concept | Description | Type | Existing/New |
|---------|-------------|------|--------------|
| {{CONCEPT_1}} | {{DESCRIPTION}} | {{TYPE}} | {{EXISTING_OR_NEW}} |
| {{CONCEPT_2}} | {{DESCRIPTION}} | {{TYPE}} | {{EXISTING_OR_NEW}} |

### Domain Invariants
- {{INVARIANT_1}}: {{DESCRIPTION}}
- {{INVARIANT_2}}: {{DESCRIPTION}}

### Domain Events
- {{EVENT_1}}: {{DESCRIPTION}}
- {{EVENT_2}}: {{DESCRIPTION}}

## Implementation Steps

### Step 1: {{STEP_DESCRIPTION}}
- **Component Type**: {{COMPONENT_TYPE}} (e.g., Entity, Value Object, Repository Interface)
- **Component Name**: `{{COMPONENT_NAME}}`
- **Package Location**: `{{PACKAGE_PATH}}`
- **Purpose**: {{PURPOSE}}
- **Key Behaviors**:
  - {{BEHAVIOR_1}}
  - {{BEHAVIOR_2}}
- **Dependencies**:
  - {{DEPENDENCY_1}}
  - {{DEPENDENCY_2}}
- **Acceptance Criteria**:
  - {{CRITERIA_1}}
  - {{CRITERIA_2}}
- **Implementation Guide**: [{{GUIDE_NAME}}](../guides/{{GUIDE_NAME}}.md)

### Step 2: {{STEP_DESCRIPTION}}
- **Component Type**: {{COMPONENT_TYPE}}
- **Component Name**: `{{COMPONENT_NAME}}`
- **Package Location**: `{{PACKAGE_PATH}}`
- **Purpose**: {{PURPOSE}}
- **Key Behaviors**:
  - {{BEHAVIOR_1}}
  - {{BEHAVIOR_2}}
- **Dependencies**:
  - {{DEPENDENCY_1}}
  - {{DEPENDENCY_2}}
- **Acceptance Criteria**:
  - {{CRITERIA_1}}
  - {{CRITERIA_2}}
- **Implementation Guide**: [{{GUIDE_NAME}}](../guides/{{GUIDE_NAME}}.md)

<!-- Add more steps as needed -->

## Bounded Context Integration
- **Primary Bounded Context**: {{PRIMARY_BOUNDED_CONTEXT}}
- **Related Bounded Contexts**:
  - {{RELATED_CONTEXT_1}}: {{INTEGRATION_APPROACH}}
  - {{RELATED_CONTEXT_2}}: {{INTEGRATION_APPROACH}}

## Environment Composition
- **Required ZIO Services**:
  - {{SERVICE_1}}
  - {{SERVICE_2}}
- **New Services to Implement**:
  - {{NEW_SERVICE_1}}
  - {{NEW_SERVICE_2}}

## Testing Strategy
### Unit Tests
- **Domain Logic Tests**:
  - {{TEST_DESCRIPTION_1}}
  - {{TEST_DESCRIPTION_2}}

### Integration Tests
- **Repository Tests**:
  - {{TEST_DESCRIPTION_1}}
  - {{TEST_DESCRIPTION_2}}

### End-to-End Tests
- **Feature Scenarios**:
  - {{SCENARIO_1}}
  - {{SCENARIO_2}}

## Implementation Schedule
- **Estimated Total Time**: {{TOTAL_TIME}} hours
- **Step Dependencies**:
  1. Step 1 → Step 2
  2. Step 3 → Step 4, Step 5
  
| Step | Description | Est. Time | Prerequisites | Developer |
|------|-------------|-----------|--------------|-----------|
| 1    | {{STEP_1_DESC}} | {{TIME_1}}h | None | {{DEV_1}} |
| 2    | {{STEP_2_DESC}} | {{TIME_2}}h | Step 1 | {{DEV_2}} |
<!-- Add more rows as needed -->

## Risk Assessment
- **Technical Risks**:
  - {{RISK_1}}: {{MITIGATION_1}}
  - {{RISK_2}}: {{MITIGATION_2}}

- **Domain Risks**:
  - {{RISK_1}}: {{MITIGATION_1}}
  - {{RISK_2}}: {{MITIGATION_2}}

## Review Checklist
- [ ] All domain concepts clearly identified
- [ ] Steps are properly sequenced
- [ ] Dependencies between components identified
- [ ] Each step has clear acceptance criteria
- [ ] Testing strategy covers all components
- [ ] Implementation guides referenced for each step
- [ ] Risks identified with mitigation strategies