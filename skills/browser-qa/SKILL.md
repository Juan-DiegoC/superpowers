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
