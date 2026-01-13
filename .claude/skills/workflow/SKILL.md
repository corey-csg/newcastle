---
name: workflow
description: "Show current Ralph workflow state and next action. Use anytime to see where you are in the development cycle. Triggers on: workflow, where am i, what next, ralph status, current state."
---

# Ralph Workflow Guide

Detect the current state of the repository and show what to do next.

---

## The Job

1. Check for prd.json and its status
2. Check git branch status
3. Determine current workflow state
4. Display state and next action

---

## State Detection

Check these conditions in order:

### 1. Check prd.json Existence

```bash
test -f prd.json
```

### 2. If prd.json Exists, Read Status

Extract from prd.json:
- `branchName`
- `description`
- Count stories with `passes: true`
- Count stories with `passes: false`

### 3. Check Git Branch

```bash
git branch --show-current
```

### 4. Check for Uncommitted Changes

```bash
git status --porcelain
```

---

## States and Next Actions

### State: No prd.json, On Master

**Detection:**
- prd.json does not exist
- On master branch

**Output:**
```
## Workflow State: Ready for New Feature

Branch: master
PRD: None

## Next Steps
1. Write your PRD in tasks/prd-<feature-name>.md
2. Run /prd to convert it to prd.json
3. Run /ralph-ready to verify setup
4. Run ./ralph.sh to start

Quick start:
  /prd
```

---

### State: prd.json Exists, 0 Stories Complete

**Detection:**
- prd.json exists
- All stories have `passes: false`

**Output:**
```
## Workflow State: Ready for Ralph

Branch: <branchName>
PRD: "<description>"
Stories: 0/<total> complete

## Next Steps
1. Run /ralph-ready to verify setup
2. Run ./ralph.sh to start Ralph

Quick start:
  /ralph-ready
```

---

### State: prd.json Exists, Partial Completion

**Detection:**
- prd.json exists
- Some stories have `passes: true`, some have `passes: false`

**Output:**
```
## Workflow State: Ralph In Progress

Branch: <branchName>
PRD: "<description>"
Stories: <passed>/<total> complete (<remaining> remaining)

## Next Steps
Ralph was interrupted or is still running.
- If Ralph is running: wait for completion
- If interrupted: run ./ralph.sh to continue

Quick start:
  ./ralph.sh
```

---

### State: prd.json Exists, All Stories Complete

**Detection:**
- prd.json exists
- All stories have `passes: true`

**Output:**
```
## Workflow State: Ralph Complete - Ready for Review

Branch: <branchName>
PRD: "<description>"
Stories: <total>/<total> complete

## Next Steps
1. Run /post-ralph to verify tests pass
2. Run /finish-ralph to merge and archive

Quick start:
  /post-ralph
```

---

### State: Uncommitted Changes

**Detection:**
- Git status shows uncommitted changes
- (Any prd.json state)

**Output:**
```
## Workflow State: Uncommitted Changes

Branch: <current-branch>
Changes: <count> files modified

## Action Required
Commit or stash your changes before proceeding:
  git add . && git commit -m "your message"

Or stash temporarily:
  git stash

Then run /workflow again.
```

---

### State: Wrong Branch

**Detection:**
- prd.json exists
- Current branch doesn't match prd.json branchName

**Output:**
```
## Workflow State: Wrong Branch

Current: <current-branch>
Expected: <branchName from prd.json>

## Action Required
Switch to the correct branch:
  git checkout <branchName>

Then run /workflow again.
```

---

## Complete Workflow Reference

```
/pre-prd → /prd <feature> → /prd-json → /ralph-ready → ./ralph.sh → /post-ralph → /finish-ralph → /workflow
    │                                                                                                    │
    └────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

| Step | Command | Purpose |
|------|---------|---------|
| 1 | /pre-prd | Load context, see available PRDs |
| 2 | /prd \<feature\> | Write PRD with clarifying questions |
| 3 | /prd-json | Convert finished PRD to prd.json |
| 4 | /ralph-ready | Verify setup before Ralph |
| 5 | ./ralph.sh | Run Ralph autonomous agent |
| 6 | /post-ralph | Run verification tests |
| 7 | /finish-ralph | Merge & archive |
| 8 | /workflow | See current state, loop back |

---

## Checklist

Before showing state:

- [ ] Checked for prd.json
- [ ] If exists, read and parsed prd.json
- [ ] Got current git branch
- [ ] Checked for uncommitted changes
- [ ] Displayed appropriate state and next action
