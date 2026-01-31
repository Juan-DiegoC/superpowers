# Browser QA Integration Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add browser-based QA verification to superpowers so UI tasks can be visually validated using agent-browser.

**Architecture:** Extend three skills (writing-plans, executing-plans, code-reviewer) to understand and execute browser QA steps. Plans explicitly declare browser QA when needed; reviewers execute those checks using agent-browser commands.

**Tech Stack:** agent-browser CLI, markdown extensions to plan format

---

## Task 1: Add Browser QA Format Documentation

**Files:**
- Create: `skills/browser-qa/SKILL.md`

**Step 1: Create the browser-qa skill directory**

```bash
mkdir -p skills/browser-qa
```

**Step 2: Write the Browser QA reference skill**

Create `skills/browser-qa/SKILL.md`:

```markdown
---
name: browser-qa
description: Reference for browser-based QA verification in plans and code review
---

# Browser QA

## Overview

Browser QA enables visual and functional verification of UI changes using agent-browser. This skill documents the format for specifying browser checks in plans and how to execute them.

## When to Include Browser QA

Include a **Browser QA** section in plan tasks when:
- Adding or modifying visible UI components
- Changing styles, layout, or CSS
- Adding user interactions (clicks, forms, navigation)
- Modifying responsive behavior

**Skip when:**
- Pure backend/API changes
- Config or tooling changes
- Test-only changes (unit tests, not E2E)

## Browser QA Format

Add this section to any task that needs visual verification:

```markdown
**Browser QA:**

> [Human-readable description of what to verify]

- **start:** `[command to start dev server]`
- **url:** `[URL to navigate to]`
- **checks:**
  - `visible: [selector]` — [why this matters]
  - `click: [selector]` — [expected result]
  - `text: [selector] "[expected text]"` — [context]
  - `styles: [selector] | [property]: [value]` — [design requirement]
  - `fill: [selector] "[value]"` — [form input test]
  - `count: [selector] [number]` — [list verification]
- **screenshot:** `[filename.png]`
```

## Check Types

| Check | agent-browser Command | Purpose |
|-------|----------------------|---------|
| `visible: button "Login"` | `agent-browser find text "Login" && agent-browser is visible @e1` | Element exists and visible |
| `click: button "Submit"` | `agent-browser find text "Submit" click` | Interaction works |
| `text: .header "Welcome"` | `agent-browser get text .header` then verify | Content correct |
| `styles: .btn \| background: #3b82f6` | `agent-browser get styles .btn` then verify | CSS correct |
| `fill: input[name="email"] "test@example.com"` | `agent-browser fill @ref "test@example.com"` | Form input works |
| `count: .list-item 5` | `agent-browser get count ".list-item"` | Correct number of items |

## Execution Workflow

When executing browser QA (in executing-plans or code-reviewer):

1. **Start server** (if specified)
   ```bash
   # Run in background, wait for ready
   npm run dev &
   sleep 3  # or wait for port
   ```

2. **Open browser**
   ```bash
   agent-browser open [url]
   ```

3. **Get interactive elements**
   ```bash
   agent-browser snapshot -i
   ```

4. **Execute each check**
   - Parse the check type and selector
   - Run appropriate agent-browser command
   - Verify result matches expectation
   - Record pass/fail

5. **Take screenshot**
   ```bash
   agent-browser screenshot [filename]
   ```

6. **Close browser**
   ```bash
   agent-browser close
   ```

7. **Stop server** (if started)

## Example Task with Browser QA

```markdown
### Task 3: Add Login Button to Header

**Files:**
- Modify: `src/components/Header.tsx:45-60`
- Test: `tests/components/Header.test.tsx`

**Step 1-5:** [TDD steps as usual...]

**Browser QA:**

> Verify login button appears in header and opens auth modal.

- **start:** `npm run dev`
- **url:** `http://localhost:3000`
- **checks:**
  - `visible: button "Login"` — should be in header top-right
  - `click: button "Login"` — opens modal
  - `visible: dialog` — auth modal appears
  - `visible: input[name="email"]` — email field in modal
  - `visible: input[name="password"]` — password field in modal
- **screenshot:** `task-3-login-button.png`
```

## Troubleshooting

**Server won't start:**
- Check if port is already in use
- Try `lsof -i :3000` to find blocking process

**Element not found:**
- Run `agent-browser snapshot -i` to see available elements
- Check if element needs time to render: `agent-browser wait @ref`

**Styles don't match:**
- Use `agent-browser get styles @ref` to see computed values
- Check for CSS specificity issues

**Screenshot blank:**
- Ensure page is fully loaded: `agent-browser wait --load networkidle`
```

**Step 3: Commit**

```bash
git add skills/browser-qa/
git commit -m "feat: add browser-qa reference skill"
```

---

## Task 2: Extend writing-plans to Include Browser QA

**Files:**
- Modify: `skills/writing-plans/SKILL.md`

**Step 1: Read current file**

Read `skills/writing-plans/SKILL.md` to understand current structure.

**Step 2: Add Browser QA section after Task Structure**

Add this section after the "Task Structure" section (around line 88):

```markdown
## Browser QA (for UI tasks)

When a task modifies user-facing UI, include a **Browser QA** section after the commit step.

**Include when:**
- Adding/modifying visible components
- Changing styles or layout
- Adding user interactions (clicks, forms, navigation)

**Format:**

```markdown
**Browser QA:**

> [What to verify in plain English]

- **start:** `[dev server command]`
- **url:** `[URL to test]`
- **checks:**
  - `visible: [selector]` — [context]
  - `click: [selector]` — [expected result]
  - `styles: [selector] | [property]: [value]` — [design requirement]
- **screenshot:** `[descriptive-filename.png]`
```

**Reference:** See `superpowers:browser-qa` skill for full check types and execution details.

**Example:**

```markdown
**Browser QA:**

> Verify the new search bar appears and accepts input.

- **start:** `npm run dev`
- **url:** `http://localhost:3000`
- **checks:**
  - `visible: input[placeholder="Search..."]` — search bar in header
  - `fill: input[placeholder="Search..."] "test query"` — accepts text
  - `click: button[aria-label="Search"]` — triggers search
  - `visible: .search-results` — results container appears
- **screenshot:** `search-bar-functional.png`
```
```

**Step 3: Commit**

```bash
git add skills/writing-plans/SKILL.md
git commit -m "feat(writing-plans): add browser QA section for UI tasks"
```

---

## Task 3: Extend executing-plans to Support Browser QA

**Files:**
- Modify: `skills/executing-plans/SKILL.md`

**Step 1: Read current file**

Read `skills/executing-plans/SKILL.md` to understand current structure.

**Step 2: Add Browser QA execution guidance**

Add this section before "When to Stop and Ask for Help" (around line 50):

```markdown
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
```

**Step 3: Commit**

```bash
git add skills/executing-plans/SKILL.md
git commit -m "feat(executing-plans): add browser QA execution guidance"
```

---

## Task 4: Extend code-reviewer to Execute Browser QA

**Files:**
- Modify: `skills/requesting-code-review/code-reviewer.md`

**Step 1: Read current file**

Read `skills/requesting-code-review/code-reviewer.md` to understand current structure.

**Step 2: Add Browser QA to Review Checklist**

Add this section after "Production Readiness" in the Review Checklist (around line 60):

```markdown
**Browser QA (if specified in plan):**
- Dev server starts successfully?
- All visibility checks pass?
- All interaction checks work?
- Styles match specifications?
- Screenshot captured as evidence?
```

**Step 3: Add Browser QA execution section**

Add this section before "Output Format" (around line 62):

```markdown
## Browser QA Verification

If the plan includes **Browser QA** sections for reviewed tasks, execute them:

1. **Parse the Browser QA section** from the plan
2. **Start dev server** as specified
3. **Execute each check** using agent-browser:
   - `visible:` → verify element exists and is visible
   - `click:` → click and verify expected result
   - `text:` → get text and compare
   - `styles:` → get computed styles and verify
   - `fill:` → input text into form field
4. **Capture screenshot** as evidence
5. **Report results** in Issues section if any checks fail

**Browser QA failures are Important issues** (should fix before merge).

**Example browser QA execution:**

```bash
# Server
npm run dev &
sleep 3

# Checks from plan
agent-browser open http://localhost:3000
agent-browser snapshot -i
# visible: button "Login"
agent-browser find text "Login"  # Returns @e1
agent-browser is visible @e1     # Should return true
# click: button "Login"
agent-browser click @e1
# visible: dialog
agent-browser wait dialog
agent-browser snapshot -i        # Re-snapshot after DOM change
# Screenshot
agent-browser screenshot review-evidence.png
agent-browser close
```
```

**Step 4: Add to Example Output**

Extend the example output section to show browser QA results:

```markdown
### Browser QA Results

**Checks:** 4/4 passed
- `visible: button "Login"` — PASS
- `click: button "Login"` — PASS (modal opened)
- `visible: dialog` — PASS
- `visible: input[name="email"]` — PASS

**Screenshot:** review-evidence.png
```

**Step 5: Commit**

```bash
git add skills/requesting-code-review/code-reviewer.md
git commit -m "feat(code-reviewer): add browser QA verification"
```

---

## Task 5: Update SKILL.md for requesting-code-review

**Files:**
- Modify: `skills/requesting-code-review/SKILL.md`

**Step 1: Read current file**

Read `skills/requesting-code-review/SKILL.md`.

**Step 2: Add Browser QA mention to "When to Request Review"**

Add to the Mandatory section:

```markdown
**Mandatory:**
- After each task in subagent-driven development
- After completing major feature
- Before merge to main
- **After UI tasks with Browser QA specs** — visual verification required
```

**Step 3: Add Browser QA to "Act on feedback"**

Update the section:

```markdown
**3. Act on feedback:**
- Fix Critical issues immediately
- Fix Important issues before proceeding (includes Browser QA failures)
- Note Minor issues for later
- Push back if reviewer is wrong (with reasoning)
```

**Step 4: Commit**

```bash
git add skills/requesting-code-review/SKILL.md
git commit -m "feat(requesting-code-review): reference browser QA in workflow"
```

---

## Task 6: Test the Integration

**Files:**
- Create: `docs/plans/example-browser-qa-task.md` (temporary test file)

**Step 1: Create a sample plan with Browser QA**

Create a minimal test plan:

```markdown
# Test Browser QA Integration

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Verify browser QA integration works.

**Architecture:** Simple test task.

**Tech Stack:** agent-browser

---

### Task 1: Test Browser QA Execution

**Files:**
- None (verification only)

**Step 1: Skip to Browser QA**

No code changes, just verify browser QA works.

**Browser QA:**

> Verify agent-browser can open a page and take a screenshot.

- **url:** `https://example.com`
- **checks:**
  - `visible: h1` — main heading exists
  - `text: h1 "Example Domain"` — correct title
- **screenshot:** `test-browser-qa.png`
```

**Step 2: Manually verify execution**

Run the browser QA steps manually:

```bash
agent-browser open https://example.com
agent-browser snapshot -i
agent-browser get text h1
agent-browser screenshot /tmp/test-browser-qa.png
agent-browser close
```

Expected: Screenshot saved, text is "Example Domain".

**Step 3: Clean up test file**

```bash
rm docs/plans/example-browser-qa-task.md
rm /tmp/test-browser-qa.png 2>/dev/null || true
```

**Step 4: Final commit**

```bash
git add -A
git commit -m "feat: complete browser QA integration"
```

---

## Summary

After completing all tasks:

1. **browser-qa skill** — Reference documentation for the format
2. **writing-plans** — Knows to add Browser QA sections for UI tasks
3. **executing-plans** — Can execute Browser QA during implementation
4. **code-reviewer** — Runs Browser QA checks as part of review

The flow: Plan specifies Browser QA → Executor/Reviewer reads it → Uses agent-browser to verify → Reports results.
