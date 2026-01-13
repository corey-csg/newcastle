---
name: prd-json
description: "Convert a finished PRD to prd.json format for Ralph execution. Use after writing a PRD with /prd. Use --fix to repair existing prd.json. Triggers on: prd-json, convert prd, prd to json, make prd.json, create prd.json, prd-json --fix, fix prd.json."
---

# PRD to JSON Converter

Converts a finished PRD (markdown) to `prd.json` format for Ralph autonomous execution.

---

## The Job

Take a finished PRD from `tasks/prd-<feature>.md` and convert it to `prd.json` in the project root.

**Workflow context:** This skill runs after `/prd <feature>` completes. The PRD should already be written and reviewed.

---

## Fix Mode: `/prd-json --fix`

Use `--fix` to repair an existing `prd.json` that is missing type tags or required acceptance criteria.

### When to Use

- Called by `/ralph-ready` when validation fails
- Manual use when you've edited prd.json and want to ensure it's complete
- After importing stories from another source

### What --fix Does

1. **Reads existing prd.json** - Loads the current file from project root
2. **Adds missing type tags** - Auto-detects type for stories without a `type` field (see Auto-Detection Rules)
3. **Adds missing acceptance criteria** - Adds type-appropriate criteria that are missing (see Auto-Added Acceptance Criteria)
4. **Preserves existing content** - Never modifies: `id`, `title`, `description`, `notes`, `passes`, `priority`, or existing criteria
5. **Reports changes** - Shows what was added/changed for transparency

### Fix Mode Output

When running `/prd-json --fix`, show a summary of changes:

```
Fixing prd.json...

US-001 "Add status field to tasks table"
  Type: data (auto-detected)
  + Added: Migration runs successfully
  + Added: Schema validation passes
  + Added: Typecheck passes

US-002 "Create dashboard page"
  Type: ui (already set)
  ✓ All required criteria present

US-003 "Wire up API to form"
  Type: integration (auto-detected from "API" + "form")
  + Added: E2E test passes: api-form.spec.ts
  + Added: API response validated
  (already had: Typecheck passes)

Fixed 2 of 3 stories. prd.json updated.
```

### Preservation Rules

The `--fix` flag ONLY adds missing data. It never:
- Changes existing type tags (explicit type always wins)
- Removes or modifies existing acceptance criteria
- Changes story order, priority, or any other fields
- Marks `passes: true` for incomplete stories

### Example: Before and After Fix

**Before (missing type and criteria):**
```json
{
  "id": "US-002",
  "title": "Display status badge on task cards",
  "description": "As a user, I want to see task status at a glance.",
  "acceptanceCriteria": [
    "Each task card shows colored status badge"
  ],
  "priority": 2,
  "passes": false,
  "notes": ""
}
```

**After `/prd-json --fix`:**
```json
{
  "id": "US-002",
  "title": "Display status badge on task cards",
  "description": "As a user, I want to see task status at a glance.",
  "type": "ui",
  "acceptanceCriteria": [
    "Each task card shows colored status badge",
    "Playwright E2E test passes: status-badge.spec.ts",
    "Visual verification passes",
    "Typecheck passes"
  ],
  "priority": 2,
  "passes": false,
  "notes": ""
}
```

### When Fix Can't Help

Some issues require manual intervention:
- Stories too large (need manual splitting)
- Circular dependencies (need reordering)
- Vague criteria (need human clarification)

---

## Output Format

```json
{
  "project": "[Project Name]",
  "branchName": "ralph/[feature-name-kebab-case]",
  "description": "[Feature description from PRD title/intro]",
  "userStories": [
    {
      "id": "US-001",
      "title": "[Story title]",
      "description": "As a [user], I want [feature] so that [benefit]",
      "type": "ui",
      "acceptanceCriteria": [
        "Criterion 1",
        "Criterion 2",
        "Typecheck passes"
      ],
      "priority": 1,
      "passes": false,
      "notes": ""
    }
  ]
}
```

### Story Type Field

Each story can have an optional `type` field to indicate what kind of story it is:

| Type | Description | Example Stories |
|------|-------------|-----------------|
| `ui` | Frontend components, pages, forms, visual elements | "Add login form", "Create dashboard page" |
| `backend` | API endpoints, services, business logic | "Add user endpoint", "Create notification service" |
| `integration` | Spans both frontend and backend | "Wire up login form to API", "Add real-time updates" |
| `data` | Database schemas, migrations, data models | "Add users table", "Create status column" |
| `devops` | Scripts, CI/CD, infrastructure, tooling | "Add deploy script", "Update CI pipeline" |

The type field is **optional**. If omitted, it will be auto-detected from story content (see Auto-Detection section).

---

## Auto-Detection Rules

When a story does not have an explicit `type` field, detect it from the story title and description using these keyword patterns:

### Detection Priority Order

Detection is evaluated in this order. First match wins:

1. **`integration`** - Story spans both frontend AND backend
   - Contains BOTH frontend keywords (page, view, form, component, button, modal, display, render, UI)
   - AND backend keywords (API, endpoint, service, database, query, server)
   - Example: "Wire up login form to API", "Add real-time updates to dashboard"

2. **`ui`** - Frontend-only changes
   - Keywords: `page`, `view`, `form`, `component`, `button`, `modal`, `display`, `render`, `UI`, `frontend`, `layout`, `style`, `CSS`, `icon`, `header`, `footer`, `navigation`, `menu`
   - Example: "Add login form", "Create dashboard page", "Display status badge"

3. **`backend`** - Backend-only changes
   - Keywords: `API`, `endpoint`, `service`, `database query`, `server action`, `route`, `handler`, `middleware`, `auth`, `session`
   - Example: "Add user endpoint", "Create notification service"

4. **`data`** - Database schema and data model changes
   - Keywords: `migration`, `schema`, `table`, `column`, `index`, `constraint`, `model`, `database`, `seed`, `field`
   - Example: "Add users table", "Create status column", "Add index on email"

5. **`devops`** - Infrastructure, tooling, scripts
   - Keywords: `script`, `CI`, `CD`, `deploy`, `infrastructure`, `skill`, `config`, `pipeline`, `docker`, `build`, `lint`, `test runner`
   - Example: "Add deploy script", "Update CI pipeline", "Create skill file"

### Override Behavior

- **Explicit type always wins**: If the PRD or story explicitly specifies a `type`, use that value
- **Case-insensitive matching**: Keywords are matched case-insensitively
- **Default fallback**: If no keywords match, default to `backend`

### Examples

| Story Title | Detected Type | Why |
|-------------|---------------|-----|
| "Add login form" | `ui` | Contains "form" keyword |
| "Create user API endpoint" | `backend` | Contains "API" and "endpoint" |
| "Wire up form to backend" | `integration` | Contains both "form" (ui) and "backend" |
| "Add users migration" | `data` | Contains "migration" keyword |
| "Update CI pipeline" | `devops` | Contains "CI" and "pipeline" |
| "Add status column to database" | `data` | Contains "column" and "database" |
| "Display user profile from API" | `integration` | Contains "display" (ui) and "API" (backend) |

---

## Auto-Added Acceptance Criteria

Based on story type, automatically add type-appropriate acceptance criteria. These criteria ensure Ralph verifies stories using the right testing approach.

### Criteria by Story Type

| Type | Auto-Added Criteria |
|------|---------------------|
| `ui` | `Playwright E2E test passes: {feature}.spec.ts`<br>`Visual verification passes`<br>`Typecheck passes` |
| `backend` | `pytest passes`<br>`pyright passes`<br>`Typecheck passes` |
| `integration` | `E2E test passes: {feature}.spec.ts`<br>`API response validated`<br>`Typecheck passes` |
| `data` | `Migration runs successfully`<br>`Schema validation passes`<br>`Typecheck passes` |
| `devops` | `Script executes successfully` |

**Note:** All story types get "Typecheck passes" **except** `devops` stories (which typically modify non-code files like skills, config, or documentation).

### {feature} Placeholder

For `ui` and `integration` stories, replace `{feature}` with a kebab-case version of the story title:
- "Add login form" → `login-form.spec.ts`
- "Create dashboard page" → `dashboard-page.spec.ts`
- "Wire up user settings" → `user-settings.spec.ts`

### Transparency in Output

When generating prd.json, show which criteria were auto-added for transparency:

```
Story: US-002 "Display status badge"
Type: ui (auto-detected)
Auto-added criteria:
  + Playwright E2E test passes: status-badge.spec.ts
  + Visual verification passes
  + Typecheck passes
```

### Duplicate Prevention

Before adding auto-criteria, check if the PRD already includes equivalent criteria:
- Skip "Typecheck passes" if already present
- Skip "pytest passes" if already present
- Skip Playwright criteria if story already has test file reference

### Example: Complete Story with Auto-Added Criteria

**Input story from PRD:**
```
## US-002: Display status badge on task cards
As a user, I want to see task status at a glance.
- Each task card shows colored status badge
- Badge colors: gray=pending, blue=in_progress, green=done
```

**Output in prd.json (after auto-detection and auto-add):**
```json
{
  "id": "US-002",
  "title": "Display status badge on task cards",
  "description": "As a user, I want to see task status at a glance.",
  "type": "ui",
  "acceptanceCriteria": [
    "Each task card shows colored status badge",
    "Badge colors: gray=pending, blue=in_progress, green=done",
    "Playwright E2E test passes: status-badge.spec.ts",
    "Visual verification passes",
    "Typecheck passes"
  ],
  "priority": 2,
  "passes": false,
  "notes": ""
}
```

The auto-added criteria appear after the original criteria from the PRD.

---

## Story Size: The Number One Rule

**Each story must be completable in ONE Ralph iteration (one context window).**

Ralph spawns a fresh Amp instance per iteration with no memory of previous work. If a story is too big, the LLM runs out of context before finishing and produces broken code.

### Right-sized stories:
- Add a database column and migration
- Add a UI component to an existing page
- Update a server action with new logic
- Add a filter dropdown to a list

### Too big (split these):
- "Build the entire dashboard" - Split into: schema, queries, UI components, filters
- "Add authentication" - Split into: schema, middleware, login UI, session handling
- "Refactor the API" - Split into one story per endpoint or pattern

**Rule of thumb:** If you cannot describe the change in 2-3 sentences, it is too big.

---

## Story Ordering: Dependencies First

Stories execute in priority order. Earlier stories must not depend on later ones.

**Correct order:**
1. Schema/database changes (migrations)
2. Server actions / backend logic
3. UI components that use the backend
4. Dashboard/summary views that aggregate data

**Wrong order:**
1. UI component (depends on schema that does not exist yet)
2. Schema change

---

## Acceptance Criteria: Must Be Verifiable

Each criterion must be something Ralph can CHECK, not something vague.

### Good criteria (verifiable):
- "Add `status` column to tasks table with default 'pending'"
- "Filter dropdown has options: All, Active, Completed"
- "Clicking delete shows confirmation dialog"
- "Typecheck passes"
- "Tests pass"

### Bad criteria (vague):
- "Works correctly"
- "User can do X easily"
- "Good UX"
- "Handles edge cases"

### Always include as final criterion:
```
"Typecheck passes"
```

For stories with testable logic, also include:
```
"Tests pass"
```

### For stories that change UI, also include:
```
"Verify in browser using dev-browser skill"
```

Frontend stories are NOT complete until visually verified. Ralph will use the dev-browser skill to navigate to the page, interact with the UI, and confirm changes work.

---

## Conversion Rules

1. **Each user story becomes one JSON entry**
2. **IDs**: Sequential (US-001, US-002, etc.)
3. **Priority**: Based on dependency order, then document order
4. **All stories**: `passes: false` and empty `notes`
5. **branchName**: Derive from feature name, kebab-case, prefixed with `ralph/`
6. **Always add**: "Typecheck passes" to every story's acceptance criteria
7. **Type field**: Optional - include if story type is clear, otherwise auto-detect later

---

## Splitting Large PRDs

If a PRD has big features, split them:

**Original:**
> "Add user notification system"

**Split into:**
1. US-001: Add notifications table to database
2. US-002: Create notification service for sending notifications
3. US-003: Add notification bell icon to header
4. US-004: Create notification dropdown panel
5. US-005: Add mark-as-read functionality
6. US-006: Add notification preferences page

Each is one focused change that can be completed and verified independently.

---

## Example

**Input PRD:**
```markdown
# Task Status Feature

Add ability to mark tasks with different statuses.

## Requirements
- Toggle between pending/in-progress/done on task list
- Filter list by status
- Show status badge on each task
- Persist status in database
```

**Output prd.json (with auto-added criteria shown):**

```
Converting PRD to prd.json...

Story: US-001 "Add status field to tasks table"
Type: data (auto-detected from "table")
Auto-added criteria:
  + Migration runs successfully
  + Schema validation passes
  + Typecheck passes

Story: US-002 "Display status badge on task cards"
Type: ui (auto-detected from "display")
Auto-added criteria:
  + Playwright E2E test passes: status-badge.spec.ts
  + Visual verification passes
  + Typecheck passes

Story: US-003 "Add status toggle to task list rows"
Type: integration (auto-detected from "toggle" + "list")
Auto-added criteria:
  + E2E test passes: status-toggle.spec.ts
  + API response validated
  + Typecheck passes

Story: US-004 "Filter tasks by status"
Type: ui (auto-detected from "filter")
Auto-added criteria:
  + Playwright E2E test passes: filter-tasks.spec.ts
  + Visual verification passes
  + Typecheck passes

Generated prd.json with 4 stories.
```

```json
{
  "project": "TaskApp",
  "branchName": "ralph/task-status",
  "description": "Task Status Feature - Track task progress with status indicators",
  "userStories": [
    {
      "id": "US-001",
      "title": "Add status field to tasks table",
      "description": "As a developer, I need to store task status in the database.",
      "type": "data",
      "acceptanceCriteria": [
        "Add status column: 'pending' | 'in_progress' | 'done' (default 'pending')",
        "Generate and run migration successfully",
        "Migration runs successfully",
        "Schema validation passes",
        "Typecheck passes"
      ],
      "priority": 1,
      "passes": false,
      "notes": ""
    },
    {
      "id": "US-002",
      "title": "Display status badge on task cards",
      "description": "As a user, I want to see task status at a glance.",
      "type": "ui",
      "acceptanceCriteria": [
        "Each task card shows colored status badge",
        "Badge colors: gray=pending, blue=in_progress, green=done",
        "Playwright E2E test passes: status-badge.spec.ts",
        "Visual verification passes",
        "Typecheck passes"
      ],
      "priority": 2,
      "passes": false,
      "notes": ""
    },
    {
      "id": "US-003",
      "title": "Add status toggle to task list rows",
      "description": "As a user, I want to change task status directly from the list.",
      "type": "integration",
      "acceptanceCriteria": [
        "Each row has status dropdown or toggle",
        "Changing status saves immediately",
        "UI updates without page refresh",
        "E2E test passes: status-toggle.spec.ts",
        "API response validated",
        "Typecheck passes"
      ],
      "priority": 3,
      "passes": false,
      "notes": ""
    },
    {
      "id": "US-004",
      "title": "Filter tasks by status",
      "description": "As a user, I want to filter the list to see only certain statuses.",
      "type": "ui",
      "acceptanceCriteria": [
        "Filter dropdown: All | Pending | In Progress | Done",
        "Filter persists in URL params",
        "Playwright E2E test passes: filter-tasks.spec.ts",
        "Visual verification passes",
        "Typecheck passes"
      ],
      "priority": 4,
      "passes": false,
      "notes": ""
    }
  ]
}
```

---

## Checklist Before Saving

Before writing prd.json, verify:

- [ ] Each story is completable in one iteration (small enough)
- [ ] Stories are ordered by dependency (schema to backend to UI)
- [ ] Story types are set (explicitly or auto-detected): `ui`, `backend`, `integration`, `data`, `devops`
- [ ] Auto-added criteria applied based on story type (see "Auto-Added Acceptance Criteria" section)
- [ ] Acceptance criteria are verifiable (not vague)
- [ ] No story depends on a later story
- [ ] Output shows which criteria were auto-added for transparency
