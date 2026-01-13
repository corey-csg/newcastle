---
name: local-dev
description: "Manage local development stack (database, API, frontend). Start, stop, and check status of services. Triggers on: local-dev, dev stack, start services, stop services, dev status."
---

# Local Development Stack Manager

Manage the Tauri Fusion local development stack from within Claude Code.

---

## Commands

The skill accepts a command argument to determine what action to take.

### Parse the Command

Extract the command from the skill invocation. If no command is provided, default to "status".

Valid commands:
- `start` - Start full stack
- `stop` - Stop all services
- `status` - Check health of all services
- `db` - Start just database
- `api` - Start just API
- `web` - Start just frontend
- `migrate` - Run database migrations

---

## Command: start

Start the full development stack (database, migrations, API, web).

**Actions:**
1. Run `./scripts/banner.sh` to display the FUSION REACTOR banner
2. Run `./scripts/db-start.sh` to start PostgreSQL
3. Run `./scripts/migrate.sh` to apply migrations
4. Run `./scripts/api-start.sh &` in background to start API
5. Wait 2 seconds for API to initialize
6. Run `./scripts/web-start.sh &` in background to start frontend

**Note:** The API and web servers run in the foreground. When starting them in Claude Code, they will run as background processes. Use `status` to check if they're running.

**Output:**
```
Starting development stack...

[FUSION REACTOR Banner]

Database: Starting... OK
Migrations: Applying... OK
API: Starting on http://localhost:8000
Web: Starting on http://localhost:5173

Development stack is running!
Use /local-dev status to check service health.
Use /local-dev stop to shut down.
```

---

## Command: stop

Stop all development services.

**Actions:**
1. Run `./scripts/dev-stop.sh`

**Output:**
```
Stopping development services...

Web: Stopped
API: Stopped
Database: Stopped

All services stopped.
```

---

## Command: status

Check the health of all services and display their status.

**Actions:**
1. Run `./scripts/db-health.sh` and capture exit code
2. Run `./scripts/api-health.sh` and capture exit code
3. Check if port 5173 is in use for web

**Status Detection:**
```bash
# Database health
./scripts/db-health.sh
DB_STATUS=$?

# API health
./scripts/api-health.sh
API_STATUS=$?

# Web health (check if port is in use)
lsof -ti:5173 > /dev/null 2>&1
WEB_STATUS=$?
```

**Output Format:**
```
## Development Stack Status

| Service  | Port | Status   |
|----------|------|----------|
| Database | 5432 | Running  |
| API      | 8000 | Running  |
| Web      | 5173 | Stopped  |

Services:
  API: http://localhost:8000
  Web: http://localhost:5173
```

Use icons:
- Running (exit code 0)
- Stopped (exit code non-zero)

---

## Command: db

Start just the database.

**Actions:**
1. Run `./scripts/db-start.sh`

**Output:**
```
Starting PostgreSQL database...

Database is ready on localhost:5432
```

---

## Command: api

Start just the API server.

**Actions:**
1. Check if database is running first (warn if not)
2. Run `./scripts/api-start.sh`

**Output:**
```
Starting FastAPI server...

API is running on http://localhost:8000

Note: API requires database to be running.
Use /local-dev db to start the database if needed.
```

---

## Command: web

Start just the frontend dev server.

**Actions:**
1. Run `./scripts/web-start.sh`

**Output:**
```
Starting Vite dev server...

Frontend is running on http://localhost:5173
```

---

## Command: migrate

Run database migrations.

**Actions:**
1. Check if database is running first (error if not)
2. Run `./scripts/migrate.sh`

**Output:**
```
Running database migrations...

[Migration output]

Migrations complete.
```

---

## Error Handling

For all commands, handle errors gracefully:

### Database Not Running (for api, migrate)

If database is required but not running:
```
Error: Database is not running.

Start the database first:
  /local-dev db

Then try again.
```

### Port Already in Use (for api, web)

If the target port is already in use:
```
Warning: Port XXXX is already in use.

The service may already be running.
Check status: /local-dev status
Or stop first: /local-dev stop
```

### Script Not Found

If a script doesn't exist:
```
Error: Required script not found: scripts/xxx.sh

Make sure you're in the Tauri Fusion project root.
```

---

## Checklist

Before executing any command:

- [ ] Identified the command (start, stop, status, db, api, web, migrate)
- [ ] Verified we're in the project root (scripts/ directory exists)
- [ ] For commands requiring database: checked database health first
- [ ] Executed the appropriate script(s)
- [ ] Displayed clear status output with service URLs
- [ ] Handled any errors with helpful messages
