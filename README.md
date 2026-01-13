# Newcastle

A full-stack web application built with Ralph workflow for AI-assisted iterative development.

## Getting Started

[To be defined via PRD]

## Development

This project uses the Ralph workflow for iterative development:

```
/pre-prd → /prd → /prd-json → /ralph-ready → ./ralph.sh → /post-ralph → /finish-ralph
```

## Project Structure

```
newcastle/
├── .claude/
│   ├── settings.json      # Permission allowlist
│   └── skills/            # Workflow skills
├── tasks/                 # PRD storage
├── archive/               # Completed features
├── ralph.sh               # Agent loop
├── prompt.md              # Iteration instructions
├── progress.txt           # Iteration log
├── AGENTS.md              # Project patterns
├── CLAUDE.md              # Tech stack & conventions
└── README.md
```
