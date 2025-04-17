---
status: ai-generated
last updated:
  "{ DATE }": 
version: "0.1"
---
> [!robot] AI-Generated Content
> This is AI-generated content pending human review. Information may be inaccurate or misaligned with actual processes.
# Implementation Handoff Protocol: {{FEATURE_NAME}}

## Pre-implementation Requirements Checklist

### Documentation Review
- [ ] Complete [Feature Implementation Plan](./{{FEATURE_PLAN_FILENAME}}.md) document
- [ ] Create [Gherkin feature file](../features/{{FEATURE_FILE_NAME}}.feature) with all scenarios
- [ ] Set up [Progress tracking document](./{{PROGRESS_TRACKING_FILENAME}}.md)
- [ ] Review all implementation guides referenced in the plan
- [ ] Create or update any necessary TODO items in the project TODO list

### Environment Setup
- [ ] Required ZIO services are available in environment
- [ ] Development environment has all necessary dependencies
- [ ] Test environment configured properly
- [ ] Feature flags configured if needed

### Dependency Verification
- [ ] All required bounded contexts available
- [ ] External system integrations configured
- [ ] Database migrations planned (if applicable)
- [ ] All required libraries and dependencies available

## Implementation Process Guidelines

### Getting Started
1. Clone/pull the latest code from the repository
2. Create a new branch following the naming convention: `feature/{{FEATURE_BRANCH_NAME}}`
3. Review the entire Feature Implementation Plan before starting
4. Set up any necessary environment configurations locally

### Implementation Workflow
1. Follow the implementation steps in the order specified in the plan
   - Only deviate from the order when explicitly noted
   - Document any deviations with clear justification
2. For each step:
   - Review the relevant implementation guide
   - Implement according to the guide's specifications
   - Write tests according to the testing strategy
   - Update the progress tracking document after completing the step

### Coding Standards
1. Follow the project's architectural guidelines:
   - Functional Core/Imperative Shell pattern
   - Proper component categorization
   - Clear separation of concerns
2. Apply code style guidelines:
   - 4-space indentation, 100 column limit
   - New Scala 3 syntax without braces when appropriate
   - End markers for methods with 5+ lines
3. Adhere to naming conventions:
   - Domain concepts use CamelCase reflecting ubiquitous language
   - Methods express intent clearly

### Communication Protocols
1. Update the progress tracking document at the end of each day
2. Document any questions, issues, or decisions in the Questions & Decisions section
3. For discovered requirements:
   - Document in the progress tracking document
   - Assess impact on existing plan
   - Consult with team before making significant deviations

### Testing Requirements
1. Write tests before or alongside code (TDD approach)
2. Ensure tests for each component as specified in the plan:
   - Unit tests for domain logic
   - Integration tests for repositories
   - E2E tests based on Gherkin scenarios
3. Verify all tests pass before submitting code for review

## Completion Criteria

### Implementation Completion
- [ ] All steps in the implementation plan are marked complete
- [ ] All acceptance criteria for each step have been met
- [ ] All planned tests are implemented and passing
- [ ] No known bugs or issues remain (or all are documented)

### Documentation Completion
- [ ] Progress tracking document fully updated
- [ ] Any deviations from the plan documented with justification
- [ ] Development logs completed for significant implementation sessions
- [ ] Code documentation completed according to project standards

### Review Process
1. Code Review:
   - Create a pull request following the project's PR template
   - Request reviews from designated reviewers
   - Address all review comments
2. Feature Testing:
   - Verify all acceptance criteria in test environment
   - Complete end-to-end testing of all scenarios
3. Final Approval:
   - Product owner reviews completed feature
   - QA signs off on testing completion

### Deployment Preparation
- [ ] Feature flags configured for controlled rollout (if applicable)
- [ ] Deployment documentation updated
- [ ] Rollback plan documented
- [ ] Monitoring and alerting configured as needed

## Post-Implementation Activities

### Knowledge Transfer
- [ ] Update project documentation
- [ ] Schedule knowledge sharing session (if needed)
- [ ] Create any necessary user documentation

### Retrospective
- [ ] Complete retrospective section in the progress tracking document
- [ ] Share lessons learned with the team
- [ ] Identify process improvements for future implementations

### Follow-up Tasks
- [ ] Create maintenance TODO items if needed
- [ ] Plan for monitoring after deployment
- [ ] Schedule follow-up review after deployment