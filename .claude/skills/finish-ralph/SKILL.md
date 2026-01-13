---
name: finish-ralph
description: "Complete a Ralph run: push, create PR, merge, and prepare for next iteration. Use after Ralph completes all stories. Triggers on: finish ralph, complete ralph, wrap up ralph, merge and continue."
---

# Finish Ralph Run

Complete the current Ralph iteration by pushing, creating a PR, merging it, and preparing for the next feature.

---

## The Job

1. Verify on a ralph/* feature branch with clean working tree
2. Push the branch to origin
3. Ensure base branch (master) exists on remote
4. Create a PR from prd.json context
5. Run AI code review (via PR-Agent)
6. Generate review PRD if issues found
7. Merge the PR
8. Checkout master and pull the merged changes
9. Archive the completed feature (prd.json + progress.txt)
10. Delete the local feature branch (cleanup)
11. Report completion (including review summary)

---

## Prerequisites

Before running, verify:
- Current branch is `ralph/*` (not main/master)
- prd.json exists in the project root
- Git working tree is clean (all changes committed)
- `gh` CLI is authenticated

---

## Security Best Practices

**API Key Handling:**
- NEVER log API keys (ANTHROPIC_API_KEY, GH_TOKEN, etc.) in any output
- Check for key presence using `grep -q` (returns exit code, not key value)
- When logging errors, only include: PR URL, error type, exit code
- Use environment variable sourcing (`source .env`) without echoing values

**Credential Safety:**
- GitHub tokens come from `gh auth token` - never log this value
- If an operation fails, log the operation name and error code, not the credentials used
- Error messages should help debug without exposing secrets

**Example - Safe error logging:**
```bash
# GOOD: Log context without secrets
echo "Error: PR-Agent failed (exit code $EXIT_CODE)"
echo "PR URL: $PR_URL"

# BAD: Never do this
echo "Failed with API key: $ANTHROPIC_API_KEY"  # NEVER!
echo "GH_TOKEN was: $GH_TOKEN"  # NEVER!
```

---

## Steps

### Step 1: Verify Environment

Run these checks:

```bash
# Get current branch
git branch --show-current

# Verify it's a ralph/* branch
git branch --show-current | grep -q '^ralph/'

# Check for uncommitted changes
git status --porcelain
```

If not on a ralph/* branch, stop and tell the user.
If there are uncommitted changes, warn the user and ask if they want to commit first.

### Step 2: Read prd.json

Read `prd.json` from the project root. Extract:
- `branchName`: Current branch name
- `description`: Feature description for PR title
- `userStories`: Build summary from completed stories (where `passes: true`)

Verify all stories have `passes: true`. If not, warn that Ralph isn't complete yet and ask if they want to proceed anyway.

### Step 3: Push Branch to Origin

```bash
# Check if branch has upstream and push
git push -u origin <branch-name>
```

### Step 4: Ensure Base Branch Exists on Remote

```bash
git ls-remote --heads origin master
```

If empty, push master first:
```bash
git push -u origin master
```

### Step 5: Create PR

Check if PR already exists:
```bash
gh pr view --json number,url 2>/dev/null
```

If PR exists, use it. Otherwise create:

```bash
gh pr create --base master --head <branch-name> \
  --title "feat: <description>" \
  --body "$(cat <<'EOF'
## Summary
<description from prd.json>

## Completed Stories
- US-001: <title>
- US-002: <title>
...

## Test plan
- [x] All acceptance criteria verified
- [x] Typecheck passes
- [x] Tests pass

Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

### Step 5.5: Run Code Review

After the PR is created, run AI-powered code review using PR-Agent.

**Important:** Code review is advisory only. Failures never block the merge. All errors are logged with clear messages, and the workflow continues.

#### 5.5.1: Pre-flight Checks

Before running PR-Agent, verify the environment.

**Important:** Always use the git repository root for file paths. The working directory may change during the workflow (e.g., after running tests in `apps/web/`).

```bash
# Get the git repository root (use this for all file paths)
PROJECT_ROOT=$(git rev-parse --show-toplevel)

# Check if .env exists at project root
if [ ! -f "$PROJECT_ROOT/.env" ]; then
  echo "Warning: .env file not found. Skipping code review."
  # Continue to merge - don't exit
fi

# Check for ANTHROPIC_API_KEY (don't log the actual key!)
if ! grep -q '^ANTHROPIC_API_KEY=' "$PROJECT_ROOT/.env" 2>/dev/null; then
  echo "Warning: ANTHROPIC_API_KEY not found in .env. Skipping code review."
  # Continue to merge - don't exit
fi
```

If either check fails, **skip the entire code review step** and proceed to Step 6 (Merge PR). Log a clear warning message.

#### 5.5.2: Get PR URL

```bash
PR_URL=$(gh pr view --json url -q '.url')
if [ -z "$PR_URL" ]; then
  echo "Warning: Could not get PR URL. Skipping code review."
  # Continue to merge
fi
```

#### 5.5.3: Run PR-Agent Review

```bash
# Load environment variables from project root and run PR-Agent with Claude
# Note: uvx runs in isolation, so we must pass model via CLI flag
PROJECT_ROOT=$(git rev-parse --show-toplevel)
set -a && source "$PROJECT_ROOT/.env" && set +a
GH_TOKEN=$(gh auth token)

# Run with timeout and capture both stdout and stderr
REVIEW_OUTPUT=$(timeout 120 GITHUB__USER_TOKEN="$GH_TOKEN" uvx pr-agent \
  --pr_url "$PR_URL" review \
  --config.model="anthropic/claude-sonnet-4-20250514" 2>&1) || {
  EXIT_CODE=$?
  if [ $EXIT_CODE -eq 124 ]; then
    echo "Warning: PR-Agent timed out after 120 seconds. Skipping code review."
  else
    echo "Warning: PR-Agent failed (exit code $EXIT_CODE). Skipping code review."
  fi
  # Log error context (but never API keys!)
  echo "PR URL: $PR_URL"
  # Continue to merge - don't exit
}
```

**Error handling behavior:**
- **Timeout (120s):** Log warning, skip review, continue to merge
- **Network errors:** Log warning with PR URL, skip review, continue
- **API errors (rate limits, auth failures):** Log warning, skip review, continue
- **Any other failure:** Log exit code and PR URL, skip review, continue

#### 5.5.4: Parse Review Findings

Only attempt to parse if we have output:

```bash
if [ -n "$REVIEW_OUTPUT" ]; then
  # Parse the output - look for structured findings
  # PR-Agent outputs findings in markdown format
fi
```

If parsing fails (malformed output, unexpected format):
- Log: "Warning: Could not parse PR-Agent output. Review may have been posted to PR directly."
- Check the PR comments manually if needed
- Continue to merge

Categorize any successfully parsed findings:
- **Critical:** Security vulnerabilities, data loss risks, breaking changes
- **Important:** Logic errors, missing error handling, performance issues
- **Minor:** Style issues, documentation gaps, minor improvements

#### 5.5.5: Generate Review PRD if Needed

If there are any Critical or Important findings:

a. Create `tasks/prd-review-YYYY-MM-DD.md` with format:
```markdown
# Code Review Fixes

Category: review
Source Feature: <feature-name from prd.json branchName>
Review Date: YYYY-MM-DD

## Introduction
Automated code review findings from <feature> that should be addressed.

## Goals
- Address code quality issues identified by AI review
- Improve codebase maintainability

## User Stories

### REV-001: <Finding Title>
**Type:** backend/ui/devops
**Priority:** high/medium/low

<finding description>

**Acceptance Criteria:**
- Fix implemented
- Typecheck passes
```

b. Update `tasks/roadmap.md` - add to Remaining section:
```
| `prd-review-YYYY-MM-DD.md` | review | Review fixes from <feature> | None |
```

**Partial failure handling:** If PRD file creation fails but roadmap update would succeed (or vice versa), continue with what succeeded. A partial record is better than none.

#### 5.5.6: Clean Up Orphaned Stub Comments

PR-Agent uses `persistent_comment = true`, which posts a "Preparing review..." placeholder before analysis. If the review fails (timeout, API error), this stub may remain as an orphaned comment.

**After a failed review, check for orphans:**
```bash
# List PR comments looking for stub text
gh pr view --json comments -q '.comments[] | select(.body | contains("Preparing")) | .id'
```

**If orphaned stub found:**
1. Note: The stub will be automatically updated if review runs successfully on a subsequent attempt
2. For immediate cleanup, manually delete the comment from the PR UI
3. Or re-run the review: `uvx pr-agent --pr_url "$PR_URL" review --config.model="anthropic/claude-sonnet-4-20250514"`

**Important:** Stub cleanup is best-effort. Don't block the merge for orphaned comments - they're cosmetic and can be cleaned later.

#### 5.5.7: Report Review Summary

Always report what happened:
- **If review ran successfully:** "Found X Critical, Y Important, Z Minor issues"
- **If PRD generated:** "Review PRD created: tasks/prd-review-YYYY-MM-DD.md"
- **If no Important/Critical issues:** "No significant issues found. Proceeding with merge."
- **If review was skipped:** State reason (no API key, timeout, network error)
- **If review failed after starting:** "Code review encountered an error. Check PR comments for any posted feedback. Orphaned 'Preparing review...' comments can be deleted manually."

**Note:** Critical issues show a warning but don't block merge. The generated PRD queues the work for a future Ralph iteration.

### Step 6: Merge PR

Merge the PR using the merge strategy (not squash, to preserve commits):

```bash
gh pr merge --merge
```

If merge fails (conflicts, checks pending), report the error and stop.

### Step 7: Checkout Master and Pull

```bash
git checkout master
git pull
```

### Step 8: Archive Completed Feature

Archive the completed PRD and progress to preserve the record:

1. Extract feature name from prd.json's branchName (e.g., `ralph/fhir-import` → `fhir-import`)
2. Create archive folder with today's date: `archive/YYYY-MM-DD-<feature-name>/`
3. Copy files to archive:

```bash
# Get today's date
DATE=$(date +%Y-%m-%d)

# Extract feature name from branchName in prd.json
FEATURE=$(jq -r '.branchName' prd.json | sed 's|^ralph/||')

# Create archive directory
mkdir -p "archive/$DATE-$FEATURE"

# Copy files
cp prd.json "archive/$DATE-$FEATURE/"
cp progress.txt "archive/$DATE-$FEATURE/"
```

4. Clean up root files:

```bash
# Remove prd.json (archived copy is the record)
rm prd.json

# Reset progress.txt for next run
cat > progress.txt << 'EOF'
# Ralph Progress Log
Started:
---
EOF
```

**Important:** Always archive BEFORE deleting the feature branch, so the files are still available.

### Step 9: Delete Local Feature Branch

Clean up the merged feature branch:

```bash
git branch -d <branch-name>
```

Use `-d` (not `-D`) so it fails safely if not fully merged.

### Step 10: Report Completion

Display:
- PR number and URL
- Merge status
- Code review summary (from Step 5.5)
- If review PRD was generated: "Review PRD queued: tasks/prd-review-YYYY-MM-DD.md"
- Archive location (e.g., `archive/2026-01-11-fhir-import/`)
- Current branch (should be master)
- Next steps:
  - If review PRD exists: "Quality fixes available. Run `/workflow` to see review PRD or `/pre-prd` to start next feature."
  - Otherwise: "Ready for next feature. Run `/workflow` to see status."

---

## Error Handling

### Blocking Errors (Stop workflow)
- **Not on ralph/* branch:** Tell user this skill is for Ralph branches
- **No prd.json:** Tell user to run `/prd` first to create prd.json
- **Push fails:** Report git error and stop
- **PR creation fails:** Report gh error and stop
- **Merge fails:** Report reason (conflicts, pending checks) and stop

### Non-Blocking Warnings (Log and continue)
- **Uncommitted changes:** Ask if user wants to commit first
- **Stories incomplete:** Warn and ask to proceed anyway
- **Archive fails:** Warn but continue (PR is merged, archive is secondary)
- **Branch delete fails:** Warn but continue (non-fatal)

### Code Review Errors (Always non-blocking, see Step 5.5)
Code review is advisory only. All errors are logged with clear messages and the workflow continues to merge.

- **No .env file:** Skip review with warning
- **No ANTHROPIC_API_KEY in .env:** Skip review with warning
- **Cannot get PR URL:** Skip review with warning
- **PR-Agent timeout (120s):** Log timeout warning, skip review, check for orphaned stub comments
- **PR-Agent network error:** Log error with PR URL (never log API keys), skip review, check for orphaned stubs
- **PR-Agent API error (rate limit, auth):** Log error type, skip review
- **Malformed PR-Agent output:** Log parsing failure warning, check PR comments manually
- **PRD file creation fails:** Log warning, continue (partial record is better than none)
- **Roadmap update fails:** Log warning, continue

**Security note:** Never log API keys or tokens in error messages. Only log the PR URL and error type/code for debugging.

**Stub comment note:** If review fails after posting "Preparing review..." placeholder, the stub may remain. This is cosmetic - don't block merge. See Step 5.5.6 for cleanup instructions.

---

## Checklist

Before completing:

- [ ] Verified on ralph/* branch
- [ ] prd.json exists and was read
- [ ] Branch pushed to origin
- [ ] PR created (or existing PR found)
- [ ] Pre-flight checks ran (API key check - skip review if missing)
- [ ] Code review attempted (or explicitly skipped with logged reason)
- [ ] Review PRD generated if Important/Critical issues found
- [ ] Roadmap updated if review PRD generated
- [ ] PR merged successfully
- [ ] Checked out master
- [ ] Pulled latest changes
- [ ] Archived prd.json and progress.txt to archive/YYYY-MM-DD-<feature>/
- [ ] Removed root prd.json
- [ ] Reset progress.txt
- [ ] Deleted local feature branch
- [ ] Reported PR URL, code review summary (or skip reason), archive location, and next steps
