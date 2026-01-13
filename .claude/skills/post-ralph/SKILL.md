---
name: post-ralph
description: "Run verification tests after Ralph completes. Extracts test commands from progress.txt and Playwright tests from prd.json. Triggers on: post ralph, after ralph, verify ralph, run tests, test ralph."
---

# Post-Ralph Test Runner

Extract and run verification tests after Ralph completes all stories.

---

## The Job

1. Verify Ralph has completed (all stories passed)
2. Extract test commands from progress.txt
3. **Extract Playwright tests from prd.json acceptance criteria**
4. Present commands to user for approval
5. Run approved commands and show results
6. Report overall pass/fail status
7. **Block /finish-ralph if Playwright tests fail**

---

## Prerequisites

Before running:
- prd.json exists with all stories `passes: true`
- progress.txt exists with Ralph's output

---

## Steps

### Step 1: Verify Ralph Complete

Read prd.json and check that all stories have `passes: true`.

If not all complete:
```
Ralph hasn't finished yet.
Stories: X/Y complete

Run ./ralph.sh to continue, or /workflow to see status.
```

### Step 2: Extract Test Commands from progress.txt

Read progress.txt and look for test commands. Common patterns:

```
# Typecheck commands
uv run pyright <path>
uv run mypy <path>

# Test commands
uv run pytest <path>
pytest <path>

# Migration commands
uv run alembic upgrade head
cd apps/api && uv run alembic upgrade head

# Build commands
make test
make lint
make typecheck

# Manual verification notes
"Verify in browser"
```

Search for lines containing:
- `uv run pytest`
- `uv run pyright`
- `uv run mypy`
- `uv run alembic`
- `make test`
- `make lint`
- `make typecheck`
- `pytest`

Deduplicate commands (same command may appear multiple times).

### Step 3: Extract Playwright Tests from prd.json

Read prd.json and scan all acceptance criteria for Playwright test references.

**Pattern to match:**
```
Playwright E2E test passes: {filename}.spec.ts
```

**Extraction rules:**
1. For each story in `userStories`, iterate through `acceptanceCriteria`
2. Match criteria containing "Playwright E2E test passes:" (case-insensitive)
3. Extract the test filename (e.g., `visual-verify.spec.ts`)
4. Deduplicate filenames (same test may appear in multiple stories)

**Build command:**
```bash
cd apps/web && npx playwright test {file1}.spec.ts {file2}.spec.ts ...
```

**Example:**
```
prd.json criteria:
- "Playwright E2E test passes: visual-verify.spec.ts"
- "Playwright E2E test passes: visual-integration.spec.ts"

Extracted files: visual-verify.spec.ts, visual-integration.spec.ts

Command: cd apps/web && npx playwright test visual-verify.spec.ts visual-integration.spec.ts
```

If no Playwright tests found, skip this step (no command generated).

### Step 4: Present Commands

Show the extracted commands:

```
## Post-Ralph Test Verification

Found X test commands from progress.txt:
1. uv run pyright packages/core
2. uv run pytest packages/core/tests/
3. cd apps/api && uv run alembic upgrade head

Found X Playwright tests from prd.json:
4. cd apps/web && npx playwright test visual-verify.spec.ts visual-integration.spec.ts

Run these tests?
```

Use AskUserQuestion to get approval:
- "Yes, run all" - Run all commands
- "Skip" - Skip to /finish-ralph (user takes responsibility)

### Step 5: Run Commands

For each approved command:

1. Show: `Running: <command>`
2. Execute the command
3. Capture output and exit code
4. Show result:
   - Success: `✓ Passed`
   - Failure: `✗ Failed` with error output
5. Track which commands are Playwright tests (for blocking logic)

```
Running: uv run pyright packages/core
✓ Passed (0 errors)

Running: uv run pytest packages/core/tests/
✓ Passed (24 tests, 0 failures)

Running: cd apps/api && uv run alembic upgrade head
✓ Passed (migration successful)

Running: cd apps/web && npx playwright test visual-verify.spec.ts
✓ Passed (1 test passed)
```

### Step 6: Handle Failures

If any command fails:

```
## Test Results

✓ uv run pyright packages/core - Passed
✗ uv run pytest packages/core/tests/ - FAILED

Error output:
<show relevant error output>

## Action Required
Fix the failing test before proceeding to /finish-ralph.

Options:
1. Fix the issue and run /post-ralph again
2. Run /finish-ralph anyway (not recommended)
```

**Special handling for Playwright test failures:**

If any Playwright test fails, `/finish-ralph` MUST be blocked:

```
## Playwright Test Failure

✗ cd apps/web && npx playwright test visual-verify.spec.ts - FAILED

Error output:
<show relevant error output>

## BLOCKING: /finish-ralph is blocked

Playwright tests are required for UI verification.
You MUST fix the failing tests before completing the Ralph run.

Options:
1. Fix the tests and run /post-ralph again
2. Skip Playwright verification (requires explicit override)
```

Use AskUserQuestion:
- "I'll fix it" - End skill, user fixes (recommended)
- "Proceed anyway" - Allow /finish-ralph with warning (NOT recommended for Playwright failures)
- "Override Playwright block" - Force allow /finish-ralph (requires explicit user acknowledgment)

**Blocking mechanism:**

When Playwright tests fail:
1. Report the failure clearly with test output
2. Tell user `/finish-ralph` is blocked
3. Only allow override if user explicitly selects "Override Playwright block"
4. If overriding, warn: "Skipping Playwright verification - UI issues may not be caught"

### Step 7: Report Success

If all commands pass:

```
## Test Results

All tests passed!

✓ uv run pyright packages/core
✓ uv run pytest packages/core/tests/
✓ cd apps/api && uv run alembic upgrade head
✓ cd apps/web && npx playwright test visual-verify.spec.ts

## Ready for Completion
Run /finish-ralph to merge and archive this feature.
```

---

## No Tests Found

If no test commands found in progress.txt AND no Playwright tests in prd.json:

```
## Post-Ralph Test Verification

No test commands found in progress.txt.
No Playwright tests found in prd.json.

This could mean:
- Ralph didn't run any tests (unusual)
- Tests were run but not logged
- No UI stories in this PRD

Options:
1. Run standard tests manually
2. Proceed to /finish-ralph

Suggested manual tests:
  uv run pyright packages/core
  uv run pytest packages/core/tests/
```

Use AskUserQuestion:
- "Run suggested tests" - Run the suggested commands
- "Skip to /finish-ralph" - Proceed without testing

**Note:** If Playwright tests ARE found in prd.json but no progress.txt commands, still run the Playwright tests (they are required for UI stories).

---

## Error Messages

| Condition | Message |
|-----------|---------|
| No prd.json | "No prd.json found. Run /workflow to see current state." |
| Stories incomplete | "Ralph hasn't finished. X/Y stories complete. Run ./ralph.sh to continue." |
| No progress.txt | "No progress.txt found. Run ./ralph.sh first." |
| Command fails | "Test failed: <command>. Fix the issue or proceed with caution." |
| Playwright fails | "Playwright test failed: <file>. /finish-ralph is BLOCKED until fixed." |
| Playwright not installed | "Playwright not found. Run: cd apps/web && npx playwright install chromium" |

---

## Checklist

Before showing "Ready for Completion":

- [ ] prd.json exists with all stories passed
- [ ] progress.txt exists
- [ ] Extracted test commands from progress.txt (or noted none found)
- [ ] Extracted Playwright tests from prd.json acceptance criteria
- [ ] Got user approval to run tests
- [ ] Ran all approved commands
- [ ] All commands passed (or user chose to proceed anyway)
- [ ] No Playwright tests failed (or user explicitly overrode block)
