# Knowledge Studio: Architecture Briefing Document

**Status:** Early Exploration - No Decisions Finalized  
**Date:** January 14, 2026  
**Purpose:** Capture context for future development sessions

---

## ⚠️ IMPORTANT CONTEXT FOR FUTURE SESSIONS

This document captures an exploratory conversation about building a healthcare knowledge management platform. **No technology decisions have been made.** The user is evaluating options and gathering information. Do not assume any stack or architecture is chosen—treat all options as still on the table.

---

## 1. Project Overview

### What We're Building

A **clinical knowledge authoring and distribution platform** for revenue cycle management (RCM) staff at healthcare provider organizations.

### Core Function

Curate, develop, version, and distribute executable knowledge assets that encode:
- Terminology mappings (screening responses → billable codes)
- Payer coverage rules (what's billable under what conditions)
- Clinical guideline logic (medical necessity documentation requirements)

### What This Is NOT

- Not a clinical decision support system for clinicians
- Not an EHR integration or SMART on FHIR app
- Not a claims adjudication engine (though it informs one)
- Not executing rules at runtime against patient data

### Target Users

Revenue cycle management staff at provider organizations:
- Medical coders
- Billers
- Revenue cycle analysts
- Clinical documentation specialists

These users need to operationalize clinical and administrative knowledge—not make clinical decisions.

---

## 2. User's Background & Expertise

Understanding the user's background is critical for recommendations:

| Area | Expertise Level |
|------|-----------------|
| Java | **Expert** - Primary development background |
| Database/SQL | **Expert** - Self-described as "mostly a backend database developer" |
| HL7 FHIR | **Expert** - Explicitly stated expertise |
| Terminology Management | **Expert** - Explicitly stated expertise |
| Python | **Learning** - Used for AI experimentation |
| Go | **None** - Never tried, expressed interest in learning more |
| Frontend Development | **Unknown** - Not discussed |

### Implication

The user has deep healthcare domain knowledge that most developers lack. The challenge is not understanding FHIR or terminology—it's selecting the right application architecture and AI integration patterns.

---

## 3. The Three Knowledge Asset Types

### Type 1: Terminology Mappings

**Example:** SDOH screening responses → SNOMED CT + ICD-10 codes

```
IF LOINC:96777-8 (Food insecurity risk) = "At risk"
THEN 
  SNOMED: 733423003 (Food insecurity)
  ICD-10: Z59.41 (Food insecurity)
```

**Characteristics:**
- Deterministic (same input → same output)
- May involve multiple questions + answers → single output
- Changes with terminology releases (quarterly-ish)
- High volume at runtime
- Auditability required

**Representation options discussed:**
- FHIR ConceptMap (standard, well-understood)
- Custom DSL for multi-input conditional mappings

### Type 2: Payer Coverage Rules

**Example:** What's required to bill a specific service

```
Payer: Aetna Commercial
Service: 99453 (Remote monitoring setup)
Effective: 2024-01-01 to 2024-12-31
Requires:
  - Diagnosis in (E11.*, I10, J44.*)
  - Order from qualified provider
  - Device meets FDA criteria
Modifier: 95 if synchronous telehealth
Frequency: 1 per patient per device per 365 days
```

**Characteristics:**
- Complex conditional logic
- Varies by payer, plan, state, date of service
- Changes frequently (policy updates)
- No universal standard exists
- High stakes (incorrect = denials or fraud risk)

**Representation options discussed:**
- Custom DSL (most likely path)
- CQL (possible but not designed for this)
- Wrapped in FHIR Library resources for distribution

### Type 3: Clinical Pathway Logic

**Example:** Guideline-derived medical necessity criteria

```
Guideline: ADA Standards of Care 2024
For: CGM coverage justification
Logic:
  IF diabetes_type IN (Type 1, Type 2 on intensive insulin)
  AND documented_hypoglycemia OR A1C_not_at_goal
  AND failed_or_unable SMBG 4x daily
  THEN CGM medically necessary
  DOCUMENT: Prior A1C values, hypoglycemic episodes, SMBG attempts
```

**Characteristics:**
- Derived from clinical practice guidelines
- Encodes documentation requirements, not treatment decisions
- Used to justify medical necessity for billing
- Must be traceable to source guideline

**Representation options discussed:**
- CQL (Clinical Quality Language) - HL7 standard for this exact purpose
- Custom DSL
- Hybrid approach

---

## 4. Tech Stack Options (UNDECIDED)

### Backend Language Options

| Option | Pros | Cons | Fit for User |
|--------|------|------|--------------|
| **Kotlin/JVM** | User's expertise, HAPI FHIR is Java, mature healthcare ecosystem, excellent database tooling | Less "modern" perception, more verbose than Go | ✅ Strong fit |
| **Go** | Simple, fast compilation, excellent concurrency, great for AI agent orchestration, single binary deployment | No FHIR ecosystem, user would need to learn, less mature database tooling | ⚠️ Learning curve, ecosystem gap |
| **Python** | Best AI/ML library support, user has some experience | Worst performance, async is awkward, not ideal for production systems | ❌ Weak fit for production |
| **TypeScript/Node** | Code sharing with frontend, large ecosystem | Single-threaded limitations, not user's strength | ⚠️ Possible but not natural |

### Frontend Options

| Option | Notes |
|--------|-------|
| **React + TypeScript** | Standard choice, large ecosystem, good AI tooling support |
| **Vue/Svelte** | Lighter weight alternatives, less discussed |

### Desktop/Mobile

| Platform | Option | Notes |
|----------|--------|-------|
| Desktop | Tauri | Rust core + web frontend, lightweight |
| Desktop | Electron | Heavier but more mature |
| Mobile | React Native | If needed—may not be for RCM users |
| Mobile | Skip for v1 | RCM staff typically work on desktops |

### Database

**PostgreSQL** is the assumed choice given:
- User's database expertise
- pgvector for potential embedding storage
- Works well with FHIR data models
- Mature, reliable, well-understood

### Rule Representation Options (CRITICAL UNDECIDED AREA)

| Approach | Description | Trade-offs |
|----------|-------------|------------|
| **FHIR-Native** | ConceptMap, Library, CQL for everything | Max interoperability, but FHIR isn't designed for payer rules |
| **Custom DSL** | Define own rule format | Maximum expressiveness, but consumers need your docs |
| **Hybrid** | FHIR for terminology, custom for coverage rules | Probably the right answer but needs design work |
| **WebAssembly** | Compile rules to WASM for portable execution | Discussed but less relevant since we're not executing at runtime |

---

## 5. Multi-Agent Architecture (EXPLORATORY)

### The Vision

AI-assisted curation with specialized agents coordinated by management agents. Human-in-the-loop for review and approval.

### Proposed Agent Roles

```
┌─────────────────────────────────────────────────────────────────┐
│                    Orchestration Layer                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Curation Coordinator Agent                  │    │
│  │  - Receives source document                              │    │
│  │  - Determines which specialists to invoke                │    │
│  │  - Manages workflow state                                │    │
│  │  - Synthesizes outputs into draft rule                   │    │
│  │  - Routes to QA                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                   │
│          ┌───────────────────┼───────────────────┐              │
│          ▼                   ▼                   ▼              │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐        │
│  │   Document   │   │   Domain     │   │   Quality    │        │
│  │   Analysis   │   │   Experts    │   │   Assurance  │        │
│  │   Agent      │   │   Pool       │   │   Agent      │        │
│  └──────────────┘   └──────────────┘   └──────────────┘        │
│                              │                                   │
│          ┌──────────┬────────┼────────┬──────────┐              │
│          ▼          ▼        ▼        ▼          ▼              │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │ SNOMED  │ │ ICD-10  │ │ CPT/    │ │ Payer   │ │ Clinical│   │
│  │ Expert  │ │ Expert  │ │ HCPCS   │ │ Policy  │ │ Guideline│  │
│  │         │ │         │ │ Expert  │ │ Expert  │ │ Expert  │   │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Agent Responsibilities (Draft)

| Agent | Input | Output |
|-------|-------|--------|
| **Document Analysis** | Raw PDF/document | Document type, structure, codes mentioned, sections identified |
| **SNOMED Expert** | Clinical concept description | Recommended SNOMED codes with justification, alternatives considered |
| **ICD-10 Expert** | Clinical scenario or SNOMED concept | ICD-10 codes with sequencing advice, documentation requirements |
| **CPT/HCPCS Expert** | Service description | Procedure codes, modifiers, bundling considerations |
| **Payer Policy Expert** | Policy document section | Structured coverage rule with conditions, exclusions, requirements |
| **Clinical Guideline Expert** | Guideline excerpt | Medical necessity criteria, required documentation |
| **QA Agent** | Draft rule from coordinator | Errors, warnings, suggestions, approval/rejection |

### Orchestration Approaches (UNDECIDED)

| Approach | Description | Pros | Cons |
|----------|-------------|------|------|
| **Simple Function Orchestration** | Coordinator is code that calls agents in sequence | Predictable, debuggable, full control | Rigid, doesn't adapt to novel situations |
| **Agent-as-Tool Pattern** | Coordinator is an LLM with agents as tools | Flexible, can reason about which experts to use | Less predictable, higher token costs, harder to debug |
| **State Machine** | Workflow as explicit states with agent transitions | Clear audit trail, can persist/resume, testable | More boilerplate, state explosion risk |
| **Framework-Based** | LangGraph, CrewAI, etc. | Don't reinvent orchestration | Framework lock-in, mostly Python-first |

### Recommendation (Tentative)

Start with **Simple Function Orchestration** for predictability, evolve toward **State Machine** as complexity grows. Build lightweight orchestration in chosen backend language rather than adopting a Python framework.

---

## 6. Proposed System Architecture (DRAFT)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Knowledge Studio                              │
│                  (Core Application)                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │   Source     │  │   Author     │  │   Publish    │           │
│  │   Import     │  │   & Edit     │  │   & Export   │           │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤           │
│  │ PDF/doc      │  │ Visual rule  │  │ Version      │           │
│  │ parsing      │  │ editor       │  │ control      │           │
│  │ AI extract   │  │ Validation   │  │ FHIR export  │           │
│  │ Terminology  │  │ Testing      │  │ Custom fmt   │           │
│  │ lookup       │  │ Annotation   │  │ API access   │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Knowledge Repository                          │
├─────────────────────────────────────────────────────────────────┤
│  - FHIR resources (ConceptMap, Library, ValueSet, etc.)         │
│  - Custom rule representations                                   │
│  - Version history and audit trail                               │
│  - Terminology bindings                                          │
│                                                                  │
│  Storage: PostgreSQL + FHIR-aware schema                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Distribution Layer                            │
├─────────────────────────────────────────────────────────────────┤
│  - FHIR API for standard resources                               │
│  - Custom API for rule packages                                  │
│  - Bulk export for offline use                                   │
│  - Webhook notifications for updates                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. AI-Powered Curation Workflow (DRAFT)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Curation Workflow                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────┐                                                │
│   │ User uploads│                                                │
│   │ source doc  │                                                │
│   └──────┬──────┘                                                │
│          │                                                       │
│          ▼                                                       │
│   ┌─────────────────────────────────────────────┐               │
│   │         AI Agent Orchestration              │               │
│   │  ┌─────────┐ ┌─────────┐ ┌─────────┐       │               │
│   │  │ Analyze │→│ Expert  │→│Synthesize│       │               │
│   │  │   Doc   │ │ Consult │ │  Draft   │       │               │
│   │  └─────────┘ └─────────┘ └─────────┘       │               │
│   └─────────────────┬───────────────────────────┘               │
│                     │                                            │
│                     ▼                                            │
│   ┌─────────────────────────────────────────────┐               │
│   │              QA Agent Review                │               │
│   │  - Logical consistency                      │               │
│   │  - Terminology validation                   │               │
│   │  - Completeness check                       │               │
│   └─────────────────┬───────────────────────────┘               │
│                     │                                            │
│                     ▼                                            │
│   ┌─────────────────────────────────────────────┐               │
│   │           Human Review Queue                │               │
│   │  - Draft rule displayed                     │               │
│   │  - Source document linked                   │               │
│   │  - AI confidence + flags shown              │               │
│   └─────────────────┬───────────────────────────┘               │
│                     │                                            │
│          ┌─────────┴─────────┐                                  │
│          ▼                   ▼                                  │
│   ┌─────────────┐     ┌─────────────┐                           │
│   │  Approved   │     │  Rejected/  │                           │
│   │             │     │   Edited    │                           │
│   └──────┬──────┘     └──────┬──────┘                           │
│          │                   │                                   │
│          ▼                   │                                   │
│   ┌─────────────┐            │                                   │
│   │  Versioned  │            │                                   │
│   │  Published  │◄───────────┘                                   │
│   └─────────────┘   (after re-review)                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. Key Open Questions

### Architecture Questions

1. **Rule representation format?**
   - Pure FHIR (ConceptMap, CQL) vs custom DSL vs hybrid?
   - How to represent multi-input conditional terminology mappings?
   - How to represent payer coverage rules (no standard exists)?

2. **Backend language final choice?**
   - Kotlin (leverage expertise, FHIR ecosystem) vs Go (learn something new, simpler concurrency)?
   - If Go, how to handle lack of FHIR libraries?

3. **Agent orchestration pattern?**
   - Simple function calls vs state machine vs framework?
   - How deep should agent recursion go?
   - How to handle agent disagreement?

4. **Distribution format?**
   - FHIR Library wrapping custom content?
   - Pure custom API?
   - Both?

### Product Questions

1. **What does the authoring UX look like?**
   - Visual rule builder vs code/DSL editor vs hybrid?
   - How technical are the users?

2. **How do downstream systems consume rules?**
   - API calls at claim time?
   - Bulk download and local execution?
   - Both?

3. **What's the MVP scope?**
   - Start with one rule type (terminology mappings)?
   - All three from the start?

### Technical Questions

1. **Terminology service: build vs buy?**
   - Run own HAPI FHIR terminology server?
   - Use external (VSAC, NLM)?
   - Hybrid with caching?

2. **How to handle payer policy documents?**
   - PDFs with complex formatting
   - Frequent changes
   - No standard structure

3. **Version control model?**
   - Git-based for rules?
   - Database with audit log?
   - Both?

---

## 9. Relevant Standards & Technologies

### Healthcare Standards

| Standard | Relevance |
|----------|-----------|
| **HL7 FHIR R4** | Core data model for clinical content, terminology, and distribution |
| **CQL (Clinical Quality Language)** | HL7 standard for expressing clinical logic—potential rule format |
| **SNOMED CT** | Clinical terminology for diagnoses, procedures, findings |
| **ICD-10-CM/PCS** | Diagnosis and procedure codes for billing |
| **CPT/HCPCS** | Procedure codes |
| **LOINC** | Lab and clinical observation codes (including SDOH screenings) |
| **VSAC (Value Set Authority Center)** | NLM service for value sets |
| **ConceptMap** | FHIR resource for terminology mappings |

### Potential Libraries/Frameworks

| Library | Language | Purpose |
|---------|----------|---------|
| **HAPI FHIR** | Java/Kotlin | FHIR parsing, validation, server |
| **cql-engine** | Java | CQL execution (if using CQL) |
| **cql-execution** | JavaScript | CQL execution in Node/browser |
| **LangChain4j** | Java | AI agent orchestration (if using framework) |
| **Spring Boot** | Kotlin/Java | Web framework |
| **Ktor** | Kotlin | Lighter web framework alternative |

---

## 10. Development Approach Notes

### Claude Code + RALPH Loop

User plans to use Claude Code with the RALPH loop (Read, Ask, Log, Plan, Hypothesize—or similar iterative AI-assisted development).

### AI Tool Considerations

- Claude generates reliable Kotlin/Java code
- Claude generates very clean Go code (simpler language = less ambiguity)
- Healthcare domain knowledge in prompts will be critical
- Agent prompts will need heavy iteration and version control

---

## 11. Summary: Where We Are

| Area | Status |
|------|--------|
| Problem definition | ✅ Clear |
| User requirements | ✅ Clear |
| Domain expertise | ✅ Strong (user is expert) |
| Target users | ✅ Defined (RCM staff) |
| Backend language | ❓ Leaning Kotlin, Go still considered |
| Frontend | ❓ Likely React + TypeScript |
| Rule representation | ❓ Needs design—hybrid approach likely |
| Agent architecture | ❓ Conceptual—needs detailed design |
| MVP scope | ❓ Not defined |
| Timeline | ❓ Not discussed |

---

## 12. Suggested Next Steps

1. **Define MVP scope** — Which rule type to start with? What's the simplest valuable thing?

2. **Design rule representation** — Especially for payer coverage rules where no standard exists

3. **Prototype one agent** — Pick the simplest (maybe SNOMED expert) and build end-to-end

4. **Decide backend language** — Kotlin (safe) vs Go (learning opportunity)

5. **Sketch the authoring UX** — What do users actually interact with?

---

## Appendix A: Comparison Tables

### Backend Language Comparison

```
                    Kotlin/JVM          Go                  Python
                    ──────────────────────────────────────────────────
User expertise      ████████████        ░░░░░░░░░░░░        ████░░░░░░░░
FHIR ecosystem      ████████████        ████░░░░░░░░        ████████░░░░
AI agent support    ████████░░░░        ████████░░░░        ████████████
Performance         ████████████        ████████████        ████░░░░░░░░
Concurrency model   ████████░░░░        ████████████        ████░░░░░░░░
Learning curve      ████████████        ████████░░░░        ████████████
                    (none for user)     (new language)      (some exp)
```

### Rule Representation Comparison

```
                    FHIR-Native         Custom DSL          Hybrid
                    ──────────────────────────────────────────────────
Interoperability    ████████████        ████░░░░░░░░        ████████░░░░
Expressiveness      ████░░░░░░░░        ████████████        ████████████
Ecosystem support   ████████████        ░░░░░░░░░░░░        ████████░░░░
Payer rule fit      ████░░░░░░░░        ████████████        ████████████
Terminology fit     ████████████        ████████░░░░        ████████████
Implementation cost ████████░░░░        ████░░░░░░░░        ████████░░░░
```

---

## Appendix B: Example Rule Formats (For Discussion)

### Terminology Mapping (FHIR ConceptMap Style)

```json
{
  "resourceType": "ConceptMap",
  "id": "sdoh-food-insecurity-mapping",
  "version": "1.0.0",
  "status": "active",
  "sourceUri": "http://loinc.org",
  "targetUri": "http://snomed.info/sct",
  "group": [{
    "source": "http://loinc.org",
    "target": "http://snomed.info/sct",
    "element": [{
      "code": "96777-8",
      "display": "Food insecurity risk",
      "target": [{
        "code": "733423003",
        "display": "Food insecurity",
        "equivalence": "equivalent",
        "dependsOn": [{
          "property": "answer",
          "value": "LA28397-0"
        }]
      }]
    }]
  }]
}
```

### Payer Coverage Rule (Custom DSL Concept)

```yaml
rule:
  id: aetna-cgm-coverage-2024
  version: 1.2.0
  payer: Aetna
  plan_types: [Commercial, Medicare Advantage]
  effective_date: 2024-01-01
  termination_date: 2024-12-31
  
  service:
    cpt: [95249, 95250, 95251]
    description: Continuous Glucose Monitoring
  
  coverage:
    conditions:
      - diagnosis:
          codes: [E11.*, E10.*]
          description: Diabetes mellitus
      - OR:
          - clinical_criteria:
              description: History of hypoglycemia
              documentation: Required
          - clinical_criteria:
              description: A1C not at goal despite therapy
              documentation: Required
      - prior_authorization:
          required: true
          validity_days: 365
    
    modifiers:
      - code: "95"
        condition: "telehealth_synchronous"
    
    frequency:
      max_units: 1
      per_period: 365 days
      per: patient_device_combination
  
  source:
    document: "Aetna Medical Policy #0873"
    page: 12
    url: "https://..."
    retrieved_date: 2024-06-15
```

### Clinical Guideline Rule (CQL-ish)

```
library CGMCoverage version '1.0.0'

using FHIR version '4.0.1'

context Patient

define "Has Diabetes":
  exists([Condition: "Diabetes"] C
    where C.clinicalStatus ~ 'active')

define "On Intensive Insulin Therapy":
  exists([MedicationRequest: "Insulin"] M
    where M.status = 'active'
    and M.dosageInstruction.timing.frequency >= 3)

define "Has Documented Hypoglycemia":
  exists([Condition: "Hypoglycemia"] C
    where C.onset during Interval[@2023-01-01, Today])

define "A1C Above Goal":
  exists([Observation: "HbA1c"] O
    where O.effective during Interval[Today - 6 months, Today]
    and O.value > 7 '%')

define "CGM Medically Necessary":
  "Has Diabetes"
  and "On Intensive Insulin Therapy"
  and ("Has Documented Hypoglycemia" or "A1C Above Goal")
```

---

*End of briefing document. This represents exploration, not decisions.*
