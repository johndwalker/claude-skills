---
name: bmad-deliver-epic
description: Deliver all incomplete stories in a BMAD Epic using an enhanced quality pipeline with worktree isolation. Runs create-story, ATDD, development, code review, and trace sequentially per story. Adds Advanced Elicitation (5 methods, party-mode cherry-picked) after planning steps (1-3), uses sonnet for code review, opus for all other agents, and triggers adversarial review whenever BMAD recommends it. Use when user wants to deliver an entire epic, run the full BMAD delivery pipeline, or says "deliver epic".
---

# BMAD Deliver Epic

## CRITICAL BEHAVIORAL RULES — READ FIRST

You are a **coordinator only**. You MUST NOT use any tool other than the **Agent tool** and text output.

1. **You NEVER use Read, Edit, Write, Bash, Grep, Glob, or any other tool.** Only the Agent tool. Every file read, file write, bash command, and slash command MUST be done by a spawned agent.
2. **EVERY pipeline step MUST be delegated to a separate Agent tool call.** One agent per step. Fresh context each time.
3. **STRICTLY SEQUENTIAL.** Do not spawn the next agent until the previous one returns successfully.
4. **On failure, STOP.** Report which step failed and why. Do not continue.
5. **Model rules are non-negotiable:** Code Review uses `model: "sonnet"`. All other agents use `model: "opus"`.
6. **All agents use `mode: "bypassPermissions"`.** This enables autonomous execution.

Your ONLY allowed actions: spawn agents via the Agent tool, read agent results, print progress text. Nothing else.

---

## Execution Flow

### Pre-step: Determine Epic & Story List

Spawn an agent to resolve the epic number and collect stories:

```
USE the Agent tool with EXACTLY these parameters:
  description: "Collect Epic story list"
  model: "opus"
  mode: "bypassPermissions"
  prompt: "
    1. Read _bmad-output/implementation-artifacts/sprint-status.yaml (or docs/sprint/sprint-status.yaml).
    2. If epic number '{ARGUMENT}' was provided, use it. Otherwise find the smallest epic number with incomplete stories.
    3. Filter entries matching that epic with status not 'done'.
    4. Sort by story number ascending.
    5. Return EXACTLY this format:
       EPIC_NUM: X
       STORIES:
       X-1 | story-name | status
       X-2 | story-name | status
       TOTAL: N
    6. If no incomplete stories, return: NO_STORIES"
```

Parse the agent's result to get `EPIC_NUM` and the story list. If `NO_STORIES`, report and stop.

---

### For Each Story: Full Pipeline

Process stories sequentially. For each story, set `STORY_ID` (e.g., `3-2`).

#### Step 0: Create Worktree

Spawn an agent to create the worktree:

```
USE the Agent tool:
  description: "Create worktree for story {STORY_ID}"
  model: "opus"
  mode: "bypassPermissions"
  prompt: "Run these bash commands:
    ORIGINAL_REPO_PATH=$(pwd)
    PROJECT_NAME=$(basename $(pwd))
    WORKTREE_PATH=\"../${PROJECT_NAME}-story-{STORY_ID}\"
    git worktree add -b feature/story-{STORY_ID} \"$WORKTREE_PATH\" $(git branch --show-current)
    If the branch already exists, use: git worktree add \"$WORKTREE_PATH\" feature/story-{STORY_ID}
    Verify with: git worktree list
    Return EXACTLY: WORKTREE_PATH=<absolute path to worktree> and ORIGINAL_REPO_PATH=<absolute path to original repo>"
```

Parse agent result to capture `WORKTREE_PATH` and `ORIGINAL_REPO_PATH`.

---

#### Steps 1-5: Pipeline Steps

Read **references/workflow-steps.md** for step definitions (or use defaults below). For each step:

**PLANNING STEPS (Steps 1, 2, 3)** — 3 sequential agents per step:

**Agent A — Execute the step command:**
```
USE the Agent tool:
  description: "{Step name} for story {STORY_ID}"
  model: "opus"
  mode: "bypassPermissions"
  prompt: "cd {WORKTREE_PATH} && Execute {COMMAND} {STORY_ID} yolo.
    Return: 1) Completion status 2) Key outputs 3) Any issues 4) Whether adversarial review is recommended"
```

Wait for Agent A to return. Then spawn:

**Agent B — Advanced Elicitation (all 5 methods):**
```
USE the Agent tool:
  description: "Advanced Elicitation for {STORY_ID} step {N}"
  model: "opus"
  mode: "bypassPermissions"
  prompt: "cd {WORKTREE_PATH} && Run Advanced Elicitation on the artifacts just produced by {step_name} for story {STORY_ID}.
    Load the elicitation methods from _bmad/core/workflows/advanced-elicitation/workflow.xml.
    Select the 5 most relevant methods for the artifacts.
    Execute ALL 5 methods. For each, generate proposed improvements with clear before/after.
    Return: all 5 sets of proposed improvements grouped by method."
```

Wait for Agent B to return. Capture output as ELICITATION_OUTPUT. Then spawn:

**Agent C — Party Mode cherry-picks:**
```
USE the Agent tool:
  description: "Party Mode review for {STORY_ID} step {N}"
  model: "opus"
  mode: "bypassPermissions"
  prompt: "cd {WORKTREE_PATH} && Execute /bmad-party-mode to review these Advanced Elicitation proposals for story {STORY_ID}:
    {ELICITATION_OUTPUT}
    Evaluate each proposed improvement. Accept, reject, or modify based on group consensus.
    Apply all accepted improvements to the artifacts on disk.
    Return: list of accepted/rejected improvements with reasoning."
```

Wait for Agent C to return. Then check for adversarial review trigger (see below).

---

**NON-PLANNING STEPS (Steps 4, 5)** — single agent:

```
USE the Agent tool:
  description: "{Step name} for story {STORY_ID}"
  model: "{sonnet for Step 4 code review, opus for Step 5}"
  mode: "bypassPermissions"
  prompt: "cd {WORKTREE_PATH} && Execute {COMMAND} {STORY_ID} yolo.
    Return: 1) Completion status 2) Key outputs 3) Any issues 4) Whether adversarial review is recommended"
```

Wait for completion. Check for adversarial review trigger.

---

#### Adversarial Review Protocol

**After EVERY step**, inspect the agent's returned text. Trigger adversarial review if ANY of these appear:

- Output recommends "adversarial review", "cynical review", or "critical review"
- Output contains unresolved HIGH severity issues
- Output explicitly recommends further review

When triggered, spawn TWO sequential agents:

**Agent AR-1 — Adversarial Review:**
```
USE the Agent tool:
  description: "Adversarial review for {STORY_ID}"
  model: "opus"
  mode: "bypassPermissions"
  prompt: "cd {WORKTREE_PATH} && Execute /bmad-review-adversarial-general for story {STORY_ID}.
    Focus on all artifacts produced so far. Return: complete findings report with severity levels."
```

Wait for AR-1. Capture output as ADVERSARIAL_OUTPUT. Then spawn:

**Agent AR-2 — Party Mode analyzes findings:**
```
USE the Agent tool:
  description: "Party Mode analyzes adversarial findings for {STORY_ID}"
  model: "opus"
  mode: "bypassPermissions"
  prompt: "cd {WORKTREE_PATH} && Execute /bmad-party-mode to analyze these adversarial review findings for story {STORY_ID}:
    {ADVERSARIAL_OUTPUT}
    Evaluate each finding. Determine which require immediate fixes vs acceptable risks.
    Apply critical fixes to artifacts. Return: action taken for each finding."
```

After remediation, continue to the next pipeline step.

---

#### Step 6: Merge Worktree

Spawn an agent to merge:

```
USE the Agent tool:
  description: "Merge worktree for story {STORY_ID}"
  model: "opus"
  mode: "bypassPermissions"
  prompt: "Run these bash commands in sequence:
    cd {WORKTREE_PATH} && git add . && git commit -m 'feat: complete story {STORY_ID}'
    cd {ORIGINAL_REPO_PATH}
    git merge feature/story-{STORY_ID} --no-edit
    git worktree remove {WORKTREE_PATH}
    git branch -d feature/story-{STORY_ID}
    Return: merge result and any conflicts"
```

If merge fails, report and preserve worktree.

---

#### Step 7: Update Status

Spawn an agent to update status files:

```
USE the Agent tool:
  description: "Update status for story {STORY_ID}"
  model: "opus"
  mode: "bypassPermissions"
  prompt: "
    1. Edit _bmad-output/implementation-artifacts/sprint-status.yaml (or docs/sprint/sprint-status.yaml): change story {STORY_ID} status to 'done'
    2. Find the story document for {STORY_ID} and set Status to 'done', check all task boxes
    Return: confirmation of updates"
```

---

### Progress Reporting

After each agent returns, output a brief status line:

```
[X/5] Step Name — status (elicitation: applied/skipped, adversarial: triggered/not)
```

On failure:
```
PIPELINE FAILED at Step X: {Step Name}
Error: {details}
Worktree preserved: {WORKTREE_PATH}
```

---

### Epic Completion

After all stories are delivered, output a summary of every story's result.

To resume after failure: re-run `/bmad-deliver-epic {EPIC_NUM}` — auto-skips stories with status `done`.

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
