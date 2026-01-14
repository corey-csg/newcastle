# Newcastle Agent Instructions

## Overview
Newcastle is a Payer Rules Intelligence Platform that aggregates and normalizes healthcare payer billing rules, providing AI-powered change detection and impact scoring.

## Project Structure
```
docs/diagrams/
├── src/           # Source files (.d2, .puml)
├── output/        # Rendered diagrams (.png, .svg)
├── prd.json       # Diagram PRD
└── ITERATION-LOG.md
```

## Commands
- D2 diagrams: `d2 --theme 0 <source.d2> <output.png>`
- PlantUML PNG: `plantuml -tpng <source.puml>`
- PlantUML SVG: `plantuml -tsvg <source.puml>`

## Critical Patterns

### Diagram Tools
- D2: Use `--theme 0` for clean white background
- PlantUML: Output filename comes from `@startuml <name>`, not input filename
- PlantUML: Use `-o <dir>` to specify output directory

### D2 Specifics
- Use `direction: right` inside containers for horizontal child layout
- Use `style.stroke: transparent` for invisible positioning connections
- Container nesting affects auto-layout significantly

### Brand Guidelines
- Cyan `#00AEEF` - Hero/primary accent
- Green `#6CC24A` - Value/success
- Dark gray `#4D4D4D` - External data sources
- Mid gray `#999999` - Customer configuration

### Diagram Iteration Process
1. Render diagram to PNG
2. View with Read tool
3. Simulate 4 personas (C-Suite, Tech Leadership, Customer, Sales/BD)
4. Edit source based on feedback
5. Repeat until consensus

## Roadmap
- Phase 1: Architectural diagrams (complete)
- Phase 2: TBD

## Learning Protocol
When you discover reusable patterns:
1. Add to appropriate AGENTS.md section
2. Keep it actionable and specific
3. Include context on when to apply
