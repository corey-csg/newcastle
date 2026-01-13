# PRD: Newcastle Diagram Refinement

## Overview

Iteratively refine two architectural diagrams for Newcastle (Payer Rules Intelligence Platform) until they achieve stakeholder consensus across multiple personas.

## Deliverables

### 1. Executive Markitecture Diagram
- **Source**: `src/newcastle-markitecture-v8.d2`
- **Output**: `output/newcastle-markitecture-final.png` and `.svg`
- **Tool**: D2 (`d2 --theme 0 <source> <output>`)
- **Purpose**: High-level value story for executives, investors, sales decks

### 2. C4 System Context Diagram
- **Source**: `src/newcastle-c4-context-v3.puml`
- **Output**: `output/newcastle-context-final.png` and `.svg`
- **Tool**: PlantUML (`plantuml -tpng <source>` and `plantuml -tsvg <source>`)
- **Purpose**: Technical architecture overview for engineering and technical buyers

## Brand Guidelines

Use Clay Street Group color palette:
- **Primary accent (hero)**: Cyan `#00AEEF`
- **Success/Value**: Green `#6CC24A`
- **Neutral dark**: 100% Black `#000000`, 70% Black `#4D4D4D`
- **Neutral mid**: 40% Black `#999999`, `#808080`
- **Neutral light**: `#f5f5f5`, `#cccccc`
- **Font**: Avenir Next (or system sans-serif fallback)

## Stakeholder Personas

For each iteration, simulate feedback from these four personas:

### 1. C-Suite Executive (CEO/CFO)
**Perspective**: Business value, ROI, market positioning
**Evaluates**:
- Can I understand what this does in 5 seconds?
- Is the value proposition clear?
- Does it look like a company I'd invest in or buy from?
- Would I be comfortable showing this to my board?

### 2. Technology Leadership (CTO/VP Engineering/Senior Architect)
**Perspective**: Technical accuracy, architectural soundness
**Evaluates**:
- Is the system boundary clear?
- Are the integrations and data flows accurate?
- Does this match how we'd actually build it?
- Is the abstraction level appropriate for the audience?

### 3. Potential Customer (RCM Director at a Health System)
**Perspective**: Problem/solution fit, ease of understanding
**Evaluates**:
- Do I recognize my problem in this diagram?
- Can I see how this fits into my workflow?
- Does this look like something that would help my team?
- Is the jargon appropriate (not too technical, not too vague)?

### 4. Sales/BD Leader
**Perspective**: Sellability, differentiation, conversation starter
**Evaluates**:
- Can I use this to open a conversation?
- Does it differentiate us from competitors?
- Is it simple enough to explain in 60 seconds?
- Does it create curiosity and questions?

## Acceptance Criteria

Both diagrams must achieve **consensus approval** from all four personas on these dimensions:

| Criterion | Definition |
|-----------|------------|
| **Intuitive** | The story/flow is immediately obvious without explanation |
| **Sharp** | Visually crisp, no blurry elements, proper resolution |
| **Slide-ready** | Fits well on a 16:9 slide, not too wide or tall, readable at presentation size |
| **Professional** | Looks like it came from a well-funded, serious company |
| **Modern** | Contemporary design aesthetic, not dated or cluttered |
| **Clean** | Appropriate whitespace, no visual noise, balanced composition |

## Iteration Process

### For each iteration:

1. **Render current diagrams** to PNG
2. **Present to each persona** (simulate their perspective)
3. **Collect specific feedback** from each persona:
   - What works?
   - What doesn't work?
   - Specific suggestions for improvement
4. **Synthesize feedback** - identify common themes and conflicts
5. **Make targeted edits** to source files based on feedback
6. **Re-render and repeat** until consensus

### Consensus Definition

- All 4 personas rate the diagram as "Approved" or "Approved with minor notes"
- No persona has a "Blocking" concern
- Any "Minor notes" are acknowledged but don't require another full iteration

### Feedback Format

For each persona, provide structured feedback:

```
## [Persona Name] Feedback - Iteration N

**Verdict**: [Approved | Approved with minor notes | Needs revision | Blocking concern]

**What works**:
- [Specific positive feedback]

**What doesn't work**:
- [Specific issues]

**Suggestions**:
- [Actionable changes]
```

## Current State

### Markitecture (v8)
- Provider Config on left feeding into Newcastle with "personalizes" label
- Data Sources (Payer Rules, Reference Standards, Industry Intel) in container
- Newcastle as cyan hero box in center
- Delivery Channels (Web Portal, Desktop Widget, AI Assistant, Smart Alerts)
- Business Value (Revenue, Fewer Denials, Less Admin Burden) in green

**Known issues to address**:
- Data Sources container is positioned to the right instead of centered above
- Overall layout may be too spread out
- "personalizes" label may be too subtle

### C4 Context (v3)
- Three data source systems on left (Payer Rules, Reference Standards, Industry Intel)
- Newcastle system in center
- Two user personas on right (RCM/Coding Staff, Provider Admin)
- Clean left-to-right flow

**Known issues to address**:
- May need better visual hierarchy
- Relationship labels could be clearer

## Output Requirements

### Final deliverables:
1. `newcastle-markitecture-final.png` - PNG for slides
2. `newcastle-markitecture-final.svg` - SVG for scaling
3. `newcastle-context-final.png` - PNG for slides
4. `newcastle-context-final.svg` - SVG for scaling
5. Updated source files with final versions
6. Summary of iterations and key changes made

### Documentation:
- Record of each iteration's feedback
- Final approval from all personas
- Change log of modifications made

## Success Metric

Both diagrams pass the "5-second test":
> A viewer seeing the diagram for the first time can accurately describe what it shows within 5 seconds of viewing.
