---
name: executing-plans
description: Use when you have a written implementation plan to execute in a separate session with review checkpoints
---

# Executing Plans

## Overview

Load plan, review critically, execute tasks in batches, report for review between batches.

**Core principle:** Batch execution with checkpoints for architect review.

**Announce at start:** "I'm using the executing-plans skill to implement this plan."

## The Process

### Step 1: Load and Review Plan
1. Read plan file
2. Review critically - identify any questions or concerns about the plan
3. If concerns: Raise them with your human partner before starting
4. If no concerns: Create TodoWrite and proceed

### Step 2: Execute Batch
**Default: First 3 tasks**

For each task:
1. Mark as in_progress
2. Follow each step exactly (plan has bite-sized steps)
3. Run verifications as specified
4. Mark as completed

### Step 3: Report
When batch complete:
- Show what was implemented
- Show verification output
- Say: "Ready for feedback."

### Step 4: Continue
Based on feedback:
- Apply changes if needed
- Execute next batch
- Repeat until complete

### Step 5: Complete Development

After all tasks complete and verified:
- Announce: "I'm using the finishing-a-development-branch skill to complete this work."
- **REQUIRED SUB-SKILL:** Use superpowers:finishing-a-development-branch
- Follow that skill to verify tests, present options, execute choice

## Browser QA Verification

If a task includes a **Browser QA** section, execute it after the commit step:

1. **Start dev server** (if `start:` specified)
   - Run the command in background
   - Wait for server to be ready (check port or output)

2. **Run browser checks**
   - Use `agent-browser` commands to verify each check
   - Reference `superpowers:browser-qa` for command mapping

3. **Take screenshot** (if specified)
   - Save to `docs/screenshots/` or path in plan
   - Include in batch report as evidence

4. **Clean up**
   - Close browser: `agent-browser close`
   - Stop dev server if you started it

**If browser QA fails:**
- Report the specific check that failed
- Include screenshot showing current state
- Mark task as blocked, don't proceed to next task

**Example execution:**

```bash
# Start server
npm run dev &
sleep 3

# Run checks
agent-browser open http://localhost:3000
agent-browser snapshot -i
agent-browser find text "Login" click
agent-browser wait dialog
agent-browser screenshot docs/screenshots/task-3-login.png
agent-browser close

# Stop server
kill %1
```

## When to Stop and Ask for Help

**STOP executing immediately when:**
- Hit a blocker mid-batch (missing dependency, test fails, instruction unclear)
- Plan has critical gaps preventing starting
- You don't understand an instruction
- Verification fails repeatedly

**Ask for clarification rather than guessing.**

## When to Revisit Earlier Steps

**Return to Review (Step 1) when:**
- Partner updates the plan based on your feedback
- Fundamental approach needs rethinking

**Don't force through blockers** - stop and ask.

## Remember
- Review plan critically first
- Follow plan steps exactly
- Don't skip verifications
- Reference skills when plan says to
- Between batches: just report and wait
- Stop when blocked, don't guess
- Never start implementation on main/master branch without explicit user consent

## Integration

**Required workflow skills:**
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
- **superpowers:writing-plans** - Creates the plan this skill executes
- **superpowers:finishing-a-development-branch** - Complete development after all tasks
