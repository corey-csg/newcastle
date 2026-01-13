---
name: pre-prd
description: "Load project context and show which PRD to write next. Use before /prd to get oriented. Triggers on: pre-prd, before prd, prd context, what prd next, start prd."
---

# Pre-PRD Context Loader

Load project context and identify the next PRD to write before starting `/prd`.

---

## The Job

1. Read `tasks/roadmap.md` to find the next PRD
2. Read AGENTS.md files and summarize key patterns
3. Read relevant schemas and summarize existing types
4. Output context summary and next steps

---

## Step 1: Read Roadmap

```bash
cat tasks/roadmap.md
```

Parse the tables to determine:
- Which PRDs are **Completed** with their Category
- Which PRDs are **In Progress** with their Category
- Which PRDs are **Remaining** with their dependencies and Category

### Parse Categories

Tables now have a Category column. Format: `| PRD | Category | Description | (Dependencies)`

Categories:
- `feature`: New functionality (Feature Development swimlane)
- `review`: Code quality fixes from AI review (Quality & Maintenance swimlane)
- `devops`: Tooling and infrastructure (Quality & Maintenance swimlane)

**Backward Compatibility:** If Category column is missing, treat all PRDs as `feature`.

### Group by Category

Group available PRDs by category/swimlane:
1. **Quality & Maintenance**: `review` and `devops` categories
2. **Feature Development**: `feature` category

### Find Next PRD

The next PRD is the first entry in "Remaining" where ALL dependencies are in "Completed".

**Dependency Resolution:**
1. For each Remaining PRD, check its Dependencies column
2. Map dependency names to PRD names:
   - "Data model" → `prd-data-model.md`
   - "Terminology" → `prd-terminology.md`
   - "FHIR import" → `prd-fhir-import.md`
3. Check if all dependencies appear in Completed table
4. First Remaining PRD with all dependencies met = next PRD

If a PRD is In Progress, inform the user they should complete it first.

---

## Step 2: Read AGENTS.md Files

Read these files:
```bash
cat AGENTS.md
cat packages/core/AGENTS.md
cat apps/api/AGENTS.md
```

**Extract and summarize:**
- Project structure overview
- Critical patterns (5-10 bullet points max)
- Common gotchas

Do NOT output the full file contents. Summarize.

---

## Step 3: Read Relevant Schemas

If the next PRD involves data model changes, read:
```bash
ls packages/core/src/schemas/
```

For each schema file, extract:
- Class/type names
- One-line description of purpose

**Example output:**
```
### Existing Schemas

- `instrument.py`: Instrument, Question, Section, AnswerOption, EnableWhen
- `terminology.py`: CodeSystem, Concept, ValueSet, CodeBinding
- `interpretation.py`: InterpretationRule, RuleCondition, RuleAction
```

---

## Step 4: Output Format

```markdown
## Next PRD: `prd-<feature>.md`

**Category:** feature/review/devops
**Description:** <description from roadmap>

**Dependencies:**
- [x] prd-data-model.md (Completed)
- [x] prd-terminology.md (Completed)

---

## Relevant Context

### Project Structure
- `packages/core/` - Pydantic schemas, SQLAlchemy models, repositories
- `apps/api/` - FastAPI backend with Alembic migrations
- `apps/web/` - React frontend (not yet built)

### Existing Schemas
- `instrument.py`: Instrument, Question, Section, AnswerOption, EnableWhen
- `terminology.py`: CodeSystem, Concept, ValueSet, CodeBinding

### Key Patterns
- Pydantic v2 with `model_config = ConfigDict(from_attributes=True)`
- SQLAlchemy 2.0 async with `AsyncSession`
- Repository pattern for data access
- Tests use `pytest-asyncio` with `AsyncMock` for database mocking

### Standards
- FHIR R4, US Core 6.1.0, SDC IG
- LOINC, SNOMED CT, ICD-10-CM

---

## Available PRDs by Category

### Quality & Maintenance
PRDs for code quality, tooling, and infrastructure improvements.

| Command | Category | Description |
|---------|----------|-------------|
| `/prd code-review` | devops | Automated AI code review in /finish-ralph |
| `prd-review-*.md` | review | Auto-generated from code review findings |

### Feature Development
PRDs for new product functionality.

| Command | Category | Description |
|---------|----------|-------------|
| `/prd auth` | feature | Public read-only + basic auth for writes |
| `/prd fhir-export` | feature | Export internal model to FHIR format |
| `/prd interpretation-rules` | feature | Rule storage + spreadsheet import |

---

## Choose Your Focus

**Quality & Maintenance** - Address technical debt and tooling
**Feature Development** - Build new product capabilities

Which category would you like to work on?

## Ready to Write PRD

Run `/prd <feature>` to start writing the PRD. Example:

```
/prd auth
```

The `/prd` skill will read the platform overview, explain what the feature should do, and ask you clarifying questions.
```

---

## Special Cases

### If a PRD is In Progress

```markdown
## Current PRD In Progress: `prd-<feature>.md`

This PRD is already being worked on. Options:

1. **Continue with Ralph**: Run `/ralph-ready` then `./ralph.sh`
2. **Start a different PRD**: Tell me which one

Run `/workflow` to see Ralph status.
```

### If All PRDs are Complete

```markdown
## All Planned PRDs Complete!

The current roadmap is complete. Options:

1. **Add new PRD to roadmap**: Edit `tasks/roadmap.md`
2. **Review completed work**: Check `archive/` for completed runs

What would you like to build next?
```

### If Dependencies Not Met

```markdown
## Next PRD: `prd-<feature>.md`

**Dependencies NOT met:**
- [ ] prd-terminology.md (In Progress - complete this first)

Complete `prd-terminology.md` before starting this PRD.

Run `/workflow` to see current Ralph status.
```

---

## Checklist

Before outputting:

- [ ] Read `tasks/roadmap.md`
- [ ] Parsed Category column (or defaulted to `feature` if missing)
- [ ] Grouped PRDs by swimlane (Quality & Maintenance / Feature Development)
- [ ] Identified next PRD based on dependency order
- [ ] Read and summarized AGENTS.md files
- [ ] Read and summarized relevant schemas
- [ ] Showed dependency status (met/unmet)
- [ ] Prompted user to choose category if both swimlanes have available PRDs
- [ ] Provided clear next step
