# SafeScribe Adapt — Step 1 UI
## Cursor-Ready Developer Instructions
### Prescription + Indication + Reason for Adaptation

**Module:** SafeScribe Adapt  
**Step:** 1 of 4 — Prescription and reason  
**Target:** Recreate the supplied UI faithfully, including clickable behavior and indication-selection states.  
**Scope:** Frontend/UI plus integration hooks to existing CCDD, SNOMED, Approved Indications Repository, patient context, and optional AI ranking.

---

# 1. Page Goal

Step 1 must let the pharmacist:

1. Review the original prescription.
2. Confirm the medication indication.
3. Select the reason for adaptation.
4. Continue to Patient Assessment.

Do not add a new workflow step for indication.

---

# 2. Page Structure

```text
Global app header
Step progress indicator

ADAPT PRESCRIPTION
Prescription and reason for adaptation
Add the original prescription, confirm what it is being used for, then select the reason for adaptation.

Main grid
├── Primary column
│   ├── Original prescription card
│   │   ├── Prescription summary
│   │   └── Indication for this medication
│   └── Reason for adaptation
└── Right sidebar
    ├── Patient context
    ├── About indication selection
    └── Helpful resources

Bottom navigation
├── Back to Dashboard
└── Save & continue to Patient Assessment
```

Desktop: primary content ~78%, sidebar ~22%, gap ~24px.

---

# 3. Visual Language

Use existing SafeScribe tokens/components wherever possible.

Typography:
- Geist Sans preferred; Inter/system fallback.
- Page title: ~30px / 650.
- Section title: 18px / 600.
- Card title: 15–16px / 600.
- Body: 14px.
- Metadata: 12–13px.

Style:
- white cards;
- light gray page background;
- 1px subtle borders;
- 12–16px radius;
- minimal shadows;
- teal for active/primary;
- soft mint for selected/supportive states;
- red only for destructive actions;
- amber only for caution/review.

Do not add heavy gradients or oversized cards.

---

# 4. Stepper

Show:

```text
1 Prescription and reason
2 Patient assessment
3 Proposed adaptation & check
4 Documentation and complete
```

Step 1 active:
- teal filled circle;
- dark active label.

Other steps:
- white outlined circle;
- muted text.

Do not allow forward-step clicking unless current SafeScribe validation already supports it.

---

# 5. Original Prescription Section

Outer card header:

```text
[1] [document icon]

Original prescription
Enter the medication details from the current prescription.

Tips     [chevron]
```

Prescription summary panel:

```text
Rosuvastatin 10 mg tablet
Oral tablet

Directions (SIG)
Take 1 tablet once daily

Prescriber
Not recorded

Date written
Not recorded

Status
Not yet dispensed
```

Top-right actions:

```text
Edit
Remove
```

## Edit behavior

Reuse existing prescription editor.

If only non-medication metadata changes, preserve indication.

If medication identity, ingredient, strength/formulation in a clinically meaningful way changes:
- mark indication stale;
- clear AI suggestions;
- rerun MedicationIndicationResolver;
- require indication reconfirmation.

## Remove behavior

Confirm:

```text
Remove original prescription?

This will also clear the selected indication and adaptation reason linked to this prescription.

[Cancel] [Remove prescription]
```

On confirm:
- remove prescription;
- clear indication;
- clear adaptation type/reason;
- disable final CTA.

---

# 6. Indication Panel Placement

Place directly below the prescription summary inside the same outer card.

Header:

```text
[clinical document icon]

Indication for this medication *

Select the primary indication for which this medication is being used.
This helps provide more relevant guidance and safety checks.

[View patient conditions] [Tips] [chevron]
```

Indication is required unless pharmacist explicitly selects:

```text
Indication not known / not available
```

---

# 7. Indication Tabs

Show:

```text
Search and select
Common indications
Patient's conditions
All indications
```

Default active:

```text
Search and select
```

Active tab:
- teal text;
- teal underline.

---

# 8. Search and Select Layout

Desktop:

```text
LEFT
Search and mapped results

RIGHT
Suggested based on patient information
```

Suggested ratio:

```css
grid-template-columns: 1.15fr 0.85fr;
gap: 20px;
```

Stack vertically on smaller widths.

---

# 9. Search Field

Placeholder:

```text
Search indication...
```

Behavior:
- debounce ~300ms;
- search approved mapped indications first;
- allow broader SNOMED search as pharmacist types;
- clear icon resets query;
- selection remains intact while query changes until pharmacist selects another result.

Do not persist arbitrary free text as a coded indication.

---

# 10. Search Results

Each row:

```text
[icon]
Hyperlipidemia (dyslipidemia)
SNOMED CT: [concept ID]

[optional badge]
```

Allowed badges:

```text
Mapped indication
Patient condition
Common for this medication
```

Do not display:
- AI diagnosis
- Best diagnosis
- Most likely diagnosis

Clicking result:
- highlights selected row;
- sets candidate;
- synchronizes matching item in suggestion pane;
- selection remains pharmacist-controlled.

---

# 11. Common Indications Tab

Source only from:

```text
Approved Indications Repository
```

using:

```text
MedicationIndicationResolver
```

Do not hardcode drug-specific indication arrays.

Empty state:

```text
No approved indications are currently mapped for this medication.
Search all indications instead.
```

Action:

```text
Search all indications
```

switches to `All indications`.

---

# 12. Patient's Conditions Tab

Use structured patient conditions.

Preferred matching:

```text
exact SNOMED match
approved normalized/equivalent concept mapping
```

If patient condition also maps to medication, show:

```text
Mapped indication
```

Do not let AI create patient conditions.

---

# 13. All Indications Tab

Use existing SNOMED search infrastructure.

Result row:
- SNOMED preferred term;
- SNOMED concept ID.

If pharmacist selects a SNOMED indication not currently approved for this medication:
- allow consultation-level selection;
- set selection source `manual_search`;
- asynchronously create a `medication_indication_candidate`;
- do not auto-publish global mapping.

---

# 14. Suggested Based on Patient Information

Heading:

```text
Suggested based on patient information
```

Maximum 3 suggestions.

Example:

```text
○ Hyperlipidemia (dyslipidemia)
  Matches an existing patient condition

○ Cardiovascular risk reduction
  Relevant cardiovascular risk factors are recorded
```

Preferred badge:

```text
Suggested
```

or:

```text
Matches patient information
```

Avoid `Most likely`.

---

# 15. Suggestion Logic

Use this order:

```text
Approved medication-indication mappings
        ↓
Deterministic patient-condition intersection
        ↓
Structured patient context
        ↓
Optional AI ranking only when still ambiguous
```

AI receives only approved candidate indication IDs.

AI must not invent indication candidates.

---

# 16. Suggestion Selection

On click:
- select candidate;
- synchronize left result highlight;
- store provenance (`patient_condition` or `ai_suggested`);
- do not mark globally confirmed until pharmacist saves/completes Step 1.

---

# 17. Enter Another Indication

Show:

```text
+ Enter another indication
```

Click:
- activate `All indications`;
- focus SNOMED search.

Do not open free-text-only input by default.

---

# 18. Unknown Indication

Show:

```text
☐ Indication not known / not available
```

When selected:
- clear selected indication candidate;
- set `status = unknown`;
- disable suggestion selection while checked;
- allow Step 1 completion.

Helper:

```text
SafeScribe will avoid indication-specific guidance where the indication cannot be confirmed.
```

No repository candidate is created for Unknown.

---

# 19. Confirmed State

Once selected and saved:

```text
Indication for this medication

✓ Hyperlipidemia (dyslipidemia)

Selected by pharmacist

[Change]
```

Optional muted provenance:

```text
Suggested from patient information
```

Never show:

```text
AI selected
```

---

# 20. View Patient Conditions

Click `View patient conditions`:
- open compact modal/drawer;
- show current structured conditions;
- no duplicate full assessment screen;
- if editing is allowed by existing product pattern, use current shared patient-condition editor.

---

# 21. Reason for Adaptation Section

Header:

```text
[2] [document icon]

Reason for adaptation
Select what needs to be changed and why the adaptation is being considered.

Tips
```

Render six cards:

```text
Dose
Dosage form / formulation
Regimen / frequency
Route
Therapeutic substitution
Other
```

Helpers:

```text
Dose
Adjust the dose strength or amount

Dosage form / formulation
Change the formulation or strength

Regimen / frequency
Change how often it is taken

Route
Change the route of administration

Therapeutic substitution
Replace with a therapeutically equivalent medication

Other
Other reason for adaptation
```

Only one card selected at a time.

Selected state:
- teal border;
- soft teal background;
- selected icon/state.

---

# 22. Reason Detail Controls

After selecting an adaptation type, show type-specific reason controls beneath cards.

Example for Dose:

```text
Why is a dose change being considered?

○ Weight-/age-based adjustment
○ Renal function
○ Hepatic function
○ Dose not appropriate for indication/guideline
○ Treatment response / optimization
○ Adverse effect / tolerability
○ Drug interaction
○ Other patient factor
○ Other
```

Reuse current Adapt Step 1 enums and backend field names.

---

# 23. Right Sidebar — Patient Context

Card:

```text
Patient context      [Edit]

Age / Sex
62 years · Female

Relevant conditions
Hypertension, Type 2 diabetes, Dyslipidemia

Allergies
Penicillin (urticaria)

Recent labs
LDL ...
eGFR ...
A1C ...

View full assessment →
```

Use structured data only.

If unavailable:

```text
Not yet recorded
```

No AI-generated patient facts.

---

# 24. About Indication Selection

Card:

```text
About indication selection

The indication helps SafeScribe provide more relevant dose options,
therapeutic alternatives, safety checks and clinical guidance.

Learn more about indications →
```

Open help drawer/modal.

---

# 25. Helpful Resources

Card:

```text
Helpful resources

Common indications by medication class →
Indication selection guide →
How this information is used →
```

These are informational and must not block workflow.

---

# 26. Bottom Navigation

Left:

```text
← Back to Dashboard
```

Right:

```text
Save & continue to Patient Assessment →
```

Use current sticky/light footer pattern if available.

---

# 27. CTA Enablement

Enable only when:

```text
original prescription exists
AND
(indication confirmed OR indication explicitly unknown)
AND
adaptation type selected
AND
required reason detail complete
```

Otherwise disabled with muted styling.

---

# 28. Save Behavior

On CTA:

1. validate prescription;
2. validate indication or explicit Unknown;
3. validate adaptation type;
4. validate required reason details;
5. persist Step 1;
6. persist mapping/repository provenance;
7. navigate to Patient Assessment.

Never save an AI suggestion as confirmed unless pharmacist selected it.

---

# 29. Consultation State Model

```ts
interface MedicationIndicationSelection {
  medicationConceptId: string;

  indicationConceptId?: string;
  indicationDisplay?: string;

  status: "confirmed" | "unknown";

  selectionSource:
    | "approved_mapping"
    | "patient_condition"
    | "ai_suggested"
    | "manual_search"
    | "unknown";

  mappingId?: string | null;
  repositoryVersion?: string | null;

  confirmedByPharmacist: boolean;

  selectedAt?: string;
}
```

---

# 30. Repository Integration

After medication normalization:

```ts
const result = await MedicationIndicationResolver.resolve({
  medicationConceptId,
  productConceptId,
  clinicalDrugConceptId,
  ingredientConceptIds,
  jurisdiction,
  patientConditionConceptIds
});
```

Consume:
- approvedMappings;
- patientConditionMatches;
- needsAIRanking;
- repositoryVersion.

---

# 31. AI Behavior

Invoke AI only when:

```text
approvedMappings.length > 1
AND
deterministic patient-condition matching does not yield one clear result
```

AI input:
- approved candidate IDs;
- minimum structured patient context.

AI may:
- rank approved candidates;
- provide a short factual reason.

AI must not:
- invent indication;
- diagnose;
- create SNOMED codes;
- auto-select;
- auto-confirm;
- modify patient conditions;
- modify medication.

UI language:

```text
Suggested based on patient information
```

Never display model confidence percentages.

---

# 32. Loading / Empty / Error States

Indication mapping loading:
- skeleton rows.

AI loading:

```text
Reviewing patient information...
```

No mapping:

```text
No approved indications are currently mapped for this medication.
Search all indications or select “Indication not known / not available.”
```

AI unavailable:
- keep deterministic results visible;
- show subtle unavailable state only.

Resolver failure:

```text
Indication suggestions are temporarily unavailable.
Search indications manually.
```

Do not block consultation.

---

# 33. Stale-State Rules

If medication identity changes:
- clear confirmed indication;
- rerun resolver;
- clear AI suggestions;
- revalidate adaptation reason.

If patient conditions change:
- rerun deterministic matching;
- optionally rerun AI ranking;
- do not silently change an already confirmed indication.

If indication changes after downstream steps exist:
- mark indication-dependent downstream Step 3 data stale.

---

# 34. Responsive Behavior

Desktop:
- main + right sidebar;
- indication split pane;
- 6 reason cards in row.

Tablet:
- sidebar moves below;
- reason cards 3 × 2;
- indication panes may stack.

Mobile:
- single column;
- compact stepper;
- reason cards 1 × 6 or 2 × 3;
- CTA full width.

---

# 35. Accessibility

Must support:
- keyboard navigation;
- visible focus rings;
- semantic tabs;
- semantic radio controls;
- `aria-label` for icon-only actions;
- color + non-color selection cues;
- accessible validation errors;
- proper disabled states.

---

# 36. Suggested Component Structure

```text
AdaptStep1
├── AdaptStepper
├── PageHeader
├── Step1MainGrid
│   ├── OriginalPrescriptionSection
│   │   ├── PrescriptionSummaryCard
│   │   └── MedicationIndicationPanel
│   │       ├── IndicationTabs
│   │       ├── IndicationSearch
│   │       ├── CommonIndicationsList
│   │       ├── PatientConditionsList
│   │       ├── AllIndicationsSearch
│   │       └── IndicationSuggestions
│   └── AdaptationReasonSection
│       ├── AdaptationTypeCards
│       └── AdaptationReasonDetails
├── Step1Sidebar
│   ├── PatientContextCard
│   ├── AboutIndicationCard
│   └── HelpfulResourcesCard
└── StepNavigationBar
```

---

# 37. Do Not Do

Do not:
- hardcode rosuvastatin indications;
- create a new frontend drug/condition repository;
- allow AI to invent indications;
- auto-confirm AI suggestions;
- expose raw mapping IDs in pharmacist UI;
- force pharmacist out of Step 1 for indication selection;
- block workflow because repository coverage is incomplete;
- show all SNOMED concepts without search/filter;
- duplicate full Patient Assessment UI here;
- alter existing CCDD/SNOMED infrastructure unnecessarily.

---

# 38. Acceptance Criteria

## Layout
- [ ] Matches supplied mock structure and visual hierarchy.
- [ ] Original prescription and indication are inside the same primary card.
- [ ] Reason for adaptation is the next major section.
- [ ] Sidebar cards match supplied layout.
- [ ] Bottom navigation matches supplied design.

## Indication
- [ ] Required unless Unknown selected.
- [ ] Common indications are repository-driven.
- [ ] Patient Conditions use structured patient data.
- [ ] All Indications uses current SNOMED search.
- [ ] Manual SNOMED selection can create review candidate.
- [ ] AI ranks only approved candidates.
- [ ] AI never auto-confirms.
- [ ] Confirmed indication persists downstream.

## Reason
- [ ] Six adaptation cards.
- [ ] One active type at a time.
- [ ] Follow-up reason controls appear contextually.
- [ ] Final CTA remains disabled until Step 1 is complete.

## Safety / robustness
- [ ] Mapping failure does not block consultation.
- [ ] AI failure does not block consultation.
- [ ] Medication change invalidates/revalidates indication.
- [ ] Remove prescription clears dependent Step 1 state.

---

# 39. Implementation Order

```text
1. Recreate page/layout shell from supplied mock.
2. Reuse existing Original Prescription component.
3. Build MedicationIndicationPanel.
4. Connect Approved Indications Repository resolver.
5. Connect existing SNOMED search.
6. Add patient-condition matching.
7. Add optional AI ranking hook.
8. Add six adaptation-type cards.
9. Add reason-specific detail controls.
10. Add sidebar cards.
11. Add Step 1 validation / CTA gating.
12. Add stale-state behavior.
13. Add responsive/accessibility behavior.
14. Add unit/integration/UI tests.
```

---

# 40. Final Experience

```text
Review prescription
        ↓
See approved/common indications
        ↓
See patient-context suggestions
        ↓
Pharmacist confirms indication
        ↓
Select reason for adaptation
        ↓
Continue to Patient Assessment
```

The UI should feel intelligent and efficient, while the pharmacist remains the person who confirms the indication.
