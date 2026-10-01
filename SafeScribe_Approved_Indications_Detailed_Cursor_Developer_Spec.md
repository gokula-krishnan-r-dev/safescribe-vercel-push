# SafeScribe Clinical Admin — Approved Indications
## Detailed Cursor-Ready Developer Implementation Specification

**Project:** SafeScribe  
**Area:** Clinical Admin → Terminology → Indication Mappings  
**Feature:** Approved Indications Repository UI + backend integration  
**Purpose:** Create a governed, scalable medication-to-indication relationship manager using SafeScribe's existing CCDD and SNOMED infrastructure.

---

# 0. IMPLEMENTATION INTENT

This feature is **not** a manually maintained list of drug aliases and conditions.

It is a governance UI over this relationship:

```text
CCDD medication concept
        ↕
SNOMED CT indication concept
```

The repository must:

- reuse the existing CCDD medication catalogue;
- reuse the existing SNOMED clinical concepts;
- avoid alias-level duplicate mappings;
- store mappings at the correct medication hierarchy level;
- preserve evidence/source provenance;
- support jurisdiction;
- support review and publication;
- preserve history;
- feed one shared `MedicationIndicationResolver`;
- scale through candidate review rather than attempting full manual mapping of every drug.

---

# 1. NON-NEGOTIABLE ARCHITECTURE

## 1.1 Medication identity

Medication identity must come from existing CCDD infrastructure.

Do not persist arbitrary medication strings such as:

```text
amox
amoxl
novamoxin
amoxicillin
```

as independent mapping keys.

Brand/product searches may resolve those strings, but the saved mapping must point to a canonical CCDD concept.

---

## 1.2 Indication identity

Indication identity must come from existing SNOMED CT infrastructure.

Do not maintain a second Conditions master specifically for this feature.

Persist:

```text
indication_concept_id
```

from SNOMED.

---

## 1.3 Repository responsibility

The repository stores:

```text
Medication concept
+
Indication concept
+
Mapping level
+
Relationship type
+
Jurisdiction
+
Source
+
Governance/version metadata
```

The repository does **not** decide the patient-specific indication.

The pharmacist confirms the actual patient indication in the clinical workflow.

---

# 2. CLINICAL ADMIN NAVIGATION

Use the existing Clinical Admin navigation shell.

Target structure:

```text
Clinical Admin
├── Dashboard
├── Pathways
├── Safety Engine
├── Terminology
│   ├── Medication Catalogue
│   ├── Clinical Concepts
│   └── Indication Mappings   ← ACTIVE
├── References
├── Audit
└── Settings
```

Sidebar label:

```text
Indication Mappings
```

Page title:

```text
Approved Indications
```

---

# 3. PAGE HEADER

Render:

```text
Approved Indications

Manage governed medication–indication relationships used by Adapt, Renew and Prescribe.
Mappings connect CCDD medication concepts with SNOMED CT indications and preserve source,
jurisdiction, version and audit information.
```

Top-right CTA:

```text
+ Add mapping
```

Button style:
- teal primary;
- 38–40px height;
- compact;
- 8–10px radius;
- no oversized shadow.

---

# 4. PAGE TABS

Use four tabs:

```text
Approved
Review Queue
Coverage
Version History
```

Recommended badge:

```text
Review Queue  [128]
```

Badge contains pending candidate count.

Default tab:

```text
Approved
```

URL/query-state support recommended:

```text
?tab=approved
?tab=review
?tab=coverage
?tab=versions
```

Tab state should survive refresh when possible.

---

# 5. APPROVED TAB — TOOLBAR

Layout:

```text
[ Search medication or indication... ]

[ Jurisdiction ▼ ]
[ Relationship ▼ ]
[ Level ▼ ]
[ Status ▼ ]

Clear filters
```

Search placeholder:

```text
Search medication or indication...
```

Optional richer placeholder:

```text
Search medication (e.g. amoxicillin) or indication (e.g. otitis media)...
```

Search across:

### Medication
- CCDD preferred label;
- generic name;
- brand aliases available through CCDD;
- ingredient label;
- clinical drug label.

### Indication
- SNOMED preferred term;
- SNOMED synonyms/descriptions.

Debounce:

```text
250–350 ms
```

Use server-side filtering/pagination.

---

# 6. APPROVED TAB — FILTERS

## Jurisdiction

Options:

```text
All
Canada
Alberta
British Columbia
Ontario
```

Use existing jurisdiction config if available.

Backend codes:

```text
CA
AB
BC
ON
```

---

## Relationship

Options:

```text
All
Approved indication
Guideline-supported
Off-label
Other
```

---

## Level

Options:

```text
All
Therapeutic moiety
Ingredient
Ingredient combination
Clinical drug
Product
```

---

## Status

Options:

```text
Approved
Retired
All
```

Default:

```text
Approved
```

Do not use `Active` as the primary governance label in the new UI.

If backend currently stores `active=true`, map it to UI status `Approved` until schema migration is completed.

---

# 7. APPROVED TAB — TABLE

Columns:

```text
MEDICATION
INDICATION
LEVEL
RELATIONSHIP
SOURCE
JURISDICTION
STATUS
ACTIONS
```

Recommended widths:

```text
Medication       18%
Indication       22%
Level            11%
Relationship     15%
Source           14%
Jurisdiction     9%
Status           7%
Actions          4%
```

Keep table dense but readable.

---

# 8. MEDICATION CELL

Display:

```text
Amoxicillin
Ingredient · CCDD: 1234567
```

or:

```text
Amoxicillin 500 mg capsule
Clinical drug · CCDD: 1234567
```

Primary line:
- 14px;
- medium/semi-bold.

Secondary line:
- 11.5–12px;
- muted gray.

Never show the old free-text mapping key as the primary identity.

---

# 9. INDICATION CELL

Display:

```text
Acute otitis media
SNOMED CT: 233604007
```

Use SNOMED preferred term for primary display.

Secondary:
- code;
- muted.

---

# 10. LEVEL COLUMN

Values:

```text
Therapeutic moiety
Ingredient
Ingredient combination
Clinical drug
Product
```

Use simple text, not large badges.

---

# 11. RELATIONSHIP COLUMN

Display compact semantic badge.

Examples:

```text
Approved indication
Guideline-supported
Off-label
Other
```

Suggested visual treatment:

- `Approved indication` → soft green;
- `Guideline-supported` → soft blue;
- `Off-label` → soft amber;
- `Other` → neutral gray.

Do not call all relationships “approved indication.”

---

# 12. SOURCE COLUMN

Display short source name.

Examples:

```text
Health Canada PM
CPS Guideline
ACP Guideline
eCPS
```

If `source_reference_id` exists:
- clicking source opens existing Reference viewer/details;
- external/open icon optional.

If no source:
- display `Not linked`;
- use muted text;
- allow admin to fix.

For newly created mappings:
- require a source for `approved_indication`, `guideline_supported`, and `off_label`;
- `other` may allow no source only if existing governance rules permit.

---

# 13. JURISDICTION COLUMN

Display:

```text
Canada
Alberta
British Columbia
Ontario
```

Do not show internal code unless in details.

---

# 14. STATUS COLUMN

Use:

```text
Approved
Retired
```

Suggested:
- Approved → green pill;
- Retired → gray pill.

Do not use red for retired.

---

# 15. ACTIONS COLUMN

Use:

```text
[Edit icon] [•••]
```

Edit icon opens edit drawer/modal.

Overflow:

```text
View details
View audit history
Retire mapping
```

For retired mapping:

```text
View details
View audit history
Create new version
```

Do not show permanent delete for production mappings.

---

# 16. TABLE SORTING

Allow sort where useful:

```text
Medication
Indication
Relationship
Source
Jurisdiction
Last updated
```

Default:

```text
Medication ASC
Indication ASC
```

or preserve existing admin sorting convention.

---

# 17. PAGINATION

Use server-side pagination.

Footer:

```text
Showing 1–20 of 542 mappings

<  1  2  3  …  28  >
```

Recommended page size:

```text
20
```

Optional:

```text
20 / 50 / 100
```

Do not render all mappings at once.

---

# 18. ADD MAPPING — UI

Click:

```text
+ Add mapping
```

Open modal or right-side drawer.

Preferred desktop implementation:
- large modal or 520–620px side drawer;
- follow existing Clinical Admin pattern.

Title:

```text
Add indication mapping
```

Helper:

```text
Create a governed relationship between an existing CCDD medication concept and an existing SNOMED CT indication.
```

---

# 19. ADD MAPPING — FIELDS

Order exactly:

```text
Medication (CCDD) *
Mapping level *
Indication (SNOMED CT) *
Relationship type *
Jurisdiction *
Source reference *
Notes (optional)
```

Footer:

```text
Cancel
Save mapping
```

---

# 20. MEDICATION (CCDD) SELECTOR

Control:

```text
[ Search medication (generic, brand or DIN)... ]
```

Use existing CCDD search endpoint/component.

Search may return:
- ingredient;
- therapeutic moiety;
- clinical drug;
- product;
- brand/product aliases.

Search row example:

```text
Amoxicillin
Ingredient
CCDD: 1234567
```

or:

```text
Amoxil 500 mg capsule
Product
CCDD: 7654321
Resolves to ingredient: Amoxicillin
```

After selection:
- persist canonical CCDD ID;
- show selected display label;
- show concept type;
- show normalized parent ingredient if useful.

Never persist the search string as mapping identity.

---

# 21. MAPPING LEVEL

Control:

```text
[ Ingredient ▼ ]
```

Options:

```text
Therapeutic moiety
Ingredient
Ingredient combination
Clinical drug
Product
```

Default selection should be derived from selected CCDD concept when possible.

Helper icon/text:

```text
Use the highest reusable level that accurately represents the indication.
```

If user chooses a more specific level than needed, allow but show contextual caution only if the backend can determine it.

---

# 22. INDICATION (SNOMED CT) SELECTOR

Control:

```text
[ Search SNOMED CT (e.g. otitis media)... ]
```

Use existing SNOMED search.

Result:

```text
Acute otitis media
SNOMED CT: 233604007
```

Selected state should show:
- preferred term;
- concept ID.

Do not allow arbitrary text as final indication identity.

---

# 23. RELATIONSHIP TYPE

Control:

```text
[ Approved indication ▼ ]
```

Options:

```text
Approved indication
Guideline-supported
Off-label
Other
```

Helper text:

```text
Classify how this medication–indication relationship is supported.
```

---

# 24. JURISDICTION

Control:

```text
[ Canada ▼ ]
```

Options from existing jurisdiction configuration.

Default:
- Canada if source is Health Canada-level;
- otherwise existing admin default.

Do not infer province from current admin user's location.

---

# 25. SOURCE REFERENCE

Control:

```text
[ Search approved references... ]
```

Use existing SafeScribe Reference Repository.

Display results:

```text
Health Canada Product Monograph — Amoxicillin
Updated: [date]
```

Persist:

```text
source_reference_id
```

For governed relationships:
- source must already exist in Reference Repository;
- do not paste arbitrary URLs as the production source field.

Optional secondary action:

```text
+ Add reference
```

should route to existing References workflow, not create unmanaged source text inline.

---

# 26. NOTES

Optional textarea:

```text
Add notes about this mapping...
```

Max length recommendation:

```text
1000 characters
```

Notes are internal Clinical Admin notes only.

Do not display in pharmacist-facing workflow.

---

# 27. SAVE MAPPING — VALIDATION

Before save validate:

- medication CCDD concept exists;
- selected mapping level valid;
- indication SNOMED concept exists;
- relationship valid;
- jurisdiction valid;
- source reference valid when required;
- duplicate active mapping does not already exist.

Duplicate definition:

```text
same medication concept
+
same mapping level
+
same indication concept
+
same relationship type
+
same jurisdiction
+
non-retired
```

Duplicate error:

```text
This mapping already exists.
View existing mapping
```

---

# 28. SAVE MAPPING — SUCCESS

On save:

1. create mapping;
2. mark status `approved` if current user has publisher permission;
3. otherwise create draft/pending according to existing RBAC;
4. create audit event;
5. invalidate resolver cache;
6. refresh table;
7. show toast:

```text
Indication mapping saved.
```

---

# 29. EDIT MAPPING

Click edit icon.

Open:

```text
Edit indication mapping
```

Display medication and indication clearly.

Fields:

```text
Medication (CCDD)
Mapping level
Indication (SNOMED CT)
Relationship type
Jurisdiction
Source reference
Status
Notes
```

Recommended identity rule:

- if medication or indication identity is changed, create a new version/new mapping and retire previous record;
- do not silently overwrite historical meaning.

Simple metadata updates such as:
- note;
- source replacement;
- jurisdiction correction;
may still create mapping version according to existing audit/versioning conventions.

---

# 30. EDIT FOOTER

Use:

```text
Retire mapping        Cancel   Save changes
```

Retire is visually separated and subtle-danger style.

Do not use trash icon.

---

# 31. RETIRE MAPPING

Confirmation:

```text
Retire indication mapping?

This mapping will no longer be returned for new consultations.
Historical consultations will retain the version they used.

[Cancel] [Retire mapping]
```

On confirm:
- `status = retired`;
- `valid_to = now`;
- set retired metadata;
- write audit event;
- invalidate resolver cache.

---

# 32. MAPPING DETAILS SIDE PANEL

`View details` opens side panel.

Display:

```text
Mapping details

Medication
Amoxicillin
Ingredient · CCDD: 1234567

Indication
Acute otitis media
SNOMED CT: 233604007

Mapping level
Ingredient

Relationship
Approved indication

Jurisdiction
Canada

Source
Health Canada Product Monograph
[View source]

Status
Approved

Version
3

Created
12-Sep-2026 by [admin]

Last updated
29-Sep-2026 by [admin]

[View audit history]
```

No editing directly inside the details panel.

---

# 33. REVIEW QUEUE TAB

Purpose:

Show medication–indication relationships that are **candidates**, not production truth.

Sources:

```text
pharmacist_selection
automated_extraction
import
admin_added
```

---

# 34. REVIEW QUEUE TOOLBAR

Controls:

```text
[ Search medication or indication... ]
[ Source ▼ ]
[ Jurisdiction ▼ ]
[ Status ▼ ]
```

Default:

```text
status = Pending review
```

---

# 35. REVIEW QUEUE TABLE

Columns:

```text
MEDICATION
SUGGESTED INDICATION
SOURCE
OBSERVATIONS
FIRST SEEN
LAST SEEN
STATUS
ACTION
```

Example:

```text
Amoxicillin
Ingredient · CCDD [ID]

Acute otitis media
SNOMED CT [ID]

Pharmacist selections

27

02-Sep-2026

29-Sep-2026

Pending review

[Review]
```

Default sort:

```text
Observations DESC
Last seen DESC
```

---

# 36. REVIEW CANDIDATE DRAWER

Click `Review`.

Render:

```text
Review indication mapping

Medication
Amoxicillin
Ingredient · CCDD [ID]

Suggested indication
Acute otitis media
SNOMED CT [ID]

Observed in SafeScribe
27 consultations

Source
Pharmacist selections

First seen
02-Sep-2026

Last seen
29-Sep-2026

Supporting source
[ Search approved references... ]

Relationship
[ Approved indication ▼ ]

Jurisdiction
[ Canada ▼ ]

Admin notes
[ optional ]

[Reject] [Approve mapping]
```

---

# 37. CANDIDATE APPROVAL

On approve:

1. revalidate CCDD concept;
2. revalidate SNOMED concept;
3. check duplicate approved mapping;
4. require source where configured;
5. create production mapping;
6. mark candidate `accepted`;
7. set `promoted_mapping_id`;
8. write audit;
9. invalidate resolver cache;
10. refresh Review Queue count.

Use one transaction.

---

# 38. CANDIDATE REJECTION

Confirmation not always required; use if existing admin pattern.

On reject:

```text
status = rejected
reviewed_by
reviewed_at
review_notes
```

Do not delete candidate.

Do not let the exact same rejected candidate create a brand-new duplicate on next observation.

Instead:
- increment observation count;
- preserve rejected state;
- optionally flag as `new evidence observed`.

Publisher may reopen if needed.

---

# 39. BULK REVIEW

Support selecting several candidate rows.

Actions:

```text
Approve selected
Reject selected
```

For approval:
- validate each candidate;
- show confirmation summary;
- return partial failures explicitly;
- do not silently ignore conflicts.

---

# 40. MEDICATION-CENTRIC REVIEW

Add action:

```text
Review mappings
```

for one medication.

UI:

```text
Amoxicillin

Pending indication candidates

☑ Acute otitis media
☑ Acute bacterial sinusitis
☑ Pharyngitis / tonsillitis
☐ Bacterial infection, unspecified

Source / observations visible per row

[Approve selected]
```

This is a primary efficiency feature.

---

# 41. COVERAGE TAB

Purpose:

Prioritize mapping work based on real SafeScribe usage.

Top metrics:

```text
Medications encountered
With approved mappings
Needs review
No approved mappings
Pending relationships
Approved relationships
```

Avoid giant dashboard cards.

Use compact KPI cards.

---

# 42. COVERAGE CALCULATION

Primary denominator:

```text
unique medication concepts actually encountered in SafeScribe
```

Do not use entire CCDD catalogue as the main coverage denominator.

Optional secondary metric:

```text
Catalogue-wide coverage
```

can be shown in muted text.

---

# 43. COVERAGE PRIORITY TABLE

Title:

```text
Most-used medications without approved mappings
```

Columns:

```text
MEDICATION
ENCOUNTERS
PENDING CANDIDATES
LAST SEEN
ACTION
```

Example:

```text
Amoxicillin
Ingredient · CCDD [ID]

148

4

29-Sep-2026

[Review mappings]
```

Sort by:

```text
encounters DESC
```

---

# 44. VERSION HISTORY TAB

Display repository releases and clinically material mapping changes.

Table:

```text
VERSION
PUBLISHED
PUBLISHED BY
CHANGES
STATUS
ACTION
```

Example:

```text
2026.09.3
29-Sep-2026
JD
14 mappings added · 2 retired
Published
[View]
```

---

# 45. VERSION DETAIL

Click View:

```text
Repository version 2026.09.3

Published
29-Sep-2026

Published by
[admin]

Changes
+ 14 mappings
~ 3 mappings updated
- 2 mappings retired

[View change list]
```

Consultations should be able to store:

```text
medication_indication_repository_version_used
```

---

# 46. CANDIDATE CREATION FROM CLINICAL WORKFLOW

When pharmacist selects a SNOMED indication manually because no approved mapping exists:

```text
CCDD medication
+
SNOMED indication
↓
candidate upsert
```

Candidate source:

```text
pharmacist_selection
```

Do this asynchronously.

Do not interrupt pharmacist workflow.

---

# 47. CANDIDATE UPSERT KEY

Use:

```text
medication_concept_id
+
medication_mapping_level
+
indication_concept_id
+
jurisdiction
```

If candidate already exists:

```text
usage_count += 1
last_seen_at = now()
```

Do not create duplicate candidates.

---

# 48. CURRENT ALIAS DATA MIGRATION

Existing rows such as:

```text
amox
amoxl
novamoxin
amoxicillin
```

must be normalized.

Build migration tooling.

For each row:

1. resolve medication alias using CCDD;
2. determine canonical concept;
3. resolve old condition to SNOMED;
4. group same normalized pair;
5. preserve source metadata;
6. migrate deterministic rows;
7. send ambiguous rows to Review Queue;
8. retain migration audit.

Example:

```text
amox → Acute otitis media
novamoxin → Acute otitis media
amoxicillin → Acute otitis media
```

becomes:

```text
Amoxicillin [CCDD ingredient]
↔
Acute otitis media [SNOMED]
```

one mapping.

---

# 49. COMBINATION PRODUCT RULE

Do not union component indications automatically.

Correct:

```text
Amoxicillin + clavulanate
→ ingredient_combination mapping
```

Do not infer:

```text
amoxicillin indications
+
clavulanate indications
```

as combination indications.

---

# 50. PRODUCT/FORMULATION-SPECIFIC RULE

When clinical indication differs by:
- route;
- formulation;
- clinical drug;
- product;

use more-specific mapping level:

```text
clinical_drug
product
```

Do not create free-text alias keys.

---

# 51. RUNTIME RESOLVER INTEGRATION

Admin data must feed:

```text
MedicationIndicationResolver
```

Resolver order:

```text
Product
↓
Clinical drug
↓
Ingredient combination
↓
Ingredient
↓
Therapeutic moiety
```

Apply:
- jurisdiction;
- status;
- relationship-type filter;
- deduplication;
- specific-over-general precedence.

---

# 52. AI ROLE

AI is **not** the repository source of truth.

AI may:
- rank approved indication candidates in live workflow;
- extract candidate indications from approved source text;
- rank SNOMED matches for admin review;
- summarize why a candidate was proposed.

AI must not:
- create final mappings from model memory;
- invent SNOMED concepts;
- publish mappings;
- retire mappings;
- label a relationship `approved_indication` without source evidence.

AI-generated relationships enter Review Queue only.

---

# 53. API ENDPOINTS

Suggested:

```text
GET  /api/clinical-admin/indication-mappings
GET  /api/clinical-admin/indication-mappings/:id
POST /api/clinical-admin/indication-mappings
PUT  /api/clinical-admin/indication-mappings/:id
POST /api/clinical-admin/indication-mappings/:id/retire

GET  /api/clinical-admin/indication-candidates
GET  /api/clinical-admin/indication-candidates/:id
POST /api/clinical-admin/indication-candidates/:id/approve
POST /api/clinical-admin/indication-candidates/:id/reject
POST /api/clinical-admin/indication-candidates/bulk-approve
POST /api/clinical-admin/indication-candidates/bulk-reject

GET /api/clinical-admin/indication-coverage
GET /api/clinical-admin/indication-versions
GET /api/clinical-admin/indication-versions/:id

POST /api/clinical/medication-indications/resolve
```

Adapt to current route conventions.

---

# 54. PRODUCTION MAPPING MODEL

```ts
interface MedicationIndicationMapping {
  id: string;

  medicationConceptId: string;

  medicationMappingLevel:
    | "therapeutic_moiety"
    | "ingredient"
    | "ingredient_combination"
    | "clinical_drug"
    | "product";

  indicationConceptId: string;

  relationshipType:
    | "approved_indication"
    | "guideline_supported"
    | "off_label"
    | "other";

  jurisdiction: string;

  sourceReferenceId?: string | null;

  status:
    | "approved"
    | "retired";

  mappingVersion: number;

  validFrom: string;
  validTo?: string | null;

  notes?: string | null;

  createdBy: string;
  createdAt: string;

  approvedBy?: string;
  approvedAt?: string;

  updatedBy?: string;
  updatedAt?: string;

  retiredBy?: string;
  retiredAt?: string;
}
```

---

# 55. CANDIDATE MODEL

```ts
interface MedicationIndicationCandidate {
  id: string;

  medicationConceptId: string;
  medicationMappingLevel: string;

  indicationConceptId: string;

  sourceType:
    | "pharmacist_selection"
    | "automated_extraction"
    | "import"
    | "admin_added";

  sourceReferenceId?: string | null;

  jurisdiction: string;

  relationshipTypeSuggested?: string | null;

  usageCount: number;

  firstSeenAt: string;
  lastSeenAt: string;

  status:
    | "pending"
    | "accepted"
    | "rejected"
    | "merged";

  reviewedBy?: string | null;
  reviewedAt?: string | null;

  reviewNotes?: string | null;

  promotedMappingId?: string | null;
}
```

---

# 56. RBAC

Reuse existing Clinical Admin permissions.

Recommended:

### Viewer
- view Approved;
- view Review Queue;
- view Coverage;
- view Version History.

### Editor
- add candidate/manual draft;
- edit notes;
- reject candidate.

### Publisher
- save approved mapping;
- approve candidate;
- bulk approve;
- retire mapping;
- publish repository version.

Frontend must hide/disable unauthorized actions.

Backend must enforce independently.

---

# 57. AUDIT EVENTS

Create events for:

```text
indication_mapping_created
indication_mapping_updated
indication_mapping_retired

indication_candidate_created
indication_candidate_observed
indication_candidate_approved
indication_candidate_rejected
indication_candidate_merged

indication_bulk_approved

indication_repository_version_published

indication_migration_normalized
```

Audit payload includes:
- actor;
- timestamp;
- before;
- after;
- source candidate ID;
- version;
- optional admin note.

Do not include patient-identifying data.

---

# 58. CACHE

Recommended resolver cache key:

```text
med-ind:{jurisdiction}:{medicationConceptId}:{repositoryVersion}
```

Invalidate when:
- approved mapping added;
- mapping edited;
- mapping retired;
- new repository version published.

---

# 59. PERFORMANCE

Targets:

```text
Approved list p95 < 500 ms
Resolver p95 < 150 ms excluding AI
Terminology search p95 according to existing CCDD/SNOMED service target
```

Use:
- indexes;
- pagination;
- server filtering;
- debounced terminology lookup;
- cache.

Do not send full terminology catalogues to browser.

---

# 60. ACCESSIBILITY

Requirements:
- keyboard-accessible tabs;
- keyboard-accessible tables/actions;
- visible focus states;
- dialog focus trap;
- `aria-label` for icon buttons;
- labels connected to form fields;
- error text connected via `aria-describedby`;
- status not conveyed by color alone.

---

# 61. RESPONSIVE

Clinical Admin is desktop-first.

Desktop:
- full table.

Tablet:
- collapse Source/Jurisdiction into row detail if needed.

Mobile:
- render rows as cards;
- preserve medication, indication, relationship, status, actions.

Do not allow unusable horizontal overflow.

---

# 62. EMPTY STATES

## Approved

```text
No approved indication mappings found.
Add a mapping or review pending candidates.
```

## Review Queue

```text
No mappings are waiting for review.
```

## Coverage

```text
No medication encounter data is available yet.
```

## Version History

```text
No published repository versions are available yet.
```

---

# 63. ERROR STATES

Terminology search unavailable:

```text
Medication search is temporarily unavailable.
Try again.
```

or:

```text
Indication search is temporarily unavailable.
Try again.
```

Save failure:

```text
Mapping could not be saved.
No changes were applied.
```

Candidate approval failure:

```text
This candidate could not be approved.
Review the highlighted issue and try again.
```

Never partially publish without telling the admin.

---

# 64. TESTING — UNIT

Test:
- duplicate prevention;
- CCDD validation;
- SNOMED validation;
- relationship validation;
- jurisdiction validation;
- source requirement;
- retirement;
- candidate upsert;
- candidate merge;
- versioning;
- cache invalidation;
- combination handling;
- mapping-level precedence.

---

# 65. TESTING — UI

Verify:
- tab navigation;
- search;
- filters;
- pagination;
- add modal;
- edit;
- details panel;
- retire;
- Review Queue;
- approve/reject;
- bulk review;
- Coverage;
- Version History;
- RBAC;
- keyboard accessibility.

---

# 66. TESTING — MIGRATION

Test examples:

```text
amox
amoxl
novamoxin
amoxicillin
```

Expected:
- aliases resolve;
- deterministic duplicates collapse;
- ambiguous rows go to Review Queue;
- no history lost.

---

# 67. IMPLEMENTATION ORDER

Follow this sequence:

```text
1. Freeze new alias-key mappings.
2. Confirm existing CCDD APIs/components.
3. Confirm existing SNOMED APIs/components.
4. Confirm Reference Repository selector.
5. Update mapping schema.
6. Build Approved tab with revised columns.
7. Rebuild Add Mapping modal.
8. Rebuild Edit Mapping modal.
9. Add Details side panel.
10. Replace delete with Retire.
11. Build Review Queue.
12. Build candidate approval/rejection.
13. Build bulk medication review.
14. Build Coverage.
15. Build Version History.
16. Build alias normalization migration.
17. Normalize existing data.
18. Connect to MedicationIndicationResolver.
19. Add RBAC/audit.
20. Add caching.
21. Add tests.
```

---

# 68. FINAL REQUIRED UI

```text
Approved Indications

[Approved] [Review Queue 128] [Coverage] [Version History]       [+ Add mapping]

[ Search medication or indication... ]
[ Jurisdiction ▼ ] [ Relationship ▼ ] [ Level ▼ ] [ Status ▼ ]  Clear filters

MEDICATION    INDICATION    LEVEL    RELATIONSHIP    SOURCE    JURISDICTION    STATUS    ACTIONS
Amoxicillin   Otitis media  Ingredient Approved indication HC PM Canada         Approved  ✎ •••
...
```

---

# 69. FINAL REQUIRED ADD/EDIT FORM

```text
Add indication mapping

Medication (CCDD) *
[ Search medication... ]

Mapping level *
[ Ingredient ▼ ]

Indication (SNOMED CT) *
[ Search SNOMED CT... ]

Relationship type *
[ Approved indication ▼ ]

Jurisdiction *
[ Canada ▼ ]

Source reference *
[ Select approved reference... ]

Notes
[ optional ]

[Cancel] [Save mapping]
```

---

# 70. FINAL PRODUCT PRINCIPLE

```text
CCDD
defines what the medication is

SNOMED
defines what the indication is

Approved Indications Repository
defines the governed relationship

Clinical Admin
reviews and publishes that relationship

AI
may help rank or propose candidates

Pharmacist
confirms the actual patient-specific indication
```

Do not revert to alias-based manual mapping.
