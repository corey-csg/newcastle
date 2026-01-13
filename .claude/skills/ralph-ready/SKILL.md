---
name: ralph-ready
description: "Pre-flight check before running ralph.sh. Verifies repo state and shows summary. Triggers on: ralph ready, pre ralph, before ralph, check ralph, ready for ralph."
---

# Ralph Pre-flight Check

Verify the repository is ready for a Ralph run and show a summary of what will be worked on.

---

## The Job

1. Check that prd.json exists and is valid
2. Check git status - auto-commit PRD files if needed
3. Verify on main/master branch (Ralph will create feature branch)
4. Check story status (should have stories to work on)
5. Display summary and next steps

---

## Steps

### Step 1: Check prd.json Exists

```bash
# Check if prd.json exists
test -f prd.json
```

If missing, stop with error: "No prd.json found. Run /prd to create one."

### Step 2: Validate prd.json Structure

Read prd.json and extract:
- `branchName`: Expected branch
- `description`: Feature description
- `userStories`: Array of stories

Verify JSON is valid and has required fields.

### Step 2b: Validate Story Criteria by Type

For each story, verify it has the required acceptance criteria for its type. This ensures Ralph can properly verify stories.

**Required criteria by type:**

| Type | Required Criteria (at least one match) |
|------|----------------------------------------|
| `ui` | Contains "Playwright" or "E2E test", AND "visual" or "Visual" |
| `backend` | Contains "pytest" or "pyright" |
| `integration` | Contains "E2E test" or "Playwright", AND "API" |
| `data` | Contains "migration" (case-insensitive), AND "schema" or "validation" |
| `devops` | Contains "script" or "executes" or "valid markdown" |

**Note:** Also check that non-devops stories have "Typecheck passes" or equivalent.

**Validation logic:**

```
for each story in userStories:
  if story.type is missing:
    mark as NEEDS_FIX (missing type)
  else:
    check story.acceptanceCriteria against required criteria for story.type
    if missing required criteria:
      mark as NEEDS_FIX (missing criteria)
```

**If any stories need fixing:**
1. Show which stories failed validation and why
2. Automatically call `/prd-json --fix` to repair
3. Re-read the updated prd.json
4. Re-validate to confirm fix worked
5. If still failing after fix, show error and block

**Validation output (before fix):**

```
## Story Validation

Checking stories have required criteria for their type...

✗ US-001 "Add status field" (data)
  Missing: migration criterion, schema criterion

✗ US-003 "Create dashboard" (ui)
  Missing: type tag (will auto-detect)

✓ US-002 "Add API endpoint" (backend)

2 stories need fixing. Running /prd-json --fix...
```

**After successful fix:**

```
Running /prd-json --fix...

Fixed 2 stories:
  US-001: Added migration + schema criteria
  US-003: Added type (ui), Playwright test, visual verification

Re-validating... ✓ All stories valid

Continuing pre-flight checks...
```

**If fix fails (edge case):**

```
Running /prd-json --fix...

Re-validating...
✗ US-001 "Add status field" still missing: migration criterion

Error: Auto-fix could not resolve all issues.
Please manually check prd.json and run /ralph-ready again.
```

### Step 3: Check Git Status and Auto-Commit PRD Files

```bash
# Check for uncommitted changes
git status --porcelain
```

**If there are uncommitted changes:**

1. Categorize each file as PRD-related or not:
   - **PRD-related patterns** (safe to auto-commit):
     - `prd.json`
     - `tasks/prd-*.md`
     - `tasks/roadmap.md`
     - `resources/*`
   - **Non-PRD files** (require manual handling):
     - Everything else (apps/*, packages/*, src/*, etc.)

2. **If ALL uncommitted files are PRD-related:**
   - Extract feature name from prd.json branchName (e.g., `ralph/instrument-browser` → `instrument-browser`)
   - Show files to be committed
   - Auto-commit with message: `docs: add <feature> PRD and prd.json`
   - Continue to Step 4

3. **If ANY non-PRD files are present:**
   - List PRD files and non-PRD files separately
   - Error: "Non-PRD files detected. Commit or stash these first."

**Auto-commit output:**
```
## PRD Files Detected

The following PRD-related files will be committed:
  prd.json
  tasks/prd-instrument-browser.md
  tasks/roadmap.md
  resources/Clay-Street-Logo.jpg

Committing: "docs: add instrument-browser PRD and prd.json"

✓ Committed: abc1234

Continuing pre-flight checks...
```

**Mixed files error output:**
```
## Mixed Uncommitted Changes

PRD files (will auto-commit):
  prd.json
  tasks/prd-instrument-browser.md

Non-PRD files (require manual handling):
  apps/web/src/App.tsx
  packages/core/schema.py

Action required: Commit or stash the non-PRD files first, then run /ralph-ready again.
```

### Step 4: Verify Branch

```bash
# Get current branch
git branch --show-current
```

**Important:** Ralph creates the feature branch from main/master. The user should be on main/master when running ralph.sh.

**Branch check logic:**
- If on `main` or `master` → OK, Ralph will create/checkout the feature branch
- If on the feature branch (matches prd.json branchName) → OK, Ralph can continue
- If on a different branch → Warning: "On branch X. Ralph expects to branch from main/master."

**Note:** This is a soft check. Ralph's prompt.md instructs it to checkout/create the correct branch, so it will handle this regardless.

### Step 5: Check Story Status

Count stories by status:
- Total stories
- Stories with `passes: false` (remaining work)
- Stories with `passes: true` (already complete)

**Status interpretation:**
- All `passes: false` → Ready for fresh Ralph run
- Some `passes: true`, some `passes: false` → Ralph was interrupted, can continue
- All `passes: true` → Ralph already complete, run /post-ralph and /finish-ralph

### Step 6: Display Summary

**Success output (no uncommitted changes):**
```
## Ralph Pre-flight Check

✓ prd.json valid
✓ Story criteria valid (all stories have required criteria for type)
✓ Git status: clean
✓ Branch: master (Ralph will create ralph/feature-name)
✓ Stories: 0/12 complete (12 remaining)

## Summary
Feature: "Feature Description"
Branch: ralph/feature-name
Stories to complete: 12

## Ready to Run
./ralph.sh
```

**Success output (after auto-commit):**
```
## Ralph Pre-flight Check

✓ prd.json valid
✓ Story criteria valid (all stories have required criteria for type)
✓ PRD files committed: "docs: add feature-name PRD and prd.json"
✓ Branch: master (Ralph will create ralph/feature-name)
✓ Stories: 0/12 complete (12 remaining)

## Summary
Feature: "Feature Description"
Branch: ralph/feature-name
Stories to complete: 12

## Ready to Run
./ralph.sh
```

**Success output (after auto-fix):**
```
## Ralph Pre-flight Check

✓ prd.json valid
✓ Story criteria fixed (2 stories updated via /prd-json --fix)
✓ Git status: clean
✓ Branch: master (Ralph will create ralph/feature-name)
✓ Stories: 0/12 complete (12 remaining)

## Summary
Feature: "Feature Description"
Branch: ralph/feature-name
Stories to complete: 12

## Ready to Run
./ralph.sh
```

**If all stories complete:**
```
## Ralph Pre-flight Check

✓ prd.json valid
✓ Git status: clean
✓ Branch: ralph/feature-name (matches prd.json)
! Stories: 12/12 complete (0 remaining)

## Ralph Already Complete!
All stories have passed. Next steps:
1. Run /post-ralph to verify tests
2. Run /finish-ralph to merge and archive
```

---

## Error Messages

| Condition | Message |
|-----------|---------|
| No prd.json | "No prd.json found. Run /prd to create one from your PRD markdown." |
| Invalid JSON | "prd.json is not valid JSON. Check for syntax errors." |
| Missing branchName | "prd.json missing branchName field." |
| Stories missing type/criteria | (auto-fix triggered - runs `/prd-json --fix` automatically) |
| Auto-fix failed | "Auto-fix could not resolve all issues. Please manually check prd.json." |
| PRD files only | (auto-commit, no error - show success message) |
| Mixed uncommitted | "Non-PRD files detected. Commit or stash these first: <list>" |
| Wrong branch | "On branch X. Consider switching to main/master before running Ralph." (soft warning) |
| All stories done | "All stories complete! Run /post-ralph then /finish-ralph." |

---

## Checklist

Before showing "Ready to Run":

- [ ] prd.json exists
- [ ] prd.json is valid JSON with branchName, description, userStories
- [ ] All stories have type tags (auto-fix if missing)
- [ ] All stories have required criteria for their type (auto-fix if missing)
- [ ] Git working tree is clean OR only PRD files uncommitted (auto-commit them)
- [ ] On main/master branch OR feature branch (soft check - Ralph handles this)
- [ ] At least one story has passes: false
