# Knowledge Studio: Quick Reference Card

**Use this for fast context loading in new sessions**

---

## 🎯 One-Liner

Building an AI-assisted clinical knowledge authoring platform for revenue cycle staff to curate, version, and distribute terminology mappings, payer coverage rules, and clinical guideline logic.

---

## 👤 User Profile

| Attribute | Value |
|-----------|-------|
| Java/DB expertise | **Expert** |
| FHIR/Terminology | **Expert** |
| Python/AI | Learning |
| Go | Never used |
| Role focus | Backend/database |

---

## 🚫 What This Is NOT

- ❌ Clinical decision support for clinicians
- ❌ EHR integration / SMART on FHIR
- ❌ Runtime claims adjudication
- ❌ Patient-facing application

---

## ✅ What This IS

- ✅ Knowledge authoring tool
- ✅ Version control for clinical rules
- ✅ Distribution API for downstream systems
- ✅ AI-assisted curation (human-in-the-loop)
- ✅ For revenue cycle / billing staff

---

## 📦 Three Rule Types

1. **Terminology Maps**: LOINC screening → SNOMED/ICD-10 (deterministic)
2. **Payer Rules**: Coverage conditions, modifiers, limits (complex, no standard)
3. **Clinical Logic**: Guideline-derived medical necessity (for documentation)

---

## ❓ Major Open Questions

### Must Decide
- [ ] Backend: Kotlin (safe) vs Go (learn)?
- [ ] Rule format: FHIR-native vs custom DSL vs hybrid?
- [ ] MVP scope: Which rule type first?

### Needs Design
- [ ] Agent orchestration pattern
- [ ] Authoring UX for non-technical users
- [ ] Payer rule representation (no standard exists)

### Can Defer
- [ ] Desktop client necessity
- [ ] Mobile (probably not needed)
- [ ] Fine-tuning vs few-shot for agents

---

## 🏗️ Likely Stack (Not Decided)

```
Backend:      Kotlin + Spring (or Go?)
Database:     PostgreSQL
Frontend:     React + TypeScript
FHIR:         HAPI FHIR libraries
AI:           Claude API
Terminology:  External (VSAC) or self-hosted
```

---

## 🤖 Agent Architecture (Conceptual)

```
Coordinator Agent
    ├── Document Analysis Agent
    ├── SNOMED Expert Agent
    ├── ICD-10 Expert Agent
    ├── CPT/HCPCS Expert Agent
    ├── Payer Policy Expert Agent
    ├── Clinical Guideline Expert Agent
    └── QA Agent
```

Human reviews all agent outputs before publish.

---

## 📁 Related Files

- `knowledge-studio-briefing.md` — Full context document
- `knowledge-studio-diagrams.md` — Architecture diagrams

---

## 💡 Key Insight

User has deep healthcare domain expertise (FHIR, terminology, RCM). The challenge is NOT healthcare knowledge—it's:
1. Choosing the right app architecture
2. Designing multi-agent orchestration
3. Creating a rule format for payer rules (where no standard exists)

---

## 🚀 Suggested Starting Points

**If exploring architecture:** Start with rule representation design  
**If ready to code:** Prototype one agent (suggest SNOMED expert)  
**If undecided on language:** Build same small component in Kotlin and Go, compare

---

*Last updated: January 2026*
