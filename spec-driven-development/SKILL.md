---
name: spec-driven-development
description: Use before writing, refactoring, or deleting any production code. Enforces Spec-Driven Development (SDD) by requiring a spec.md and plan.md to be written and saved under <project-root>/specs/[feature-id]-[kebab-case-name]/ in the Git repository that owns the changed files, before any code is touched, and requires plan.md checkboxes to be updated as tasks complete. Trigger on "new feature", "implement", "add endpoint", "refactor", "plan this", or any request that changes application logic.
license: MIT
metadata:
  audience: maintainers
  workflow: planning
---

# Local Spec-Driven Development (SDD)

This skill enforces Spec-Driven Development by compelling the agent to write, update, and validate feature specifications and technical execution plans at the root of the Git repository that owns the files being changed (`<project-root>/specs/`) before writing or modifying any production code. This prevents global memory drift and ensures all agent planning is captured in Git version control.

## 0. Scope and Precedence

**Precedence:** This skill sets the floor for how a change is planned, not the whole of it. Where a project ships its own agent instructions (`AGENTS.md`, `CLAUDE.md`, or equivalent), those orders govern, and this skill is applied inside them rather than in place of them.

## 1. Mandatory Pre-Flight Phase (No-Code Gate)

Before writing, refactoring, or deleting any production code, you **must** pause and execute the following ritual:

**Scope of this gate.** Application code is always in scope: compiled or interpreted source, scripts, database queries and code templates. Committed declarative infrastructure is in scope as well, so Ansible playbooks, CMake files, CI definitions and configuration files all require a spec. A directory that is not a Git repository, such as `psf-ansible-deploy` in this workspace, is still in scope; section 2.1 step 3 then applies and you must ask the user where the artifacts belong before writing anything. Do not assume a location, and do not skip the ritual because the change looks small.

- **Read Project Instructions First:** Locate and read the agent instruction files the project ships (`AGENTS.md`, `CLAUDE.md`, or equivalent) and every document they mandate, before anything else. Those orders are project-specific and can contradict what a model would otherwise assume from training data. In `psf-user-web`, `AGENTS.md` requires reading the Next.js guides in `node_modules/next/dist/docs/` first, and states that the installed version breaks from published conventions.
- **Check for Specification:** Inspect the `specs/` directory to see if a relevant `spec.md` or feature folder exists.
- **Halt and Plan First:** You are strictly forbidden from editing application logic until an explicit execution plan is formally written down and saved to disk locally.

**When the session cannot write.** If the session cannot write to disk, because it is in plan mode, on a read-only mount, or the user asked for a dry run, do not treat the missing files as a failure and do not skip the ritual. Write the complete `spec.md` and `plan.md` content in the response, state plainly that nothing was saved to disk, and write the files as the first action once editing is permitted.

## 2. File Architecture & Directory Mapping

All plans and requirements must be organized strictly within the root of the Git repository that owns the files being changed, never at the root of a workspace that merely contains several repositories. Do not rely on home-directory agent paths (`~/.claude/` or `~/.opencode/`).

### 2.1 Resolving the Project Root

Before creating any artifact, resolve the owning project root from the first file you are about to change:

1. Run `git rev-parse --show-toplevel` from the directory of the target file.
2. If it prints a path, that directory is the project root. Place artifacts in `<project-root>/specs/`, even when the session was started from a parent directory or from a subdirectory of the project.
3. If the command fails, no Git repository owns the file. Do not fall back to the workspace root. Ask the user where the artifacts should live, and wait for the answer before writing anything.
4. If the change spans more than one repository, resolve each repository separately and follow section 2.3.

### 2.2 Layout

```text
<project-root>/
`-- specs/
    `-- [feature-id]-[kebab-case-name]/
        |-- spec.md   # Functional requirements, edge cases, and success criteria.
        `-- plan.md   # Step-by-step technical implementation path and checkboxes.
```

Never create `specs/` at the root of a directory that is not itself a Git repository. Such a directory cannot capture the artifacts in version control, and a single folder shared by unrelated projects mixes their plans together.

### 2.3 Cross-Repository Changes

When one change touches more than one repository, write a complete `spec.md` and `plan.md` set in the root of every affected repository, all sharing the same `[feature-id]-[kebab-case-name]`.

The dependency is mandatory in both directions and must be verified before the change is considered planned:

- Every `spec.md` carries a `## Cross-references` section naming the sibling `spec.md` path of each other affected project.
- Every `plan.md` lists the sibling `plan.md` paths under Affected Files.
- If any one of them is missing a cross-reference, the plan is incomplete and no code may be written.

Create the feature IDs numerically per project, counting only the existing `specs/` folders in that project root, so IDs stay unique and ordered within each repository. When a change spans repositories, reuse one ID across all of them. Infer the kebab-case-name from the current context and the features asked to be developed, to avoid the necessity to edit such names to make them meaningful.

## 3. Document Templates

Both documents are mandatory and their contents must not be duplicated across them. `spec.md` states what the change is and how it will be judged. `plan.md` states how it will be built.

- A section that genuinely does not apply is written as `Not applicable: <reason>`. Never delete a section, and never pad it with text that carries no information.
- Every requirement, edge case, and success criterion carries an identifier: `R#` for requirements, `EC#` for edge cases, `AC#` for success criteria.
- Identifiers are unique within the `spec.md`, and every `plan.md` step names the identifiers it delivers.

### 3.1 Specification Template (spec.md)

Save this as `<project-root>/specs/[feature-id]-[kebab-case-name]/spec.md`.

```markdown
# Specification: [Feature Name]

Feature ID: [feature-id]-[kebab-case-name]
Project: [project-root repository name]
Date: [YYYY-MM-DD]

## Summary
[Two or three sentences: the problem, who feels it, what changes.]

## Scope
In scope:
- [What this change delivers]

Out of scope:
- [What is explicitly excluded, and why]

## Requirements
- R1: [Testable functional requirement]
- R2: [Testable functional requirement]

## Edge Cases and Failure Modes
- EC1: [Condition] -> [Expected behaviour]

## Interface and Data Impact
- API: [Endpoint added or changed, with method and path]
- Database: [Table or column added, changed or migrated]
- Inter-service: [Message or interface affected, and the peer project]
- None: [Use only when the change touches no interface at all]

## Success Criteria
- AC1: [Observable outcome that proves R1 is met]
- AC2: [Observable outcome that proves R2 is met]

## Cross-references
- [Sibling spec.md path of each other affected project, as required by section 2.3. Write "None" for a single-repository change.]

## Open Questions
- [Question] -> [Answer, or "Blocking: awaiting user decision"]
```

## 4. Execution Plan Template (plan.md)

When creating or updating an execution plan, you **must** use the following exact Markdown template. Save this as `<project-root>/specs/[feature-id]-[kebab-case-name]/plan.md`:

```markdown
# Execution Plan: [Feature Name]

## Objective
[One concise sentence summarizing the goal of this technical intervention]

## Proposed Changes
### Affected Files
- [ ] `path/to/file.ext`: [Brief description of planned change]
<!-- Cross-repository changes only: also list the sibling plan.md path of each
     other affected project, as required by section 2.3. -->

### New Files to Create
- [ ] `path/to/new_file.ext`: [Purpose of the new file]

## Step-by-Step Implementation Tasklist
Each step must name the spec.md identifiers it delivers (R#, EC#, AC#).

- [ ] **Step 1: Scaffolding & Setup (R1)**
  - [ ] Task detail 1
- [ ] **Step 2: Core Logic Implementation (R2, EC1)**
  - [ ] Task detail 2
- [ ] **Step 3: Verification & Integration Tests (AC1, AC2)**
  - [ ] Task detail 3

## Risk Mitigation & Edge Cases
- **Risk:** [e.g., Database migration locking table X] -> **Mitigation:** [e.g., Use concurrent index creation]
```

## 5. Interactive State Tracking Workflow

1. **Drafting:** Write the `spec.md` and `plan.md` to `<project-root>/specs/[feature-id]-[kebab-case-name]/`.
2. **Approval Request:** Present the path of the saved local file to the user: *"I have saved the execution plan to `<project-root>/specs/[feature-id]-[kebab-case-name]/plan.md`. Please review and approve."*
3. **Execution Verification:** As you make progress through your tasks, you **must** update the checkboxes (`[ ]` to `[x]`) in the local `plan.md` file after completing each milestone. Do not keep the state solely in your context window. When the session cannot write to disk, track progress in the response instead and reconcile the checkboxes in one pass as soon as editing is permitted.
