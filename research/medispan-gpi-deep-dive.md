# Medispan GPI Deep Dive

## Executive Summary

The Generic Product Identifier (GPI) is a **proprietary 14-character hierarchical drug classification system** owned by Wolters Kluwer (Medi-Span). It serves as the backbone for drug identification, formulary management, and pricing decisions across the US healthcare system—particularly in PBMs, pharmacy systems, and health plans.

---

## Part 1: Product & Business Perspective

### What Problems Does GPI Solve?

1. **Drug Identification & Grouping**: Classifies drugs from therapeutic category down to specific strength, enabling systematic organization of 300,000+ drug products

2. **Formulary Management**: PBMs use GPI hierarchy to build formularies, set co-pay tiers, and include/exclude drugs by therapeutic class, ingredient, or form

3. **Generic Substitution**: Identifies pharmaceutically equivalent products (same active ingredient, route, form, strength) regardless of manufacturer

4. **Pricing Analysis**: Links to AWP, WAC, NADAC, and other pricing benchmarks for reimbursement decisions

5. **Population Health Analytics**: Enables analysis of drug utilization patterns by therapeutic class

### Who Uses It?

| Customer Segment | Use Case |
|------------------|----------|
| **PBMs** | Formulary building, tier assignment, claims adjudication |
| **Health Plans/Payers** | Coverage decisions, utilization management |
| **Retail Pharmacies** | Product substitution, inventory management |
| **Pharmacy Software Vendors** | Core drug data integration |
| **EMR/EHR Systems** | Clinical decision support integration |
| **Wholesalers (McKesson, etc.)** | Ordering systems, inventory classification |
| **State Medicaid Programs** | Reimbursement rate setting |

### Business Model

Wolters Kluwer operates a **B2B data licensing model**:

- **Data Delivery Options**: Flat files, API, or Web Services
- **Update Frequency**: Regular updates (typically weekly/monthly) as drug landscape changes
- **Pricing**: Enterprise licensing—not publicly disclosed, negotiated per customer
- **Bundling**: Often sold with complementary Medi-Span products (drug pricing data, clinical screening, interoperability tools)

Medi-Span is positioned as the authoritative source for manufacturer-provided AWP (Average Wholesale Price), which gives it significant market leverage.

### Known Product Challenges

| Challenge | Description |
|-----------|-------------|
| **Proprietary Lock-in** | GPI is proprietary; switching costs are high once integrated |
| **No FDA Therapeutic Equivalency** | Hierarchy doesn't include FDA TE codes—can cause false substitution signals |
| **Biosimilar Gaps** | Each biosimilar gets its own GPI (different suffix), won't link to reference biologic |
| **Partial GPIs** | Supplements, vitamins, devices get catch-all codes (*asterisk names*), limiting usefulness |
| **Crosswalk Complexity** | GPI-to-GSN mapping creates one-to-many relationships, complicating multi-system integration |
| **AWP Accuracy Concerns** | AWP values can vary up to 20% between Medi-Span and FDB |

---

## Part 2: Market Landscape & Alternatives

### Primary Competitors

| Vendor | Product | Key Identifier |
|--------|---------|----------------|
| **First Databank (FDB)** | MedKnowledge | GSN (Generic Sequence Number) - 6 digits |
| **Cerner Multum** | Lexicon | Drug ID system |
| **IBM Micromedex** | Drug database | Own classification |
| **Gold Standard/Elsevier** | Clinical drug data | Own system |
| **AHFS** | Drug Information | AHFS Classification (ASHP) |

### GPI vs GSN (FDB) Comparison

| Aspect | GPI (Medi-Span) | GSN (First Databank) |
|--------|-----------------|----------------------|
| **Structure** | 14 chars, 7 hierarchical levels | 6 digits, flat |
| **Hierarchy** | Built-in therapeutic grouping | Requires separate HIC/ETC codes |
| **Maintenance** | More updates needed at each level | Fewer changes required |
| **Flexibility** | Can query at any hierarchy level | All-or-nothing identifier |
| **Market Position** | Strong in PBM/payer space | Strong in clinical/hospital space |

### Key Research Finding

A 2017 JAMIA study comparing FDB, Micromedex, and Multum for drug-drug interaction detection found:
- **79% of drug pairs were unique to only one knowledge base**
- Only **5% of pairs were in all three**
- Severity ranking agreement ranged from 57-68% between any two vendors
- Medi-Span declined to participate in the study

**Implication**: No single drug knowledge base is comprehensive—many organizations license multiple sources.

---

## Part 3: Technical Architecture Deep Dive

### The 14-Character Hierarchy

```
Position:  [1-2] [3-4] [5-6] [7-8] [9-10] [11-12] [13-14]
Level:       1     2     3     4      5       6       7
```

| Level | Positions | Name | Description | Example |
|-------|-----------|------|-------------|---------|
| 1 | 1-2 | Drug Group | Broadest therapeutic category | 58 = Antidepressants |
| 2 | 3-4 | Drug Class | Therapeutic subgroup | 20 = Tricyclic agents |
| 3 | 5-6 | Drug Subclass | Further refinement | 00 = Tricyclic agents |
| 4 | 7-8 | Drug Name (Base) | Generic ingredient base | 60 = Nortriptyline |
| 5 | 9-10 | Drug Name (Extended) | Salt form/extension | 10 = Nortriptyline HCL |
| 6 | 11-12 | Dosage Form | Route/form | 01 = Capsule |
| 7 | 13-14 | Strength | Specific strength | 05 = 10mg |

**Full Example**: `58-20-00-60-10-01-05` = Nortriptyline HCL Capsule 10mg

### Key Technical Properties

1. **Therapeutic Singularity**: A 14-character GPI resides in exactly one therapeutic classification (can't span categories)

2. **Alphanumeric Support**: Since July 2013, positions 11-12 support letters to expand dosage form capacity

3. **Pharmaceutical Equivalence Only**: Products with same GPI-14 are identical in:
   - Active ingredient(s)
   - Route of administration
   - Dosage form
   - Strength/concentration

4. **What GPI Does NOT Include**:
   - Package size
   - Package type
   - Manufacturer
   - Brand vs generic status
   - FDA therapeutic equivalency (Orange Book)
   - NDC (separate crosswalk required)

### Data Relationships

```
NDC (11-digit) ──┬──> GPI-14 (pharmaceutically equivalent products)
                 │
                 ├──> DDID (Drug Descriptor ID) ──> Brand/generic linking
                 │
                 └──> Pricing (AWP, WAC, NADAC, etc.)
```

**NDC to GPI is Many-to-One**: Multiple NDCs (different manufacturers, package sizes) map to one GPI-14.

### Partial GPI Pattern

Non-standard products get "catch-all" codes identified by asterisks:

```
*Nutritional Supplement Caps*
*Blood Pressure Monitoring – Device*
*Dietary Management Product*
```

These create **one-to-many crosswalk problems** and are used for:
- Vitamins/supplements
- Medical foods
- Dietary management products
- Medical devices

### Integration Patterns

**Delivery Options**:
1. **Flat Files**: Relational database format for bulk integration
2. **API**: RESTful services for real-time lookups
3. **Web Services**: SOAP-based integration

**Common Integration Points**:
- NDC → GPI crosswalk table
- GPI → Therapeutic class descriptions
- GPI → Pricing data (AWP, WAC, etc.)
- GPI → Clinical screening rules (DDI, allergies, etc.)

---

## Part 4: Strengths & Limitations Summary

### Strengths

| Strength | Detail |
|----------|--------|
| **Industry Standard** | De facto standard in PBM/payer space |
| **Hierarchical Flexibility** | Query at any level (class, ingredient, form, strength) |
| **Comprehensive Coverage** | 300,000+ products with regular updates |
| **AWP Authority** | Only reliable published source for manufacturer-provided AWP |
| **Mature Ecosystem** | Well-understood by pharmacy systems, PBMs, health plans |
| **Interoperability Tools** | Built-in crosswalks to NDC, HCPCS, DDID |

### Limitations

| Limitation | Detail |
|------------|--------|
| **Proprietary/Costly** | Enterprise licensing, vendor lock-in |
| **No TE Codes** | Missing FDA therapeutic equivalency—substitution gaps |
| **Biosimilar Blind Spot** | Can't link biosimilars to reference products |
| **Partial GPI Weakness** | Supplements, devices poorly classified |
| **Single Therapeutic Class** | Multi-indication drugs can only be in one class |
| **Cross-vendor Variation** | AWP differs up to 20% vs FDB |
| **Maintenance Overhead** | Hierarchy requires more frequent updates than flat systems |

---

## Part 5: Considerations for Development

If building a system that uses drug data, consider:

1. **Licensing Requirement**: GPI is not free—requires Wolters Kluwer contract
2. **NDC as Primary Key**: NDC is the universal identifier; GPI supplements it
3. **Multi-source Strategy**: Many enterprises license both Medi-Span and FDB
4. **Crosswalk Tables**: You'll need NDC↔GPI mapping updated regularly
5. **Partial GPI Handling**: Build logic for asterisk-wrapped codes
6. **Biosimilar Logic**: Separate data structure may be needed for biologic/biosimilar linking
7. **Update Cadence**: Plan for weekly/monthly data refreshes
8. **Alternative**: Consider RxNorm (NLM) as a free, public alternative for certain use cases

---

## Sources

- [Wolters Kluwer Medi-Span GPI Overview](https://www.wolterskluwer.com/en/solutions/medi-span/about/gpi)
- [Wikipedia: Generic Product Identifier](https://en.wikipedia.org/wiki/Generic_Product_Identifier)
- [PAAS National: Medi-Span GPI](https://paasnational.com/medi-span-generic-product-identifier/)
- [Pharmacy Healthcare Solutions: GPI vs GSN](https://phslrx.com/gpi-vs-gsn/)
- [Pharmacy Healthcare Solutions: Partial GPIs and FDB Supersets](https://phslrx.com/a-primer-on-partial-gpis-and-fdb-supersets/)
- [No World Borders: Drug Pricing & Classification Systems](https://noworldborders.com/2018/03/01/drug-pricing-classification-systems/)
- [PMC: Comparison of Three Commercial Knowledge Bases](https://pmc.ncbi.nlm.nih.gov/articles/PMC6080681/)
- [Wolters Kluwer: Medi-Span Drug Pricing](https://www.wolterskluwer.com/en/solutions/medi-span/medi-span/drug-pricing-data)
- [Wolters Kluwer: Embedded Drug Data](https://www.wolterskluwer.com/en/solutions/medi-span/medi-span/content-sets)
