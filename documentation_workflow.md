---
status: tested
last updated: 2025-04-06
version: "1.0"
---
> [!tested] Tested Document
> This document has been validated in real projects.
# Documentation and Planning Workflow

This document outlines our approach to documentation, planning, and tracking throughout the development process. We use a structured system of support files to maintain clarity, provide context, and ensure continuity across the project lifecycle.

## Important Note on Documentation Storage

All development logs, TODO lists, and development plans are stored **locally in each project repository** in the appropriate `dev-logs/` directory, not in this Obsidian vault. This ensures:

1. All project-specific documentation remains with the code it describes
2. Documentation is included in version control alongside code changes
3. Team members can access documentation without needing Obsidian
4. Documentation history is preserved in the git history

The Obsidian vault contains higher-level documentation like this workflow guide, principles, and cross-project references, but day-to-day development logs stay with their respective projects.

## Documentation Types and Their Purpose

### 1. Change Requests

**Location:** `change-requests/XXXXXXX.md` (where XXXXXXX is a unique identifier)

**Purpose:** Defines the scope, requirements, and acceptance criteria for a new feature or significant change.

**Structure:**
- Original client request
- Change specification
- Time estimate and timeline
- Acceptance criteria
- Technical solution overview
- Approvals

**When to create:** At the beginning of a new feature/project or when a significant change is requested.

**Example:**
```markdown
# Change Request #001/2025

**Date Created:** 06.03.2025
**Author:** Michal Příhoda

## 1. Original Client Request
...

## 2. Change Specification
...

## 3. Time Estimate and Cost
...
```

### 2. Gherkin Feature Files

**Location:** `project-name/features/*.feature`

**Purpose:** Describes feature behavior in a business-readable, domain-specific language that serves as both specification and test basis.

**Structure:**
- Feature title and description
- Scenarios with Given-When-Then steps
- Examples for scenario outlines

**When to create:** When defining a new feature, before or alongside implementation.

**Example:**
```gherkin
Feature: Source Account Management
  As a user of the YNAB Importer
  I want to manage bank source accounts
  So that I can import transactions from my bank accounts to YNAB

  Scenario: View list of source accounts
    When I navigate to the source accounts page
    Then I should see a list of all source accounts
    And I should see details including account name, bank ID, and status
```

### 3. TODO Lists

**Location:** `dev-logs/TODO.md`

**Purpose:** Tracks remaining tasks for project completion, organized by feature area.

**Structure:**
- Overview of project goals
- References to feature files
- Completed tasks (with ✅)
- Remaining tasks organized by feature/component
- Revised priority order
- Timeline reminders

**When to update:** Regularly as tasks are completed and new tasks are identified.

**Example:**
```markdown
# TODO: Project Name Completion

## Features
- [Feature 1](/path/to/feature1.feature)
- [Feature 2](/path/to/feature2.feature)

## Completed Tasks
- ✅ Task A
- ✅ Task B

## Remaining Tasks
### 1. Feature 1 
- [ ] Task C
- [ ] Task D

### 2. Feature 2
- [ ] Task E
```

### 4. Development Logs

**Location:** `dev-logs/YYYYMMDD.md` (date format)

**Purpose:** Documents the actual implementation, decisions made, problems encountered, and solutions applied.

**Structure:**
- Title (what was implemented/changed)
- Feature references
- Problem description
- Solution approach
- Implementation details with code snippets
- Challenges and their solutions

**When to create:** For each significant development session, dated by when the work occurred.

**Example:**
```markdown
# Implementation: Feature X Component

**Feature Reference**: [Feature X](/path/to/feature.feature)

## Problem
...

## Solution Approach
...

## Implementation Details
```

### 5. Development Plans

**Location:** `dev-logs/YYYYMMDD-plan.md` (date format)

**Purpose:** Outlines the planned work for an upcoming development session.

**Structure:**
- Goals for the day/session
- Specific tasks with time estimates
- Success criteria
- Notes and considerations

**When to create:** Before beginning a new development session or at the end of the previous one, planning for the next day.

**Example:**
```markdown
# Development Plan for YYYY-MM-DD

## Goals
1. Complete component X
2. Start implementation of feature Y

## Tasks
### 1. Task A (1 hour)
- [ ] Subtask 1
- [ ] Subtask 2

## Success Criteria
...
```

## Workflow Integration

### Daily Development Cycle

1. **Start of Day:**
   - Review the current development plan (`YYYYMMDD-plan.md`)
   - Check the TODO list for context on where the task fits
   - Reference related feature files for behavior specifications

2. **During Development:**
   - Document decisions, challenges, and solutions as notes
   - Update tests to reflect implementation
   - Track progress against plan

3. **End of Day:**
   - Create a development log (`YYYYMMDD.md`) documenting what was accomplished
   - Update the TODO list, marking completed items
   - Create a plan for the next day (`YYYYMMDD-plan.md` for tomorrow)

### Feature Implementation Cycle

1. **Feature Definition:**
   - Create or reference a change request
   - Write Gherkin feature files defining behavior
   - Add tasks to the TODO list

2. **Implementation:**
   - Follow daily development cycle
   - Document progress in dev logs
   - Update TODO as tasks are completed

3. **Completion:**
   - Ensure all scenarios from the feature file are implemented and tested
   - Mark feature as complete in TODO
   - Reference implementation in change request for acceptance

## Best Practices

1. **Consistent Reference:**
   - Always reference feature files in dev logs
   - Link change requests where applicable
   - Use consistent naming conventions

2. **Detailed Documentation:**
   - Include code snippets in dev logs
   - Document both the "what" and the "why" of implementation decisions
   - Note challenges and how they were overcome

3. **Living Documentation:**
   - Update TODO lists regularly
   - Revise feature files if requirements change
   - Keep development plans realistic and focused

4. **Balanced Detail:**
   - Keep daily plans actionable and specific
   - Focus dev logs on significant decisions and implementations
   - Ensure feature files are readable by non-technical stakeholders

## Tools and Integration

- Use Playwright with ZIO for e2e testing of features
- Integrate feature files with e2e tests
- Use git commits to track documentation changes alongside code
- Consider automating the creation of daily log templates

---

This documentation workflow helps maintain clarity throughout the development process, provides context for future work, and ensures that all team members understand what has been done and what remains to be completed.