# Newcastle System Design Brief

**Purpose**: This document captures all system requirements and design thinking for Newcastle. Use this to continue architecture work in a new Claude session.

---

## Executive Summary

**Newcastle** is a B2B SaaS platform that aggregates, normalizes, and delivers payer billing rules to healthcare provider revenue cycle management (RCM) teams. It solves the problem of fragmented, hard-to-track payer rules that cause lost revenue and denied claims.

**Value Proposition**: 5-40% revenue uplift, fewer denied claims, reduced administrative burden.

**Target Users**: RCM teams, clinical coders, and provider administrators at health systems.

---

## Problem Statement

Healthcare providers must track hundreds of payer-specific billing rules:
- Rules are scattered across multiple payer portals (CMS, BCBS, Aetna, state Medicaid, etc.)
- Rules are published as PDFs/HTML at different times with no central notification
- Missing or outdated rules leads to:
  - Incorrect coding → denied claims → lost revenue
  - Rework costs and administrative burden
  - Delayed reimbursement

**Current state**: Providers use encoders (3M, Optum) for billing codes, but these don't always have the latest payer-specific rules or update in time.

---

## Solution Overview

A subscription service with:
1. **Web Portal** - Research tool for RCM teams to search/browse rules
2. **Desktop Widget** - Quick lookup tool for clinical coders during claim prep
3. **AI Assistant** - Natural language Q&A over payer rules
4. **Smart Alerts** - Prioritized notifications when rules change

**Key differentiator**: AI-native architecture for document ingestion, normalization, and retrieval.

---

## Data Architecture

### Input Sources

| Source Type | Examples | Format | Update Frequency |
|-------------|----------|--------|------------------|
| **Payer Rules** | CMS transmittals, commercial payer policies, state Medicaid bulletins | PDF, HTML | Weekly-Monthly |
| **Reference Standards** | CPT, ICD-10, HCPCS, RxNorm, SNOMED | Structured data | Quarterly-Annual |
| **Industry Intel** | AAPC forums, coder communities | Web scraping | Continuous |
| **Provider Config** | Payer mix, service lines, alert preferences | User input | On-demand |

### Data Pipeline (AI-Native)

```
┌─────────────────────────────────────────────────────────────────┐
│                     INGESTION LAYER                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │ Payer    │  │ Reference│  │ Industry │  │ Provider │        │
│  │ Portals  │  │ Standards│  │ Sources  │  │ Config   │        │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘        │
│       └─────────────┴─────────────┴─────────────┘              │
│                           │                                     │
│                    ┌──────▼──────┐                              │
│                    │  RAW STORE  │  (S3/blob: PDFs, HTML)       │
│                    └──────┬──────┘                              │
└───────────────────────────┼─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                   PROCESSING LAYER                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Doc Parsing  │  │ Entity       │  │ Change       │          │
│  │ (LLM-based)  │  │ Extraction   │  │ Detection    │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                 │                 │                   │
│  - PDF → text             - Payer ID        - Semantic diff     │
│  - Table extraction       - Effective date  - Version tracking  │
│  - Section parsing        - Codes affected  - Change summary    │
│                           - Rule type                           │
│                    ┌──────▼──────┐                              │
│                    │ NORMALIZED  │  (Postgres: structured)      │
│                    │ RULES STORE │                              │
│                    └──────┬──────┘                              │
└───────────────────────────┼─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                   INTELLIGENCE LAYER                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Impact       │  │ Embeddings/  │  │ Alert        │          │
│  │ Scoring      │  │ RAG Index    │  │ Generation   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│         │                 │                 │                   │
│  - Provider payer mix     - Vector DB       - Change detected   │
│  - Service line volume    - Semantic search - Impact scored     │
│  - Revenue estimate       - Citation links  - Notification sent │
└───────────────────────────┼─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                   SERVING LAYER                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Web Portal   │  │ Desktop      │  │ API          │          │
│  │ (React/Next) │  │ Widget       │  │ (REST/GQL)   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐                            │
│  │ AI Chat      │  │ Smart Alerts │                            │
│  │ (RAG-based)  │  │ (Email/Push) │                            │
│  └──────────────┘  └──────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
```

### Content Schema (Hybrid Approach)

**Core Entity: Rule**
```
Rule
├── id (UUID)
├── payer_id (FK → Payer)
├── rule_type (enum: coverage, coding, billing, payment, prior_auth)
├── effective_date
├── end_date (nullable)
├── title
├── summary (LLM-generated)
├── content_raw (full text)
├── content_structured (JSON - extracted fields)
├── source_document_id (FK → SourceDocument)
├── codes_affected (array: CPT, ICD-10, HCPCS codes)
├── service_types (array)
├── change_type (enum: new, updated, deprecated)
├── change_summary (LLM-generated diff)
├── created_at
├── updated_at
└── embedding (vector for RAG)
```

**Supporting Entities**
```
Payer
├── id, name, type (federal, commercial, medicaid)
├── regions (array)
├── portal_urls (array)
└── update_frequency

SourceDocument
├── id, payer_id, url, file_path
├── document_type (pdf, html)
├── fetched_at, hash (for change detection)
└── raw_content

ProviderConfig
├── provider_id, payer_ids (array)
├── service_lines (array)
├── alert_preferences
└── user_ids
```

---

## Technical Considerations

### AI/LLM Integration Points

| Function | Approach | Model Recommendation |
|----------|----------|----------------------|
| PDF parsing | LLM extraction with structured prompts | Claude/GPT-4 |
| Entity extraction | Few-shot prompting with examples | Claude |
| Change detection | Semantic diff using embeddings | Embedding model + LLM summary |
| Impact scoring | Rule-based + LLM reasoning | Hybrid |
| RAG retrieval | Vector search + reranking | Embedding + Claude |
| Chat interface | RAG with citations | Claude |

### Key Technical Challenges

1. **PDF parsing accuracy** - Healthcare PDFs are complex (tables, multi-column, headers)
2. **Change detection** - Distinguishing semantic changes from formatting changes
3. **Entity resolution** - Mapping payer-specific terminology to standard codes
4. **Latency** - Near real-time for chat; batch acceptable for alerts
5. **Citation accuracy** - Must link answers to source documents

### Suggested Tech Stack

| Layer | Technology | Rationale |
|-------|------------|-----------|
| Frontend | Next.js + React | SSR, fast iteration |
| Backend API | Python FastAPI | LLM ecosystem, async support |
| Database | Postgres + pgvector | Relational + vector in one |
| Vector store | pgvector or Pinecone | RAG retrieval |
| Document store | S3 | Raw PDFs/HTML |
| LLM | Claude API | Best for extraction/reasoning |
| Job queue | Celery + Redis | Async document processing |
| Hosting | Vercel (FE) + Railway/Render (BE) | Fast to deploy |

---

## MVP Scope

### Phase 1: Core Research Tool (MVP)

**In Scope**:
- Web portal with search/browse
- 5-10 payers (Medicare + top commercials in one region)
- Manual document upload (not automated scraping)
- Basic change detection (document-level, not semantic)
- Simple alert emails

**Out of Scope for MVP**:
- Desktop widget
- AI chat
- Automated scraping
- Impact scoring
- Full semantic change detection

### MVP User Stories

1. As an RCM analyst, I can search for rules by payer, code, or keyword
2. As an RCM analyst, I can view the source document for any rule
3. As an admin, I can configure my organization's payer mix
4. As an RCM analyst, I receive email alerts when rules change for my payers

---

## Prototype Ideas

### n8n Prototype (Document Ingestion)
- Trigger: Manual upload or scheduled fetch
- Action 1: Fetch PDF from URL
- Action 2: Send to Claude for extraction (structured output)
- Action 3: Store extracted data in Postgres/Airtable
- Action 4: Generate embedding, store in vector DB
- Action 5: Send Slack/email notification if change detected

### Figma UI Mocks
- Dashboard: Recent changes, my payers, search bar
- Search results: Rule cards with payer, effective date, summary
- Rule detail: Full content, source document link, related rules
- Settings: Payer selection, alert preferences

---

## Business Context

### Target Customer
- Mid-size health systems (100-500 beds)
- RCM departments (5-20 people)
- Decision maker: RCM Director or VP Revenue Cycle

### Pricing Model (TBD)
- Per-provider subscription
- Tiered by number of payers tracked
- Estimated range: $500-2000/month

### Competitors
- 3M, Optum (encoder tools - not exactly this)
- Availity (payer connectivity - different focus)
- No direct competitor for rules aggregation (gap in market)

### Exit Potential
- Strategic acquirers: 3M, Optum, Waystar, R1, Change Healthcare
- Timeline: 5-7 years typical for healthcare SaaS
- Multiples: 4-10x ARR depending on growth

---

## Next Steps for Architecture Work

1. **Data model refinement** - Finalize schema based on sample rules
2. **PDF extraction testing** - Run 10-20 sample PDFs through Claude, measure accuracy
3. **RAG prototype** - Build simple vector search over extracted rules
4. **UI wireframes** - Figma mocks for core flows
5. **n8n pipeline** - End-to-end prototype of document ingestion
6. **Tech stack decision** - Confirm choices based on team skills

---

## Reference Materials

- Final markitecture: `docs/diagrams/output/newcastle-markitecture-final.png`
- C4 context diagram: `docs/diagrams/output/newcastle-context-final.png`
- Medispan GPI research: `research/medispan-gpi-deep-dive.md`
- PRD for diagrams: `docs/diagrams/prd-diagram-refinement.md`

---

## Questions to Resolve

1. **Data sourcing** - Partner with payers? Manual curation? Scraping?
2. **Accuracy liability** - What happens if we're wrong? Disclaimers?
3. **Pricing** - Per seat? Per payer? Usage-based?
4. **Sales motion** - Direct? Channel partners? PLG?
5. **Initial market** - Geographic focus? Specialty focus?
