---
name: chore
description: "Commit non-functional infrastructure changes separately from feature work. Use for skill updates, tooling changes, documentation. Triggers on: chore, commit infra, commit tooling, infrastructure commit."
---

# Infrastructure Commit Helper

Commit non-functional/infrastructure changes with a generated conventional commit message.

---

## The Job

1. Get list of uncommitted changes
2. Categorize files as infrastructure, documentation, or feature
3. Generate appropriate commit message (`chore:` for infra, `docs:` for documentation)
4. Ask for confirmation and commit

---

## File Categorization

### Infrastructure Files (commit with `chore:`)

These are development tooling and configuration:

```
.claude/skills/*          # Claude skills
.claude/plans/*           # Planning files
ralph.sh                  # Ralph runner
prompt.md                 # Ralph prompt
prd.json                  # Active Ralph work file (root level)
progress.txt              # Ralph progress log
CLAUDE.md                 # Project instructions
AGENTS.md                 # Agent patterns
Makefile                  # Build tooling
*.md in root              # Documentation (README, etc.)
archive/*                 # Archived PRDs (includes prd.json copies)
.gitignore                # Git config
pyproject.toml            # Only if no src changes
```

**Note:** `prd.json` in the project root is the active Ralph work file, NOT feature code. It defines what Ralph will build, not the build itself.

### Documentation Files (commit with `docs:`)

These are planning and documentation artifacts:

```
tasks/*.md                # PRD files
tasks/roadmap.md          # Roadmap
```

### Feature Files (commit with `/finish-ralph`)

These are application code:

```
packages/*                # Core packages
apps/*                    # Applications
src/*                     # Source code
tests/*                   # Test files
migrations/*              # Database migrations
```

---

## Steps

### Step 1: Get Changed Files

```bash
git status --porcelain
```

Parse output to get list of changed files with their status:
- `A` or `??` = Added (new file)
- `M` = Modified
- `D` = Deleted

### Step 2: Categorize Files

For each file, determine its category based on path patterns above:
- **Documentation**: `tasks/*.md` files → use `docs:` prefix
- **Infrastructure**: skills, config, tooling → use `chore:` prefix
- **Feature**: application code → redirect to `/finish-ralph`

### Step 3: Handle by Category

**All Documentation:**
1. Generate commit message with `docs:` prefix
2. Show file list and message
3. Ask for confirmation
4. Commit

**All Infrastructure:**
1. Generate commit message with `chore:` prefix
2. Show file list and message
3. Ask for confirmation
4. Commit

**Documentation + Infrastructure (no feature):**
1. Group by category
2. Generate appropriate commit message (e.g., "docs: add PRD" or combined message)
3. Show file list and message
4. Ask for confirmation
5. Commit all together

**All Feature:**
```
## Feature Files Detected

These files should be committed as part of a feature:
  packages/core/src/schemas/new.py
  apps/api/src/routes/new.py

Use /finish-ralph to commit feature changes after Ralph completes.
```

**Mixed (includes feature files):**
```
## Mixed Changes Detected

Documentation (2 files):
  tasks/prd-local-dev.md
  tasks/roadmap.md

Infrastructure (1 file):
  .claude/skills/chore/skill.md

Feature (2 files):
  packages/core/src/schemas/new.py
  apps/api/src/routes/new.py

How would you like to proceed?
```

Use AskUserQuestion with options:
- "Commit docs + infra only" - Commit non-feature files, leave feature files
- "Cancel" - Exit without committing

### Step 4: Generate Commit Message

Analyze the infrastructure changes and generate a descriptive message:

**Pattern:** `chore: <summary of changes>`

**Examples:**
- Single skill: `chore: add /workflow skill for state detection`
- Multiple skills: `chore: add workflow skills (/workflow, /ralph-ready, /post-ralph)`
- Mixed infra: `chore: streamline Ralph workflow and update documentation`
- Config only: `chore: update Makefile with new targets`

### Step 5: Commit

```bash
git add <infrastructure-files>
git commit -m "<generated-message>"
```

Show result:
```
## Committed

Commit: <hash>
Message: chore: <message>
Files: <count> files changed

Run /workflow to continue.
```

---

## Commit Message Generation

Based on what changed, generate an appropriate message:

### Documentation (`docs:` prefix)

| Changes | Message Pattern |
|---------|-----------------|
| New PRD file | `docs: add <feature-name> PRD` |
| Modified PRD file | `docs: update <feature-name> PRD` |
| Roadmap changes | `docs: update roadmap` |
| Multiple PRDs | `docs: add <feature-1> and <feature-2> PRDs` |

### Infrastructure (`chore:` prefix)

| Changes | Message Pattern |
|---------|-----------------|
| New skill(s) | `chore: add /<skill-name> skill` |
| Modified skill(s) | `chore: update /<skill-name> skill` |
| prd.json changes | `chore: set up <feature-name> for Ralph` |
| ralph.sh changes | `chore: update ralph.sh` |
| Root documentation | `chore: update documentation` |
| Multiple categories | `chore: <primary change> and <secondary>` |
| Archive changes | `chore: archive <feature-name>` |

### Mixed (docs + infra)

When both documentation and infrastructure changed, prioritize by what's more significant:
- If PRDs are the main change: `docs: add <feature> PRD`
- If skills are the main change: `chore: add /<skill> skill`

---

## Error Handling

| Condition | Response |
|-----------|----------|
| No changes | "No uncommitted changes. Nothing to commit." |
| All feature files | "These are feature files. Use /finish-ralph instead." |
| Git not clean after commit | "Commit succeeded but some files remain. Run /chore again." |
| Commit fails | Show git error message |

---

## Checklist

Before committing:

- [ ] Got list of changed files
- [ ] Categorized all files correctly
- [ ] Generated appropriate commit message
- [ ] Got user confirmation
- [ ] Committed successfully
- [ ] Showed result summary
