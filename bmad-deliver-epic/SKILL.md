---
name: bmad-deliver-epic
description: Deliver all incomplete stories in a BMAD Epic using an enhanced quality pipeline with worktree isolation. Uses Claude Code agent teams (not subagents) for each pipeline step. Runs create-story, ATDD, development, code review, and trace sequentially per story. Adds Advanced Elicitation (5 methods, party-mode cherry-picked) after planning steps (1-3), uses sonnet for code review, opus for all other agents, and triggers adversarial review whenever BMAD recommends it. Use when user wants to deliver an entire epic, run the full BMAD delivery pipeline, or says "deliver epic".
---

# BMAD Deliver Epic

## CRITICAL BEHAVIORAL RULES — READ FIRST

You are the **team lead and coordinator**. You orchestrate work by creating a team, spawning teammates, creating tasks, and sending messages. You MUST NOT do implementation work yourself.

### Tool Usage Rules

1. **Use TeamCreate** to create the team at the start.
2. **Use the Agent tool with `team_name` and `name` parameters** to spawn each teammate. This creates real teammates, NOT subagents. Every teammate MUST have both `team_name` and `name` set.
3. **Use TaskCreate** to define work items. Use **TaskUpdate** to assign them and track completion.
4. **Use SendMessage** to communicate with teammates and to shut them down when done.
5. **You NEVER use Read, Edit, Write, Bash, Grep, or Glob yourself.** All file operations and command execution happen inside teammates.
6. **STRICTLY SEQUENTIAL pipeline execution.** Do not spawn the next pipeline teammate until the previous one completes its task and you have confirmed success.
7. **On failure, STOP.** Report which step failed and why. Do not continue.

### Model Rules (Non-Negotiable)

- **Code Review (Step 4)**: `model: "sonnet"`
- **All other teammates**: `model: "opus"`

### Permission Rules

- **All teammates**: `mode: "bypassPermissions"`

---

## Phase 0: Setup Team and Collect Stories

### 0.1 Create Team

```
USE TeamCreate:
  team_name: "bmad-epic-{EPIC_NUM}"
  description: "BMAD delivery pipeline for Epic {EPIC_NUM}"
```

If `{ARGUMENT}` is empty, use team name `"bmad-epic-auto"` and let the first teammate determine the epic number.

### 0.2 Spawn Scout Teammate to Collect Story List

```
USE Agent tool:
  team_name: "bmad-epic-{EPIC_NUM}"
  name: "scout"
  model: "opus"
  mode: "bypassPermissions"
  description: "Collect Epic story list"
  prompt: "You are on team bmad-epic-{EPIC_NUM}. Your task:
    1. Read _bmad-output/implementation-artifacts/sprint-status.yaml (or docs/sprint/sprint-status.yaml).
    2. If epic number '{ARGUMENT}' was provided, use it. Otherwise find the smallest epic number with incomplete stories.
    3. Filter entries matching that epic with status not 'done'.
    4. Sort by story number ascending.
    5. Return EXACTLY this format in your message back:
       EPIC_NUM: X
       STORIES:
       X-1 | story-name | status
       X-2 | story-name | status
       TOTAL: N
    6. If no incomplete stories, return: NO_STORIES"
```

Wait for scout to return. Parse result to get `EPIC_NUM` and story list. If `NO_STORIES`, report and stop. Send scout a shutdown request.

---

## Phase 1: Deliver Stories Sequentially

For each story in the list, set `STORY_ID` (e.g., `3-2`) and execute the full pipeline below.

### Step 0: Create Worktree

Create a task and spawn a teammate:

```
USE TaskCreate:
  title: "Create worktree for story {STORY_ID}"
  description: "Create isolated git worktree for story {STORY_ID}"

USE Agent tool:
  team_name: "bmad-epic-{EPIC_NUM}"
  name: "worktree-{STORY_ID}"
  model: "opus"
  mode: "bypassPermissions"
  description: "Create worktree for story {STORY_ID}"
  prompt: "You are on team bmad-epic-{EPIC_NUM}. Run these bash commands:
    ORIGINAL_REPO_PATH=$(pwd)
    PROJECT_NAME=$(basename $(pwd))
    WORKTREE_PATH=\"../${PROJECT_NAME}-story-{STORY_ID}\"
    git worktree add -b feature/story-{STORY_ID} \"$WORKTREE_PATH\" $(git branch --show-current)
    If the branch already exists, use: git worktree add \"$WORKTREE_PATH\" feature/story-{STORY_ID}
    Verify with: git worktree list
    Return EXACTLY: WORKTREE_PATH=<absolute path> and ORIGINAL_REPO_PATH=<absolute path>
    Then mark your task completed."
```

Wait for result. Capture `WORKTREE_PATH` and `ORIGINAL_REPO_PATH`. Shut down teammate.

---

### Steps 1-5: Pipeline Steps

Refer to **references/workflow-steps.md** for step definitions, or use the defaults at the bottom of this file.

---

#### PLANNING STEPS (Steps 1, 2, 3) — Enhanced Quality Flow

For each planning step, spawn 3 sequential teammates:

**Teammate A — Execute the step command:**

```
USE TaskCreate:
  title: "Step {N}: {step_name} for story {STORY_ID}"

USE Agent tool:
  team_name: "bmad-epic-{EPIC_NUM}"
  name: "step{N}-{STORY_ID}"
  model: "opus"
  mode: "bypassPermissions"
  description: "{Step name} for story {STORY_ID}"
  prompt: "You are on team bmad-epic-{EPIC_NUM}. Your task:
    cd {WORKTREE_PATH}
    Execute {COMMAND} {STORY_ID} yolo
    Return: 1) Completion status 2) Key outputs 3) Any issues 4) Whether adversarial review is recommended
    Mark your task completed when done."
```

Wait for Teammate A. Shut it down. Then spawn:

**Teammate B — Advanced Elicitation (all 5 methods):**

```
USE TaskCreate:
  title: "Elicitation for step {N} story {STORY_ID}"

USE Agent tool:
  team_name: "bmad-epic-{EPIC_NUM}"
  name: "elicit{N}-{STORY_ID}"
  model: "opus"
  mode: "bypassPermissions"
  description: "Advanced Elicitation for {STORY_ID} step {N}"
  prompt: "You are on team bmad-epic-{EPIC_NUM}. Your task:
    cd {WORKTREE_PATH}
    Run Advanced Elicitation on the artifacts just produced by {step_name} for story {STORY_ID}.
    Load the elicitation methods from _bmad/core/workflows/advanced-elicitation/workflow.xml.
    Select the 5 most relevant methods for the artifacts.
    Execute ALL 5 methods. For each, generate proposed improvements with clear before/after.
    Return: all 5 sets of proposed improvements grouped by method.
    Mark your task completed when done."
```

Wait for Teammate B. Capture output as ELICITATION_OUTPUT. Shut it down. Then spawn:

**Teammate C — Party Mode cherry-picks:**

```
USE TaskCreate:
  title: "Party Mode review for step {N} story {STORY_ID}"

USE Agent tool:
  team_name: "bmad-epic-{EPIC_NUM}"
  name: "party{N}-{STORY_ID}"
  model: "opus"
  mode: "bypassPermissions"
  description: "Party Mode review for {STORY_ID} step {N}"
  prompt: "You are on team bmad-epic-{EPIC_NUM}. Your task:
    cd {WORKTREE_PATH}
    Execute /bmad-party-mode to review these Advanced Elicitation proposals for story {STORY_ID}:
    {ELICITATION_OUTPUT}
    Evaluate each proposed improvement. Accept, reject, or modify based on group consensus.
    Apply all accepted improvements to the artifacts on disk.
    Return: list of accepted/rejected improvements with reasoning.
    Mark your task completed when done."
```

Wait for Teammate C. Shut it down. Check for adversarial review trigger (see below).

---

#### NON-PLANNING STEPS (Steps 4, 5) — Single Teammate

```
USE TaskCreate:
  title: "Step {N}: {step_name} for story {STORY_ID}"

USE Agent tool:
  team_name: "bmad-epic-{EPIC_NUM}"
  name: "step{N}-{STORY_ID}"
  model: "{sonnet for Step 4, opus for Step 5}"
  mode: "bypassPermissions"
  description: "{Step name} for story {STORY_ID}"
  prompt: "You are on team bmad-epic-{EPIC_NUM}. Your task:
    cd {WORKTREE_PATH}
    Execute {COMMAND} {STORY_ID} yolo
    Return: 1) Completion status 2) Key outputs 3) Any issues 4) Whether adversarial review is recommended
    Mark your task completed when done."
```

Wait for completion. Shut down teammate. Check for adversarial review trigger.

---

### Adversarial Review Protocol

**After EVERY step**, inspect the teammate's returned text. Trigger adversarial review if ANY of:

- Output recommends "adversarial review", "cynical review", or "critical review"
- Output contains unresolved HIGH severity issues
- Output explicitly recommends further review

When triggered, spawn TWO sequential teammates:

**Teammate AR-1 — Adversarial Review:**

```
USE TaskCreate:
  title: "Adversarial review for story {STORY_ID}"

USE Agent tool:
  team_name: "bmad-epic-{EPIC_NUM}"
  name: "adversarial-{STORY_ID}"
  model: "opus"
  mode: "bypassPermissions"
  description: "Adversarial review for {STORY_ID}"
  prompt: "You are on team bmad-epic-{EPIC_NUM}. Your task:
    cd {WORKTREE_PATH}
    Execute /bmad-review-adversarial-general for story {STORY_ID}.
    Focus on all artifacts produced so far.
    Return: complete findings report with severity levels.
    Mark your task completed when done."
```

Wait. Capture ADVERSARIAL_OUTPUT. Shut down. Then:

**Teammate AR-2 — Party Mode analyzes findings:**

```
USE TaskCreate:
  title: "Party Mode analyze adversarial findings for story {STORY_ID}"

USE Agent tool:
  team_name: "bmad-epic-{EPIC_NUM}"
  name: "adversarial-party-{STORY_ID}"
  model: "opus"
  mode: "bypassPermissions"
  description: "Party Mode analyzes adversarial findings for {STORY_ID}"
  prompt: "You are on team bmad-epic-{EPIC_NUM}. Your task:
    cd {WORKTREE_PATH}
    Execute /bmad-party-mode to analyze these adversarial review findings for story {STORY_ID}:
    {ADVERSARIAL_OUTPUT}
    Evaluate each finding. Determine which require immediate fixes vs acceptable risks.
    Apply critical fixes to artifacts.
    Return: action taken for each finding.
    Mark your task completed when done."
```

Shut down after completion. Continue to next pipeline step.

---

### Step 6: Merge Worktree

```
USE TaskCreate:
  title: "Merge worktree for story {STORY_ID}"

USE Agent tool:
  team_name: "bmad-epic-{EPIC_NUM}"
  name: "merge-{STORY_ID}"
  model: "opus"
  mode: "bypassPermissions"
  description: "Merge worktree for story {STORY_ID}"
  prompt: "You are on team bmad-epic-{EPIC_NUM}. Run these bash commands:
    cd {WORKTREE_PATH} && git add . && git commit -m 'feat: complete story {STORY_ID}'
    cd {ORIGINAL_REPO_PATH}
    git merge feature/story-{STORY_ID} --no-edit
    git worktree remove {WORKTREE_PATH}
    git branch -d feature/story-{STORY_ID}
    Return: merge result and any conflicts.
    Mark your task completed when done."
```

If merge fails, report and preserve worktree.

---

### Step 7: Update Status

```
USE TaskCreate:
  title: "Update status for story {STORY_ID}"

USE Agent tool:
  team_name: "bmad-epic-{EPIC_NUM}"
  name: "status-{STORY_ID}"
  model: "opus"
  mode: "bypassPermissions"
  description: "Update status for story {STORY_ID}"
  prompt: "You are on team bmad-epic-{EPIC_NUM}. Your task:
    1. Edit _bmad-output/implementation-artifacts/sprint-status.yaml (or docs/sprint/sprint-status.yaml): change story {STORY_ID} status to 'done'
    2. Find the story document for {STORY_ID} and set Status to 'done', check all task boxes
    Return: confirmation of updates.
    Mark your task completed when done."
```

---

## Phase 2: Epic Completion

### Progress Reporting

After each teammate returns, output a brief status line:

```
[X/5] Step Name — status (elicitation: applied/skipped, adversarial: triggered/not)
```

On failure:
```
PIPELINE FAILED at Step X: {Step Name}
Error: {details}
Worktree preserved: {WORKTREE_PATH}
```

### Epic Complete Summary

After all stories are delivered, output a summary of every story's result.

### IMPORTANT: Post-Epic Reminder

After the epic delivery summary, you MUST display this reminder to the user:

```
REMINDER: Before proceeding to any new work, run a BMAD Epic Review:
  /bmad-bmm-retrospective {EPIC_NUM}
This reviews the epic delivery for lessons learned, quality assessment,
and process improvements. Do not skip this step.
```

### Cleanup

Send shutdown requests to any remaining teammates. Then:

```
USE TeamDelete
```

### Resume After Failure

Re-run `/bmad-deliver-epic {EPIC_NUM}` — auto-skips stories with status `done`.

---

## Default Pipeline Steps

If references/workflow-steps.md is not accessible, use these defaults:

| Step | Command | Model | Planning |
|------|---------|-------|----------|
| 1. Create Story | `/bmad-bmm-create-story` | opus | yes |
| 2. ATDD Tests | `/bmad-tea-testarch-atdd` | opus | yes |
| 3. Development | `/bmad-bmm-dev-story` | opus | yes |
| 4. Code Review | `/bmad-bmm-code-review` | sonnet | no |
| 5. Trace Coverage | `/bmad-tea-testarch-trace` | opus | no |

## Configuration

Customize by editing **references/workflow-steps.md**: add/remove steps, toggle planning flag, change models, reorder.
