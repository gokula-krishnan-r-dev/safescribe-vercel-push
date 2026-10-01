# SafeScribe Adapt Module — AI-Generated Counselling Cards 2–4
## Cursor-Ready Developer Instructions

**Scope:** SafeScribe **Adapt** module only.

**Goal:** Automatically generate counselling content for Cards **2, 3 and 4** using AI because SafeScribe does not maintain a manually curated backend counselling repository for every medication.

Card 1 remains deterministic and backend-owned from the pharmacist-confirmed adapted prescription.

---

# 1. Final Counselling Model

The Adapt module counselling section contains four cards:

```text
1. How to use your medicine
2. What to expect
3. Self-care & non-drug measures
4. Follow-up & when to seek care
```

Source ownership must be:

```text
Card 1 → backend / confirmed adapted prescription
Cards 2–4 → AI-generated draft
```

Do not use AI to reconstruct Card 1.

---

# 2. Card Ownership

## Card 1 — How to use your medicine

Source:

```text
confirmed adapted prescription
→ display_name
→ patient_directions
```

Render exactly:

```text
{display_name}: {patient_directions}
```

Do not regenerate from:

- dose
- route
- frequency
- duration
- quantity
- indication
- structured prescription fields

`patient_directions` is the canonical patient-facing SIG.

---

## Card 2 — What to expect

AI-generated.

Purpose:

- explain the expected treatment response;
- explain what the patient may reasonably notice after starting/continuing the adapted therapy;
- include a treatment-response timeline only when it is well established and supported.

Do not turn Card 2 into disease-description content.

Do not include diagnostic symptoms merely because they describe the condition.

---

## Card 3 — Self-care & non-drug measures

AI-generated.

Purpose:

- practical supportive measures;
- lifestyle/non-drug advice directly relevant to the confirmed indication and treatment;
- condition-specific prevention/support where appropriate.

Do not generate generic filler.

If no meaningful self-care recommendation exists:

return an empty array.

---

## Card 4 — Follow-up & when to seek care

AI-generated, but with stronger backend controls.

Internally separate:

```json
{
  "routine_follow_up": [],
  "seek_care": []
}
```

The UI may display them together in one card.

Routine follow-up must use structured pharmacist-confirmed follow-up data when available.

Do not allow the AI to invent:

- follow-up intervals;
- lab timing;
- numeric thresholds;
- referral timing;
- clinician responsibility.

---

# 3. High-Level Flow

```text
Pharmacist confirms adaptation
        ↓
Canonical adapted prescription
        ↓
Confirmed indication
        ↓
Adaptation type/reason
        ↓
Relevant confirmed patient context
        ↓
Structured follow-up plan, if available
        ↓
AI counselling generator
        ↓
Cards 2–4 JSON
        ↓
Validation
        ↓
Pharmacist review/edit
        ↓
Patient Care Summary
```

---

# 4. Required Input Payload

Recommended input:

```json
{
  "module": "adapt",

  "medication": {
    "display_name": "Ramipril 5 mg",
    "ingredient": "ramipril",
    "dosage_form": "capsule",
    "route": "oral"
  },

  "adapted_prescription": {
    "patient_directions": "Take 1 capsule by mouth once daily."
  },

  "indication": {
    "display_name": "Hypertension",
    "pharmacist_confirmed": true
  },

  "adaptation": {
    "type": "dose_change",
    "reason": "Dose adjusted following pharmacist assessment"
  },

  "patient_context": {
    "age": 67,
    "sex": "female",
    "pregnancy": null,
    "breastfeeding": null,
    "relevant_conditions": [],
    "relevant_allergies": [],
    "relevant_medications": [],
    "relevant_labs": []
  },

  "follow_up_plan": {
    "responsible_party": "Pharmacist",
    "timeframe": "2 weeks",
    "monitoring_targets": [
      "blood pressure",
      "treatment tolerability"
    ],
    "action_if_not_met": "reassess therapy or refer as clinically appropriate",
    "pharmacist_confirmed": true
  }
}
```

Only send confirmed patient-specific information.

---

# 5. AI Output Schema

The AI must return JSON only:

```json
{
  "what_to_expect": [],
  "self_care": [],
  "routine_follow_up": [],
  "seek_care": []
}
```

Each array contains short patient-facing strings.

Example:

```json
{
  "what_to_expect": [
    "This medicine is intended to help keep your blood pressure controlled over time.",
    "You may not feel noticeably different even when the medicine is working."
  ],
  "self_care": [
    "Continue the diet, activity, and lifestyle measures recommended for your blood pressure."
  ],
  "routine_follow_up": [
    "Follow up with your pharmacist in 2 weeks to review your blood pressure and how you are tolerating the treatment."
  ],
  "seek_care": [
    "Contact your pharmacist or another healthcare provider if your condition is worsening or the treatment is not working as expected."
  ]
}
```

---

# 6. Card 2 Guardrails

`what_to_expect` must:

- describe treatment response;
- remain conservative;
- be patient-friendly;
- avoid unsupported timelines.

Do not generate:

```text
High blood pressure often causes no symptoms.
```

unless it is specifically useful as treatment-expectation context.

Prefer:

```text
You may not feel noticeably different even when the medicine is working.
```

Do not invent:

- onset of action;
- exact number of days to improvement;
- probability of response;
- cure rates.

If uncertain:

omit.

---

# 7. Card 3 Guardrails

`self_care` should contain only clinically relevant non-drug measures.

Good:

```text
Keep the affected area clean and avoid products that irritate the skin.
```

Bad:

```text
Eat healthy and exercise regularly.
```

unless those measures are clearly relevant to the confirmed indication.

Do not produce lifestyle boilerplate.

---

# 8. Card 4 — Routine Follow-Up

If a confirmed structured follow-up plan exists:

use it as the source of truth.

Example input:

```json
{
  "responsible_party": "Pharmacist",
  "timeframe": "2 weeks",
  "monitoring_targets": [
    "blood pressure",
    "treatment tolerability"
  ],
  "pharmacist_confirmed": true
}
```

Expected:

```text
Follow up with your pharmacist in 2 weeks to review your blood pressure and how you are tolerating the treatment.
```

The AI may make wording patient-friendly but must preserve:

- responsible party;
- timeframe;
- monitoring targets.

Do not modify those facts.

---

# 9. Card 4 — Seek Care

`seek_care` may contain conservative escalation advice.

The AI may use well-established medication/condition knowledge to create a draft, but must not invent:

- patient-specific diagnoses;
- new contraindications;
- numeric thresholds;
- urgency levels unsupported by context;
- referral plans already assigned to a specific provider.

Use plain patient-facing wording.

Prefer:

```text
Contact your pharmacist or another healthcare provider if your symptoms are worsening or the treatment is not working as expected.
```

Avoid alarmist language.

---

# 10. No Medication-Dose Generation

Cards 2–4 must never contain reconstructed prescription instructions.

Do not allow:

```text
Take 5 mg once daily.
```

unless it is quoted from confirmed `patient_directions` for context and specifically allowed.

Default:

do not repeat SIG in Cards 2–4.

Card 1 already contains it.

---

# 11. AI Knowledge Boundary

The AI may use general medication and condition knowledge to draft Cards 2–4 because a complete counselling repository does not exist.

However, it must not independently decide or invent:

- diagnosis;
- dose;
- route;
- frequency;
- duration;
- indication;
- adaptation reason;
- monitoring interval;
- lab schedule;
- referral decision;
- pharmacist follow-up responsibility;
- patient-specific contraindication.

Those must come from structured confirmed inputs.

---

# 12. Omit Rather Than Guess

If the model cannot confidently produce a safe, useful point:

return an empty array or fewer bullets.

Do not force every card to contain content.

Examples:

```json
{
  "what_to_expect": [],
  "self_care": [],
  "routine_follow_up": [],
  "seek_care": []
}
```

is preferable to unsupported content.

---

# 13. Bullet Limits

Recommended maximum:

```text
Card 2: 2–4 bullets
Card 3: 2–4 bullets
Card 4 routine follow-up: 1–2 bullets
Card 4 seek care: 1–3 bullets
```

Keep each bullet short.

Avoid mini-monographs.

---

# 14. Pharmacist Review Requirement

AI-generated Cards 2–4 are drafts.

Before content is used in the Patient Care Summary:

```text
AI draft
→ pharmacist review
→ pharmacist edit if needed
→ pharmacist confirms
→ downstream use
```

Recommended field:

```json
{
  "ai_generated": true,
  "pharmacist_confirmed": false
}
```

After review:

```json
{
  "pharmacist_confirmed": true
}
```

---

# 15. Save Confirmed Counselling

After pharmacist confirmation, store the exact confirmed counselling content.

Recommended:

```json
{
  "confirmed_counselling": {
    "what_to_expect": [],
    "self_care": [],
    "routine_follow_up": [],
    "seek_care": [],
    "confirmed_at": "",
    "confirmed_by": ""
  }
}
```

Do not regenerate after confirmation unless pharmacist explicitly requests regeneration.

---

# 16. Regenerate Button

Recommended UI:

```text
Regenerate suggestions
```

If clicked:

- generate fresh Cards 2–4;
- do not overwrite pharmacist edits silently;
- ask for confirmation if confirmed content would be replaced.

---

# 17. UI Mapping

Existing cards:

```text
1. How to use your medicine
2. What to expect
3. Self-care & non-drug measures
4. Follow-up & when to seek care
```

Map:

```text
Card 1 ← canonical patient_directions
Card 2 ← what_to_expect
Card 3 ← self_care
Card 4 ← routine_follow_up + seek_care
```

Within Card 4, optionally visually distinguish:

```text
Follow-up
When to seek care
```

without adding unnecessary complexity.

---

# 18. Loading State

When Cards 2–4 are generating:

```text
Generating guidance...
```

Do not show:

```text
No Patient Guidance is available...
```

until generation actually fails or returns empty.

---

# 19. Empty-State Copy

If a card legitimately returns no content:

Card 2:

```text
No additional treatment-expectation guidance was generated.
```

Card 3:

```text
No additional self-care guidance was generated.
```

Card 4:

```text
No additional follow-up guidance was generated.
```

Do not imply a pathway repository is missing.

This is Adapt, not a pathway-driven counselling repository.

---

# 20. Failure Handling

If AI generation fails:

- Card 1 remains available;
- Cards 2–4 show a recoverable state;
- pharmacist may retry;
- pharmacist may manually add counselling.

Suggested:

```text
Unable to generate counselling guidance. Retry or add guidance manually.
```

Do not block the entire Adapt encounter solely because AI counselling generation failed.

---

# 21. Validation Rules

Before showing AI output:

```text
[ ] valid JSON
[ ] only expected keys
[ ] arrays only
[ ] strings only inside arrays
[ ] no dose reconstruction
[ ] no invented follow-up interval
[ ] no invented monitoring threshold
[ ] no duplicated Card 1 SIG
[ ] no unsupported patient-specific claim
[ ] no internal/system terminology
[ ] bullet count within limits
```

---

# 22. Follow-Up Integrity Validation

If structured `follow_up_plan` exists:

validate that AI output preserves:

```text
responsible_party
timeframe
monitoring_targets
```

If any are changed:

reject AI output and fall back to deterministic rendering of the confirmed follow-up plan.

Recommended:

```typescript
if (!matchesConfirmedFollowUp(aiOutput.routine_follow_up, followUpPlan)) {
  aiOutput.routine_follow_up = renderConfirmedFollowUpDeterministically(followUpPlan);
}
```

---

# 23. Deterministic Follow-Up Fallback

Recommended helper:

```typescript
function renderConfirmedFollowUpDeterministically(plan) {
  if (!plan?.pharmacist_confirmed) return [];

  const who =
    plan.responsible_party === "Pharmacist"
      ? "your pharmacist"
      : plan.responsible_party;

  const targets = joinNaturalLanguage(plan.monitoring_targets);

  return [
    `Follow up with ${who} ${formatTimeframe(plan.timeframe)} to review ${targets}.`
  ];
}
```

Use confirmed data only.

---

# 24. Patient Care Summary Integration

The final Patient Care Summary should use pharmacist-confirmed counselling only.

Do not directly export unreviewed AI content.

Pipeline:

```text
AI-generated counselling
→ pharmacist confirms
→ confirmed counselling
→ canonical English Patient Care Summary
→ translation if non-English
```

The translation pipeline remains unchanged.

---

# 25. Auditability

Store:

```text
generation timestamp
model identifier/version
prompt version
input payload hash/reference
AI output
pharmacist edits
final confirmed content
confirmation timestamp
confirming pharmacist
```

Do not expose this metadata in the patient handout.

---

# 26. Example — Adapted Statin

Input:

```json
{
  "medication": {
    "display_name": "Pravastatin"
  },
  "adapted_prescription": {
    "patient_directions": "Take 1 tablet by mouth once daily for 30 days."
  },
  "indication": {
    "display_name": "Hyperlipidemia",
    "pharmacist_confirmed": true
  },
  "follow_up_plan": {
    "responsible_party": "Pharmacist",
    "timeframe": "4 weeks",
    "monitoring_targets": [
      "treatment tolerability",
      "response to therapy"
    ],
    "pharmacist_confirmed": true
  }
}
```

Possible AI output:

```json
{
  "what_to_expect": [
    "This medicine is intended to help lower cholesterol over time.",
    "You may not feel noticeably different even when the medicine is working."
  ],
  "self_care": [
    "Continue heart-healthy eating and regular physical activity as recommended."
  ],
  "routine_follow_up": [
    "Follow up with your pharmacist in 4 weeks to review how you are tolerating the medicine and your response to therapy."
  ],
  "seek_care": [
    "Contact your pharmacist or another healthcare provider if you develop new or concerning symptoms after starting the adapted treatment."
  ]
}
```

---

# 27. Acceptance Criteria

This feature is complete when:

- Card 1 always comes from canonical confirmed `patient_directions`;
- Cards 2–4 are generated automatically using AI;
- AI generation works without a medication-by-medication counselling repository;
- confirmed indication and patient context are included in the AI payload;
- confirmed pharmacist follow-up data override AI invention;
- the AI cannot invent a follow-up interval;
- Cards 2–4 remain short and patient-facing;
- the pharmacist can edit all generated content;
- pharmacist confirmation is required before downstream use;
- failed AI generation does not block manual counselling;
- confirmed counselling is stored and reused in Patient Care Summary;
- non-English translation happens only after canonical English counselling is confirmed.
