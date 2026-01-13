---
name: create-pr
description: "Create a GitHub pull request after completing a Ralph run. Use when you've finished implementing stories and want to submit a PR. Triggers on: create pr, make pull request, open pr, submit pr, create a pull request."
---

# Create Pull Request

Automate GitHub PR creation from your current branch after completing Ralph iterations.

---

## The Job

1. Verify you're on a feature branch (not main/master)
2. Check that prd.json exists and read it
3. Push the branch to origin (if needed)
4. Ensure base branch (master) exists on remote
5. Create a PR with title and body derived from prd.json
6. Report the PR URL

---

## Prerequisites

Before running, verify:
- Current branch is a feature branch (ralph/* or feat/*)
- Not on main or master
- prd.json exists in the project root
- Git working tree is clean (changes committed)
- `gh` CLI is authenticated

---

## Steps

### Step 1: Verify Environment

Run these checks:

```bash
# Get current branch
git branch --show-current

# Verify not on main/master
git branch --show-current | grep -qvE '^(main|master)$'

# Check for uncommitted changes
git status --porcelain
```

If on main/master, stop and tell the user to switch branches.
If there are uncommitted changes, warn the user and ask if they want to commit first.

### Step 2: Read prd.json

Read `prd.json` from the project root. Extract:
- `branchName`: Validate it matches current branch
- `description`: Use for PR title
- `userStories`: Build summary from completed stories (where `passes: true`)

### Step 3: Push Branch to Origin

Check if the branch needs to be pushed:

```bash
# Check if branch has upstream
git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null

# Check if local is ahead of remote
git status -sb
```

Push the branch if needed:

```bash
# If no upstream, push with -u to set tracking
git push -u origin <branch-name>

# If upstream exists but local is ahead, just push
git push
```

If push fails, stop and report the error to the user.

### Step 4: Ensure Base Branch Exists on Remote

Check if `master` exists on the remote:

```bash
git ls-remote --heads origin master
```

If the output is empty, `master` doesn't exist on the remote. This happens with new repos. Push it:

```bash
git push -u origin master
```

If `master` doesn't exist locally either, stop and tell the user to create a base branch first.

### Step 5: Build PR Content

**Title format:**
```
feat: <description from prd.json>
```

**Body format:**
```markdown
## Summary
<description from prd.json>

## Completed Stories
- US-001: <title>
- US-002: <title>
(list all stories where passes: true)

## Test plan
- [ ] All acceptance criteria verified
- [ ] Typecheck passes
- [ ] Tests pass

Generated with [Claude Code](https://claude.com/claude-code)
```

### Step 6: Create PR

Use the GitHub CLI with proper escaping:

```bash
gh pr create --base master --head <branch-name> \
  --title "<title>" \
  --body "<body>"
```

**Important:** Use a HEREDOC for the body to avoid shell escaping issues:

```bash
gh pr create --base master --head ralph/terminology \
  --title "feat: Terminology Services" \
  --body "$(cat <<'EOF'
## Summary
...body content...

Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

### Step 7: Report Results

After successful creation, display:
- PR number and URL
- Branch: `<branch-name> -> master`
- Next steps (review, merge)

---

## Error Handling

- **Not on feature branch:** Tell user to switch to a feature branch
- **No prd.json:** Tell user to run `/prd-json` first to create a prd.json
- **Uncommitted changes:** Ask if user wants to commit first
- **Push fails:** Report the git error (auth issues, conflicts, etc.)
- **Base branch missing on remote:** Push master to origin (new repo case)
- **Base branch missing locally:** Tell user to create a base branch first
- **gh not authenticated:** Tell user to run `gh auth login`
- **PR already exists:** Show existing PR URL instead

---

## Checklist

Before completing:

- [ ] Verified on feature branch (not main/master)
- [ ] prd.json exists and was read
- [ ] Branch pushed to origin
- [ ] Base branch (master) exists on remote
- [ ] PR title includes "feat:" prefix
- [ ] PR body includes completed stories
- [ ] PR was created successfully
- [ ] PR URL was displayed to user
