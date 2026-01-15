# Knowledge Studio: Architecture Diagrams

**Purpose:** Visual reference diagrams for system architecture discussions  
**Status:** DRAFT - All diagrams represent options under consideration, not decisions

---

## Diagram 1: High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           KNOWLEDGE STUDIO                                   │
│                     Clinical Knowledge Management Platform                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         PRESENTATION LAYER                             │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                    │ │
│  │  │  Web App    │  │  Desktop    │  │   Mobile    │                    │ │
│  │  │  (React)    │  │  (Tauri?)   │  │  (Maybe?)   │                    │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                    │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│                                    ▼                                         │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                          API LAYER                                     │ │
│  │                    (Kotlin/Spring or Go?)                              │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │ │
│  │  │  Knowledge   │  │   Agent      │  │ Distribution │                 │ │
│  │  │   CRUD       │  │   Runtime    │  │     API      │                 │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘                 │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│         ┌──────────────────────────┼──────────────────────────┐             │
│         ▼                          ▼                          ▼             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐ │
│  │   Knowledge     │  │    Agent        │  │      External Services      │ │
│  │   Repository    │  │    Services     │  │                             │ │
│  │                 │  │                 │  │  ┌─────────┐ ┌───────────┐  │ │
│  │  - PostgreSQL   │  │  - Coordinator  │  │  │ Claude  │ │Terminology│  │ │
│  │  - FHIR Store   │  │  - Experts      │  │  │   API   │ │  Server   │  │ │
│  │  - Version Ctrl │  │  - QA           │  │  └─────────┘ │(VSAC/NLM) │  │ │
│  │                 │  │                 │  │              └───────────┘  │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Diagram 2: Multi-Agent Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AGENT ORCHESTRATION LAYER                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                    ┌───────────────────────────────────┐                    │
│                    │    CURATION COORDINATOR AGENT     │                    │
│                    │                                   │                    │
│                    │  • Receives source documents      │                    │
│                    │  • Routes to appropriate experts  │                    │
│                    │  • Synthesizes expert outputs     │                    │
│                    │  • Manages workflow state         │                    │
│                    └───────────────┬───────────────────┘                    │
│                                    │                                         │
│         ┌──────────────────────────┼──────────────────────────┐             │
│         │                          │                          │             │
│         ▼                          ▼                          ▼             │
│  ┌─────────────────┐    ┌─────────────────────┐    ┌─────────────────┐     │
│  │    DOCUMENT     │    │    DOMAIN EXPERT    │    │   QA AGENT      │     │
│  │ ANALYSIS AGENT  │    │       POOL          │    │                 │     │
│  │                 │    │                     │    │ • Consistency   │     │
│  │ • PDF parsing   │    │                     │    │ • Completeness  │     │
│  │ • Structure ID  │    │                     │    │ • Terminology   │     │
│  │ • Code extract  │    │                     │    │ • Logic errors  │     │
│  └─────────────────┘    │                     │    └─────────────────┘     │
│                         │  ┌───────────────┐  │                             │
│                         │  │ SNOMED Expert │  │                             │
│                         │  └───────────────┘  │                             │
│                         │  ┌───────────────┐  │                             │
│                         │  │ ICD-10 Expert │  │                             │
│                         │  └───────────────┘  │                             │
│                         │  ┌───────────────┐  │                             │
│                         │  │CPT/HCPCS Expert│ │                             │
│                         │  └───────────────┘  │                             │
│                         │  ┌───────────────┐  │                             │
│                         │  │ Payer Policy  │  │                             │
│                         │  │    Expert     │  │                             │
│                         │  └───────────────┘  │                             │
│                         │  ┌───────────────┐  │                             │
│                         │  │   Clinical    │  │                             │
│                         │  │   Guideline   │  │                             │
│                         │  │    Expert     │  │                             │
│                         │  └───────────────┘  │                             │
│                         └─────────────────────┘                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

Each agent is essentially:
┌─────────────────────────────────────────────────────────────────────────────┐
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐           │
│  │  System Prompt  │ + │  Tools/APIs     │ + │ Structured      │ = Agent   │
│  │  (Domain        │   │  (Terminology   │   │ Output Parser   │           │
│  │   expertise)    │   │   lookup, etc)  │   │                 │           │
│  └─────────────────┘   └─────────────────┘   └─────────────────┘           │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Diagram 3: Human-in-the-Loop Curation Workflow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     KNOWLEDGE CURATION WORKFLOW                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌───────────────────────┐                                                 │
│   │  SOURCE DOCUMENT      │                                                 │
│   │                       │                                                 │
│   │  • Payer policy PDF   │                                                 │
│   │  • Clinical guideline │                                                 │
│   │  • LCD/NCD document   │                                                 │
│   │  • Screening tool     │                                                 │
│   └───────────┬───────────┘                                                 │
│               │                                                              │
│               ▼                                                              │
│   ┌───────────────────────────────────────────────────────────────────────┐ │
│   │                    AI AGENT PROCESSING                                │ │
│   │                                                                       │ │
│   │  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐           │ │
│   │  │ Analyze │───▶│ Consult │───▶│Synthesize───▶│   QA    │           │ │
│   │  │   Doc   │    │ Experts │    │  Draft  │    │ Review  │           │ │
│   │  └─────────┘    └─────────┘    └─────────┘    └─────────┘           │ │
│   │                                                                       │ │
│   └───────────────────────────────────┬───────────────────────────────────┘ │
│                                       │                                      │
│                                       ▼                                      │
│   ┌───────────────────────────────────────────────────────────────────────┐ │
│   │                    HUMAN REVIEW QUEUE                                 │ │
│   │                                                                       │ │
│   │  ┌─────────────────────────────────────────────────────────────────┐ │ │
│   │  │  Draft Rule Display                                             │ │ │
│   │  │  ├── Structured rule content                                    │ │ │
│   │  │  ├── AI confidence score                                        │ │ │
│   │  │  ├── Flags and warnings                                         │ │ │
│   │  │  ├── Source document link                                       │ │ │
│   │  │  └── Expert reasoning/justification                             │ │ │
│   │  └─────────────────────────────────────────────────────────────────┘ │ │
│   │                                                                       │ │
│   └───────────────────────────────────┬───────────────────────────────────┘ │
│                                       │                                      │
│                    ┌──────────────────┴──────────────────┐                  │
│                    │                                     │                  │
│                    ▼                                     ▼                  │
│        ┌───────────────────────┐           ┌───────────────────────┐       │
│        │       APPROVE         │           │   EDIT / REJECT       │       │
│        │                       │           │                       │       │
│        │  Rule is correct,     │           │  Human corrects       │       │
│        │  ready for publish    │           │  errors, adds info    │       │
│        └───────────┬───────────┘           └───────────┬───────────┘       │
│                    │                                   │                    │
│                    │                                   │                    │
│                    ▼                                   │                    │
│        ┌───────────────────────┐                      │                    │
│        │    VERSION & PUBLISH  │◄─────────────────────┘                    │
│        │                       │    (after re-review if edited)            │
│        │  • Assign version     │                                           │
│        │  • Add to repository  │                                           │
│        │  • Notify subscribers │                                           │
│        │  • Audit log entry    │                                           │
│        └───────────────────────┘                                           │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  FEEDBACK LOOP: Human corrections improve future AI curation          │ │
│  │  • Track which agents produce most corrections                        │ │
│  │  • Build dataset of corrections for few-shot examples                 │ │
│  │  • Identify systematic errors in prompts                              │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Diagram 4: Knowledge Distribution Model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     KNOWLEDGE DISTRIBUTION                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                    KNOWLEDGE REPOSITORY                                 ││
│  │                                                                         ││
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   ││
│  │  │ Terminology │  │  Coverage   │  │  Clinical   │  │   Audit     │   ││
│  │  │   Maps      │  │   Rules     │  │  Guidelines │  │   History   │   ││
│  │  │             │  │             │  │             │  │             │   ││
│  │  │ ConceptMap  │  │ Custom DSL  │  │ CQL or DSL  │  │ Version log │   ││
│  │  │ ValueSet    │  │ (TBD)       │  │ (TBD)       │  │ Changes     │   ││
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘   ││
│  │                                                                         ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                       │                                      │
│                                       │                                      │
│       ┌───────────────────────────────┼───────────────────────────────┐     │
│       │                               │                               │     │
│       ▼                               ▼                               ▼     │
│  ┌─────────────┐              ┌─────────────┐              ┌─────────────┐ │
│  │  FHIR API   │              │  Custom API │              │ Bulk Export │ │
│  │             │              │             │              │             │ │
│  │ Standard    │              │ Rule        │              │ Offline     │ │
│  │ resources:  │              │ packages:   │              │ bundles:    │ │
│  │ ConceptMap  │              │ Coverage    │              │ Full repo   │ │
│  │ ValueSet    │              │ Clinical    │              │ snapshots   │ │
│  │ Library     │              │ Payer-spec  │              │ Delta       │ │
│  │             │              │             │              │ updates     │ │
│  └──────┬──────┘              └──────┬──────┘              └──────┬──────┘ │
│         │                            │                            │        │
│         │                            │                            │        │
│         └────────────────────────────┼────────────────────────────┘        │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                         DOWNSTREAM CONSUMERS                            ││
│  │                                                                         ││
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   ││
│  │  │   Billing   │  │   Claims    │  │    CDI      │  │   Other     │   ││
│  │  │   Systems   │  │ Scrubbers   │  │   Tools     │  │   Vendors   │   ││
│  │  │             │  │             │  │             │  │             │   ││
│  │  │ Real-time   │  │ Batch       │  │ Lookup      │  │ Integration │   ││
│  │  │ API calls   │  │ downloads   │  │ reference   │  │ partners    │   ││
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘   ││
│  │                                                                         ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Diagram 5: Agent Orchestration Options

### Option A: Simple Function Orchestration (Code controls flow)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│    Kotlin/Go Code (Coordinator)                                             │
│    ┌────────────────────────────────────────────────────────────────────┐   │
│    │                                                                    │   │
│    │  fun curateDocument(doc: Document): CurationResult {              │   │
│    │      val analysis = documentAgent.analyze(doc)      // Step 1     │   │
│    │                                                                    │   │
│    │      val experts = selectExperts(analysis)          // Step 2     │   │
│    │      val outputs = experts.map { it.analyze() }     // Parallel   │   │
│    │                                                                    │   │
│    │      val draft = synthesize(analysis, outputs)      // Step 3     │   │
│    │      val qa = qaAgent.review(draft)                 // Step 4     │   │
│    │                                                                    │   │
│    │      return CurationResult(draft, qa)               // Done       │   │
│    │  }                                                                 │   │
│    │                                                                    │   │
│    └────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│    Pros: Predictable, debuggable, testable, full control                    │
│    Cons: Rigid, doesn't adapt to novel situations                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Option B: Agent-as-Tool (LLM decides flow)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│    Coordinator Agent (LLM with tools)                                       │
│    ┌────────────────────────────────────────────────────────────────────┐   │
│    │                                                                    │   │
│    │  System: "You are a curation coordinator. Use these tools..."     │   │
│    │                                                                    │   │
│    │  Tools available:                                                  │   │
│    │    - analyze_document(doc) → analysis                             │   │
│    │    - consult_snomed_expert(concepts) → recommendations            │   │
│    │    - consult_icd10_expert(concepts) → recommendations             │   │
│    │    - consult_payer_expert(policy) → coverage_rule                 │   │
│    │    - run_qa_check(draft) → qa_result                              │   │
│    │                                                                    │   │
│    │  LLM decides which tools to call and in what order                │   │
│    │                                                                    │   │
│    └────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│    Pros: Flexible, can adapt to novel document types                        │
│    Cons: Less predictable, higher token cost, harder to debug               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Option C: State Machine (Explicit states and transitions)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│    ┌──────────────┐     ┌──────────────┐     ┌──────────────┐              │
│    │   ANALYZE    │────▶│   CONSULT    │────▶│  SYNTHESIZE  │              │
│    │   DOCUMENT   │     │   EXPERTS    │     │    DRAFT     │              │
│    └──────────────┘     └──────────────┘     └──────────────┘              │
│                                                     │                       │
│                                                     ▼                       │
│    ┌──────────────┐     ┌──────────────┐     ┌──────────────┐              │
│    │   COMPLETE   │◀────│    HUMAN     │◀────│   QA CHECK   │              │
│    │              │     │    REVIEW    │     │              │              │
│    └──────────────┘     └──────────────┘     └──────────────┘              │
│           ▲                    │                                            │
│           │                    │ (if rejected)                              │
│           │                    ▼                                            │
│           │             ┌──────────────┐                                    │
│           └─────────────│    REVISE    │                                    │
│                         └──────────────┘                                    │
│                                                                              │
│    Pros: Clear audit trail, can persist/resume, explicit states             │
│    Cons: More boilerplate, state explosion if complex                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Diagram 6: Three Rule Types Comparison

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     THREE KNOWLEDGE ASSET TYPES                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  TYPE 1: TERMINOLOGY MAPPINGS                                               │
│  ─────────────────────────────                                              │
│  ┌─────────────────┐      ┌─────────────────┐                               │
│  │ LOINC Question  │      │ SNOMED CT Code  │                               │
│  │ +               │ ───▶ │ +               │                               │
│  │ Answer Value    │      │ ICD-10 Code     │                               │
│  └─────────────────┘      └─────────────────┘                               │
│                                                                              │
│  Characteristics: Deterministic, high volume, changes quarterly             │
│  Format: FHIR ConceptMap (probably)                                         │
│                                                                              │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                              │
│  TYPE 2: PAYER COVERAGE RULES                                               │
│  ────────────────────────────                                               │
│  ┌─────────────────────────────────────────────────────┐                    │
│  │ IF diagnosis IN (codes...)                          │                    │
│  │ AND procedure = code                                │                    │
│  │ AND patient meets criteria                          │      ┌───────────┐ │
│  │ AND prior_auth approved                             │ ───▶ │ Covered   │ │
│  │ THEN                                                │      │ + Modifier│ │
│  │   covered with modifier X                           │      │ + Limits  │ │
│  │   max N units per period                            │      └───────────┘ │
│  └─────────────────────────────────────────────────────┘                    │
│                                                                              │
│  Characteristics: Complex logic, payer-specific, changes frequently         │
│  Format: Custom DSL (no standard exists)                                    │
│                                                                              │
│  ═══════════════════════════════════════════════════════════════════════   │
│                                                                              │
│  TYPE 3: CLINICAL GUIDELINE LOGIC                                           │
│  ────────────────────────────────                                           │
│  ┌─────────────────────────────────────────────────────┐                    │
│  │ Per [Guideline Name]:                               │                    │
│  │                                                     │      ┌───────────┐ │
│  │ IF patient has condition X                          │      │ Medically │ │
│  │ AND has not responded to treatments A, B, C         │ ───▶ │ Necessary │ │
│  │ AND meets additional criteria                       │      │ + Docs    │ │
│  │ THEN treatment Y is medically necessary             │      │ Required  │ │
│  │ DOCUMENT: specific items needed                     │      └───────────┘ │
│  └─────────────────────────────────────────────────────┘                    │
│                                                                              │
│  Characteristics: Guideline-derived, for documentation not treatment        │
│  Format: CQL or custom DSL (TBD)                                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Diagram 7: Tech Stack Decision Space

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     BACKEND LANGUAGE DECISION                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                        User's Expertise                                      │
│                              │                                               │
│                              │                                               │
│    ┌─────────────────────────┼─────────────────────────┐                    │
│    │                         │                         │                    │
│    ▼                         ▼                         ▼                    │
│                                                                              │
│  KOTLIN/JVM              GO                      PYTHON                     │
│  ───────────            ────                    ────────                    │
│  ✅ User expertise      ⚠️ Must learn           ✅ AI learning done here    │
│  ✅ HAPI FHIR           ❌ No FHIR libs         ✅ Best AI libs             │
│  ✅ Database tools      ✅ Simple concurrency   ❌ Worst performance        │
│  ✅ Enterprise ready    ✅ Single binary        ❌ Async is awkward         │
│  ⚠️ More verbose        ✅ Fast compilation     ❌ Not user's strength      │
│                         ✅ Clean AI codegen                                 │
│                                                                              │
│  LEANING: Kotlin        ALTERNATIVE: Go         UNLIKELY: Python            │
│  (safe choice)          (learning opportunity)  (keep for experiments)      │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                     RULE FORMAT DECISION                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐         │
│  │   FHIR-NATIVE   │    │   CUSTOM DSL    │    │     HYBRID      │         │
│  │                 │    │                 │    │                 │         │
│  │ ConceptMap      │    │ Your own format │    │ FHIR for        │         │
│  │ CQL             │    │ optimized for   │    │ terminology     │         │
│  │ ValueSet        │    │ your use cases  │    │                 │         │
│  │ Library         │    │                 │    │ Custom for      │         │
│  │                 │    │                 │    │ coverage rules  │         │
│  ├─────────────────┤    ├─────────────────┤    ├─────────────────┤         │
│  │ ✅ Interop      │    │ ✅ Expressive   │    │ ✅ Best of both │         │
│  │ ✅ Ecosystem    │    │ ✅ Payer fit    │    │ ⚠️ Two formats  │         │
│  │ ❌ Payer rules  │    │ ❌ No ecosystem │    │ ⚠️ More complex │         │
│  │    don't fit    │    │ ❌ Must document│    │                 │         │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘         │
│                                                                              │
│                              LEANING: Hybrid                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*All diagrams are drafts representing options under consideration.*
