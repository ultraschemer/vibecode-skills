### OpenCode Skill: Local Spec-Driven Development (SDD)

### Description

This skill enforces Spec-Driven Development by compelling the agent to write, update, and validate feature specifications and technical execution plans directly within the local repository (specs/ directory) before writing or modifying any production code. This prevents global memory drift and ensures all agent planning is captured in Git version control. 

### System Prompt & Instructions

### 1. Mandatory Pre-Flight Phase (No-Code Gate)

Before writing, refactoring, or deleting any production code, you **must** pause and execute the following ritual: 

* **Check for Specification:** Inspect the specs/ directory to see if a relevant spec.md or feature folder exists.
* **Halt and Plan First:** You are strictly forbidden from editing application logic until an explicit execution plan is formally written down and saved to disk locally.

### 2. File Architecture & Directory Mapping

All plans and requirements must be organized strictly within the workspace root. Do not rely on home-directory agent paths (~/.claude/ or ~/.opencode/). 

Structure your artifacts using this exact layout: 

text

specs/
└── [feature-id]-[kebab-case-name]/
    ├── spec.md   # Functional requirements, edge cases, and success criteria.
    └── plan.md   # Step-by-step technical implementation path and checkboxes.

Use code with caution.

Please, create the feature-ids numerically, in a way we can order the features by the time they have been asked to be generated. Infer the kebab-case-name from the current context and the features asked to be developed, to avoid the necessity to edit such names to make them meaninful.

### 3. Execution Plan Template (plan.md)

When creating or updating an execution plan, you **must** use the following exact Markdown template. Save this as specs/[feature-id]-[kebab-case-name]/plan.md: 

markdown

# Execution Plan: [Feature Name]

## 🎯 Objective
[One concise sentence summarizing the goal of this technical intervention]

## 🛠 Proposed Changes
### Affected Files
- [ ] `path/to/file.ext`: [Brief description of planned change]

### New Files to Create
- [ ] `path/to/new_file.ext`: [Purpose of the new file]

## 📋 Step-by-Step Implementation Tasklist
- [ ] **Step 1: Scaffolding & Setup**
  - [ ] Task detail 1
- [ ] **Step 2: Core Logic Implementation**
  - [ ] Task detail 2
- [ ] **Step 3: Verification & Integration Tests**
  - [ ] Task detail 3

## 🚨 Risk Mitigation & Edge Cases
- **Risk:** [e.g., Database migration locking table X] -> **Mitigation:** [e.g., Use concurrent index creation]

Use code with caution.

### 4. Interactive State Tracking Workflow

1. **Drafting:** Write the spec.md and plan.md to the local folder.
2. **Approval Request:** Present the path of the saved local file to the user: *"I have saved the execution plan to specs/[feature-id]/plan.md. Please review and approve."*
3. **Execution Verification:** As you make progress through your tasks, you **must** update the checkboxes ([ ] to [x]) in the local plan.md file after completing each milestone. Do not keep the state solely in your context window.
