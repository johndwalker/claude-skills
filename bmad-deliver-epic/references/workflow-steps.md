# BMAD Deliver Epic - Workflow Steps

## Pipeline Configuration

Execute steps in order. Each step runs in its own agent with fresh context.
Steps marked `planning: true` get Advanced Elicitation (5 methods) + Party Mode cherry-pick.

## Steps

### Step 1: Create User Story
- Command: `/bmad-bmm-create-story`
- Description: Create story file with context from planning docs
- Model: opus
- Planning: true
- Return: Story ID, Title, Created files

### Step 2: Generate ATDD Tests
- Command: `/bmad-tea-testarch-atdd`
- Description: Generate failing acceptance tests (TDD red phase)
- Model: opus
- Planning: true
- Return: ATDD checklist, test files created

### Step 3: Development
- Command: `/bmad-bmm-dev-story`
- Description: Implement story to pass tests (TDD green phase)
- Model: opus
- Planning: true
- Return: Modified files, changes summary

### Step 4: Code Review
- Command: `/bmad-bmm-code-review`
- Description: Adversarial code review for quality issues
- Model: sonnet
- Planning: false
- Return: Conclusion (pass/needs-fix), issues by severity

### Step 5: Trace Test Coverage
- Command: `/bmad-tea-testarch-trace`
- Description: Generate traceability matrix and quality gate
- Model: opus
- Planning: false
- Return: Coverage percentage, gate decision

## Post-Pipeline

After all steps complete successfully:
1. Update sprint-status.yaml: story status -> done
2. Update story document: Status -> done, Tasks -> checked

## Customization

To modify the pipeline:
- Add/remove steps (update step numbers accordingly)
- Toggle `Planning: true/false` to control which steps get elicitation + party mode
- Change `Model:` to control which model runs each step
- Change step commands or reorder them
- Add new BMAD commands as steps (e.g., `/bmad-bmm-retrospective`)
