# SafeScribe Patient Handout Generator - Developer Instructions

**Document:** v1.0  
**Module:** Consultation Documents - Patient Care Summary / Patient Handout  
**Primary use:** Pharmacist-reviewed, patient-facing print or PDF summary of the care plan  
**Jurisdictional baseline:** Alberta  
**Status:** Implementation specification

## 1. Objective

Implement a Patient Handout generator that automatically assembles confirmed SafeScribe consultation data and approved patient-education content into a concise, clear, patient-facing care summary.

The handout must:

1. Explain the final care plan in plain language.
2. Include exact treatment directions without changing or recalculating them.
3. Tell the patient what improvement to expect.
4. Include relevant self-care, precautions, follow-up, and instructions for seeking additional care.
5. Use only confirmed consultation data and approved, versioned handout content.
6. Silently omit missing optional information.
7. Never invent, assume, normalize, or imply missing clinical facts.
8. Remain editable and require explicit pharmacist review before final print or download.
9. Render as a clean, accessible one- or two-page document.

This feature does not conduct a new assessment, create a new care plan, or provide autonomous medical advice. It explains the plan already selected and confirmed by the pharmacist.

## 2. Controlling Product Decision

The Patient Handout is not:

- A simplified copy of the pharmacist consultation note.
- A copy of the PCP communication.
- A pathway-question export.
- A complete drug monograph.
- A substitute for verbal pharmacist counselling.

It is a short, individualized after-visit summary that answers:

1. What was the outcome of today's consultation?
2. What treatment or action was recommended?
3. How should the treatment be used?
4. What should the patient expect?
5. What else can the patient do?
6. When and how will follow-up occur?
7. When should the patient seek additional or urgent care?

The internal document type remains:

```ts
type DocumentType = "PATIENT_CARE_SUMMARY";
```

The user-facing title may be `Your Care Plan`, `Patient Care Summary`, or another approved product label. Use one label consistently across the document card, editor, PDF, and print view.

## 3. Regulatory and Clinical Purpose

The handout supports clear patient communication and implementation of the care plan.

The Alberta College of Pharmacy standards require pharmacists to communicate at an appropriate level using plain language, confirm patient understanding, provide appropriate information about prescribed therapy, and support patients with information about benefits, relevant adverse effects, interactions, monitoring, and follow-up. Written information may complement verbal communication, but it must not replace required verbal communication.

Accordingly:

- SafeScribe may generate the written handout only after the counselling content has been reviewed in the workflow.
- Generating or giving the handout does not prove that verbal counselling occurred.
- The handout must not state that the patient understood, agreed, or was counselled unless the corresponding workflow state is explicitly confirmed.
- The handout is not the permanent clinical record. The pharmacist consultation note remains the record of assessment, rationale, care, and counselling.

Plain-language targets in this specification are based on Alberta Health Services and Health Canada guidance: use direct language, short sentences, active voice, meaningful headings, adequate white space, and a general Grade 5-8 reading target without altering exact medical terms or directions.

Official references:

- [ACP Standards of Practice for Pharmacists and Pharmacy Technicians - Standards 3.1, 3.2, 7.8, and 7.9](https://abpharmacy.ca/wp-content/uploads/Standards_SPPPT.pdf)
- [Alberta Health Services - Plain Language Tips](https://www.albertahealthservices.ca/news/Page16967.aspx)
- [Health Canada - Product Monograph Plain-Language Guidance](https://www.canada.ca/en/health-canada/services/drugs-health-products/drug-products/applications-submissions/guidance-documents/product-monograph/frequently-asked-questions-product-monographs-posted-health-canada-website.html)
- [OpenAI Structured Outputs guidance](https://developers.openai.com/api/docs/guides/structured-outputs)
- [OpenAI Responses API migration and storage guidance](https://developers.openai.com/api/docs/guides/migrate-to-responses)

## 4. Terminology

Use these terms consistently:

| Term | Meaning |
| --- | --- |
| Counselling screen | The pharmacist-facing SafeScribe step used to review and confirm concise counselling and follow-up content. |
| Patient Handout | The final patient-facing document. |
| Patient Care Summary | The existing SafeScribe document-manifest name for the Patient Handout. |
| Summary item | A concise point shown on the counselling screen. |
| Detailed handout item | Approved patient-facing content eligible for the final handout. |
| Protected fragment | Exact text created by application code that the model may position but not alter. |
| Required handout concept | A semantic item that must be represented for the applicable pathway, treatment, or care outcome. |
| Approved content | Clinically reviewed, versioned content published in the SafeScribe repository. |
| Source snapshot | Immutable, filtered consultation and content data used to create one handout version. |

## 5. Scope

This ticket includes:

- Source-data projection from the finalized consultation.
- Selection of approved detailed handout content.
- Missing-data and non-invention behaviour.
- Patient-friendly organization and wording.
- Exact-value protection for treatment directions, dates, and follow-up.
- Structured AI input and output contracts.
- Deterministic validation and rendering.
- Pharmacist review and editing.
- Print and PDF output.
- Delivery-status recording.
- Versioning, invalidation, privacy controls, and acceptance tests.

This ticket does not include:

- New clinical assessment questions.
- A new counselling interview at the document stage.
- Autonomous treatment selection.
- Medication-safety evaluation inside the language model.
- Use of the public internet at runtime to create patient advice.
- Reproduction of copyrighted guideline or monograph text.
- Direct delivery by email, SMS, or patient portal unless separately approved.
- Replacing verbal counselling with the generated document.

## 6. Handout Outcome Modes

The generator must support more than prescribing consultations.

```ts
type HandoutOutcomeMode =
  | "PRESCRIPTION_ISSUED"
  | "OTC_OR_SELF_CARE"
  | "SUPPORTIVE_CARE_ONLY"
  | "REFERRAL_ROUTINE"
  | "REFERRAL_URGENT"
  | "NO_TREATMENT_REQUIRED"
  | "TREATMENT_DECLINED"
  | "PHARMACIST_DECLINED_TO_PRESCRIBE";
```

The mode controls which sections are required and which are omitted.

| Mode | Required patient-facing content |
| --- | --- |
| Prescription issued | Care outcome, exact medication directions, relevant administration guidance, expected response, relevant precautions, follow-up, and when to seek care |
| OTC or self-care | Product or action selected, exact directions when applicable, expected response, follow-up, and when to seek care |
| Supportive care only | Recommended self-care, expected course, follow-up, and when to seek care |
| Routine referral | Reason for referral in patient-friendly language, where or whom to contact, timeframe, interim advice, and when urgency should increase |
| Urgent referral | Deterministic urgent-action banner, destination/action, timing, and interim safety instructions |
| No treatment required | Confirmed care decision, relevant self-care or observation instructions, expected course, follow-up, and when to seek care |
| Treatment declined | What was offered when appropriate, the agreed alternative plan, follow-up, and when to seek care; do not disclose a reason unless it is appropriate and confirmed for the patient copy |
| Pharmacist declined to prescribe | Patient-friendly explanation of the disposition, alternative recommendation or referral, follow-up, and when to seek care |

Do not force medication sections into non-medication outcomes.

## 7. Source-of-Truth Rule

The generator may select, organize, deduplicate, and simplify supplied content. It must not add clinical facts or general medical knowledge.

Only the following sources may feed the handout:

1. Pharmacist-entered or pharmacist-confirmed consultation data.
2. Final pathway responses that are appropriate to disclose to the patient.
3. Deterministic eligibility, safety, referral, and urgency results.
4. Final selected treatments and exact prescription fields.
5. Counselling items confirmed in the Counselling and Follow-up step.
6. Approved `detailedHandoutItems` linked to the active pathway and selected treatment.
7. Final monitoring and follow-up instructions.
8. Confirmed referral instructions.
9. Approved pharmacy contact and encounter metadata.
10. AI-extracted information only after pharmacist confirmation.

Do not use:

- A raw transcript.
- Unconfirmed intake extraction.
- An outdated treatment selection.
- Hidden internal notes not intended for the patient.
- Raw clinical questions.
- Model memory or public web content.
- Unpublished or unapproved repository content.

When duplicate or conflicting values exist, apply this precedence:

1. Most recent pharmacist edit.
2. Explicit final workflow selection.
3. Deterministic rule-engine output.
4. Published pathway or treatment handout content for the active version.
5. Confirmed AI extraction.
6. Earlier structured value.

If a material conflict remains, block generation. Do not ask the model to resolve it.

## 8. Approved Handout Content Repository

The handout must be grounded in a versioned content repository. Do not ask the language model to create medication counselling or condition education from scratch.

The existing two-level counselling model must be preserved:

```ts
type CounsellingSection = {
  id: string;
  title: string;
  summaryItems: string[];
  detailedHandoutItems: DetailedHandoutItem[];
};

type DetailedHandoutItem = {
  id: string;
  category:
    | "MEDICATION_COUNSELLING"
    | "SELF_CARE"
    | "EXPECTED_RESPONSE"
    | "FOLLOW_UP"
    | "WHEN_TO_SEEK_CARE";
  text: string;
  importance: "REQUIRED" | "RECOMMENDED" | "OPTIONAL";
  appliesWhen: RuleExpression;
  treatmentId?: string;
  pathwayVersionId: string;
  locale: string;
  approvalStatus: "DRAFT" | "CLINICAL_REVIEW" | "APPROVED" | "RETIRED";
  approvedBy?: string;
  approvedAt?: string;
  sourceReferences: Array<{
    referenceId: string;
    sourceVersion?: string;
  }>;
};
```

Runtime rules:

- Only `APPROVED` content may be used.
- Use content from the exact active pathway and treatment versions.
- Evaluate `appliesWhen` deterministically before the model call.
- Do not include content merely because it is generally true.
- Do not send source-reference text to the model unless it is separately licensed and required.
- Do not reproduce guideline or product-monograph passages verbatim.
- If a required handout concept has no approved active content, fail readiness validation instead of asking the model to invent it.

`summaryItems` are not automatically eligible for the final handout. They may be reused only when the repository record explicitly marks the same approved text for both display levels.

## 9. Upstream Validation Versus Document Generation

Mandatory clinical and counselling information must be resolved before document generation.

Examples of blocking upstream issues:

- Unresolved red flag or referral decision.
- Unresolved medication-allergy or other safety conflict.
- Treatment selection no longer valid.
- Missing value required for the selected dose.
- Missing exact prescription directions.
- Counselling content changed after confirmation.
- Required follow-up plan absent.
- Required urgent-care instruction absent.
- Missing approved handout content required by the pathway manifest.

The document stage must not ask the patient or pharmacist to complete missing clinical information. It reports the problem and links back to the owning workflow step.

Minimum readiness requirements:

| Outcome | Required source data before generation |
| --- | --- |
| Every handout | Care outcome, active pathway/outcome version, confirmed counselling version, encounter date, pharmacy contact, and valid safety state |
| Medication recommended | Exact medication/product record, exact directions, applicable approved counselling, expected-response guidance, and follow-up/safety-net plan |
| Self-care only | Approved self-care items, expected course, follow-up, and when-to-seek-care guidance |
| Referral | Referral urgency, destination or action, timing, interim advice, and escalation instructions |
| No treatment | Confirmed disposition, observation/self-care plan when applicable, follow-up, and escalation instructions |

Generation behaviour:

| Source state | Behaviour |
| --- | --- |
| Required value missing | Block before model call |
| Optional value missing or `null` | Omit completely |
| Field not asked | Omit; do not convert to a negative |
| Explicitly not applicable | Omit |
| Explicitly unavailable and patient-relevant | Include only when the pharmacist intentionally selected a patient-facing unavailable-information statement |
| Confirmed negative finding | Include only when it helps the patient understand the plan and is approved for patient display |
| Unresolved contradiction | Block and return to the relevant step |

## 10. Missing-Information Rules

These rules are non-negotiable.

### 10.1 Optional missing information

If optional information is absent:

- Omit the sentence or section.
- Do not show an empty heading.
- Do not show a dash, blank label, placeholder, or question.
- Do not ask for it during document generation.
- Do not write `not available` by default.
- Do not state or imply that it is normal, negative, or not relevant.

If no blood pressure or laboratory result was entered, the handout simply omits blood pressure and laboratory information.

### 10.2 Missing is not negative

Maintain distinct source states:

```ts
type AnswerState =
  | "CONFIRMED_PRESENT"
  | "CONFIRMED_ABSENT"
  | "EXPLICITLY_UNAVAILABLE"
  | "NOT_ASKED"
  | "NOT_APPLICABLE";
```

`NOT_ASKED`, `NOT_APPLICABLE`, empty text, and `null` must never become `none`, `normal`, `no concerns`, `you do not have`, or `you are not at risk`.

### 10.3 Explicitly unavailable information

The handout should rarely mention missing clinical information. Include it only when:

1. The pharmacist explicitly marked it for patient disclosure; and
2. The patient needs the statement to carry out the plan safely.

The exact patient-facing wording must be supplied or approved upstream. The model must not create a clinical explanation for why the information was unavailable.

### 10.4 Do not invent patient actions

Never state that the patient:

- Understood the plan.
- Agreed to treatment.
- Declined treatment.
- Received counselling.
- Was given a referral.
- Contacted another provider.
- Completed follow-up.

unless the corresponding action is explicitly confirmed and appropriate for the patient copy.

## 11. Safety Blocking

The generator must receive a current deterministic safety state.

```ts
type HandoutSafetyState =
  | "CLEAR"
  | "REFERRAL_PLAN_CONFIRMED"
  | "UNRESOLVED_CONFLICT"
  | "OUTDATED";
```

Rules:

- `CLEAR`: generate the applicable treatment handout.
- `REFERRAL_PLAN_CONFIRMED`: generate the referral-mode handout.
- `UNRESOLVED_CONFLICT`: block handout generation.
- `OUTDATED`: require the safety engine to re-evaluate before generation.

For an unresolved conflict, show:

> **Treatment safety review required**  
> The selected care plan conflicts with current patient safety information. Return to Treatment Options and resolve the selection before creating the patient handout.

Do not show normal treatment instructions beside an unresolved allergy, contraindication, interaction, or referral conflict.

Urgent or emergency directions must come from deterministic rule output and approved jurisdictional wording. The model must not independently decide urgency or generate emergency-service instructions.

## 12. Patient, Recipient, and Encounter Header

### 12.1 Identifiers

Patient identifiers are optional for this document unless a local workflow policy requires them.

Default header fields:

- `Patient` - full name when entered and selected for the handout.
- `Date` - consultation date.
- `Care plan for` - patient-friendly pathway or concern label when confirmed.
- `Pharmacist` - name.
- `Pharmacy` - name and contact information.
- `Consultation ID` - optional; do not show if it adds no value to the patient.

Do not include by default:

- PHN or patient ID.
- Full date of birth.
- Street address.
- Unrelated medical conditions.
- Internal pathway IDs or versions.

The application renders identifiers deterministically. Do not send them to the language model.

### 12.2 Recipient mode

```ts
type HandoutRecipientMode =
  | "PATIENT"
  | "PARENT_OR_GUARDIAN"
  | "CAREGIVER_OR_AGENT";
```

The recipient mode must be selected or derived from confirmed workflow data, not guessed by the model.

- `PATIENT`: use `you` and `your`.
- `PARENT_OR_GUARDIAN`: use `your child` where natural.
- `CAREGIVER_OR_AGENT`: use approved neutral wording such as `the person you care for` or the person's confirmed name.

Do not infer gender, family relationship, or decision-making authority.

### 12.3 Language and locale

For the MVP, generate only in clinically approved locales.

- Default locale: `en-CA`.
- Use Canadian spelling.
- Do not translate medication directions, safety instructions, or clinical content through an unrestricted prompt.
- A new locale requires approved repository content, a locale-specific prompt, validation, and clinical review.
- If the requested locale is not supported, show a clear unsupported-language message and continue with the approved locale only if the pharmacist chooses it.

## 13. Patient-Facing Content Structure

Use a short title and conditionally render only populated sections.

Recommended order:

1. `Your care plan`
2. `Your treatment`
3. `How to use your medicine`
4. `What to expect`
5. `What you can do`
6. `Side effects and precautions`
7. `Follow-up`
8. `When to get help`

The exact section set depends on the outcome. Do not render all eight headings when fewer are needed.

### 13.1 Your care plan

Provide a brief, patient-friendly description of:

- What the pharmacist assessed or discussed.
- The confirmed care outcome.
- The main action the patient should take.

Do not include:

- Detailed clinical reasoning.
- Differential diagnoses.
- Eligibility criteria.
- Internal safety classifications.
- Claims that a diagnosis was confirmed when the pharmacist recorded only a working assessment.

Preserve uncertainty. For example, a source recorded as `consistent with` must not become `you have`.

### 13.2 Your treatment

Show each selected treatment in a visually distinct block.

Include as applicable:

- Medication or product name.
- Strength and dosage form.
- Patient-friendly purpose when supplied.
- Exact directions.
- Duration or stop date.
- Other selected non-drug action.

Use the final prescription or treatment object as the source. Do not reconstruct directions from counselling prose.

### 13.3 How to use your medicine

Include only approved, applicable administration instructions, such as:

- Timing in relation to food when relevant.
- Technique or device instructions.
- What to do about a missed dose when an approved item is present.
- Storage, handling, or disposal when applicable.
- Adherence advice that was confirmed and not redundant.

Storage belongs here or under `Side effects and precautions`; it must not be classified as self-care.

Do not automatically add `complete the full course`. Include it only when the approved content specifically requires it and it adds information not already clear from the exact regimen.

### 13.4 What to expect

Describe only approved expected treatment or condition outcomes:

- What improvement may occur.
- Approximate time to improvement when supplied.
- What lack of improvement means for follow-up.

Do not place medication adverse effects in this section.

Do not promise effectiveness. Use the degree of certainty in the approved source, such as `may help`, `should begin to improve`, or `is expected to`.

### 13.5 What you can do

Include relevant approved non-drug and self-care actions.

Rules:

- Use short action-oriented bullets.
- Include only actions applicable to the confirmed patient context.
- Do not include generic wellness advice unrelated to the consultation.
- Do not include medication storage or administrative information here.

### 13.6 Side effects and precautions

Include only common or important effects and precautions selected from approved content and applicable to the patient.

Prioritize:

1. What the patient might notice.
2. What the patient can do.
3. When the patient should stop, contact the pharmacy, or seek care, if the approved plan says so.

Do not generate an exhaustive adverse-effect list. Do not copy product-monograph sections. Do not add interactions or restrictions from model knowledge.

### 13.7 Follow-up

State the confirmed follow-up plan in actionable language:

- When follow-up will occur.
- Who will initiate it.
- How it will occur when captured.
- What will be checked.
- What the patient should do if follow-up does not occur, when included in the plan.

Do not say `follow up as needed` when a specific interval exists.

Do not invent an appointment date from a relative interval. The application may calculate a date only when an approved deterministic rule explicitly defines that calculation and the source date/time zone are known.

### 13.8 When to get help

Separate urgency levels when both are present:

- `Contact the pharmacy or a healthcare provider` for non-urgent concerns.
- `Get urgent medical care` for confirmed urgent triggers.
- Approved emergency wording only when supplied by the deterministic rule set.

Patient-facing statements must be instructions, not raw questions.

Incorrect:

> Does the patient have fever or worsening symptoms?

Correct structure:

> Contact a healthcare provider if [approved patient-facing trigger].

The trigger wording must come from approved content. The model may simplify it but may not add new symptoms or change urgency.

## 14. Content Selection and Deduplication

The application should pre-select applicable content before the model call.

Required steps:

1. Resolve the active pathway and treatment versions.
2. Evaluate `appliesWhen` rules.
3. Exclude unpublished, retired, irrelevant, or unconfirmed items.
4. Map each item to a patient-facing section.
5. Identify required semantic concepts.
6. Collapse exact duplicate content records.
7. Convert exact clinical values into protected fragments.
8. Send the remaining closed content set to the model for organization and limited plain-language editing.

The model may deduplicate equivalent ideas, but it must report which source items were represented or omitted.

Reject these duplication patterns:

- Directions shown once as `2 g` and again as `2,000 mg`.
- The same duration repeated in the medication block and a separate bullet without additional meaning.
- `Take until finished` when the exact one-day or single-dose regimen already communicates completion.
- The same urgent-care trigger in two different sections.
- Repeated generic statements such as `take as directed` beside exact directions.

Do not deduplicate two distinct safety instructions merely because they share words.

## 15. Plain-Language and Health-Literacy Rules

The output should:

- Aim for approximately Grade 5-8 readability.
- Use `you`, `your child`, or the approved recipient wording.
- Use active voice.
- Use common words before technical terms.
- Explain an unavoidable technical term the first time it appears.
- Use short sentences, usually one main idea per sentence.
- Use short paragraphs and bullets.
- Use Arabic numerals.
- Use clear action verbs.
- Use respectful, inclusive, non-stigmatizing, gender-neutral language.
- Use Canadian spelling.

The output should not:

- Use DAP or SOAP labels.
- Use clinical workflow terms such as `red flag`, `eligible`, `contraindicated`, `differential diagnosis`, or `rule triggered` in patient-facing content unless an approved item intentionally explains the term.
- Use unexplained abbreviations such as `PRN`, `PO`, `BID`, `TID`, or `QID`.
- Use vague phrases such as `use as directed` when exact directions are available.
- Use generic filler such as `monitor symptoms closely` without saying what to monitor and what action to take.
- Use alarming language when calm, direct wording is sufficient.
- Use minimizing or stigmatizing language.

Readability scoring is a QA aid, not a clinical rewrite authority:

- Calculate a readability score after token replacement.
- Flag content above the approved threshold for pharmacist review.
- Do not automatically alter a drug name, exact direction, medical term, or safety instruction merely to improve the score.
- Exclude protected fragments and unavoidable drug names from automated rewrite decisions where feasible.

## 16. Exact-Value Protection

The model must not freely rewrite high-risk values.

Render these deterministically or protect them with immutable tokens:

- Drug and product names.
- Strengths and dosage forms.
- Dose, route, frequency, duration, quantity, and refills when displayed.
- Start, stop, or follow-up dates.
- Laboratory values and units when intentionally included.
- Vital signs and units when intentionally included.
- Allergy names and reactions when intentionally included.
- Referral destination, urgency, and timing.
- Pharmacy and pharmacist contact details.
- Patient identifiers.
- Pathway-approved emergency-service wording.

Recommended approach:

1. The server builds a patient-friendly exact fragment.
2. The model receives a token such as `[[TREATMENT_1]]`, `[[FOLLOWUP_1]]`, or `[[URGENT_ACTION_1]]`.
3. The model may position the token in an allowed section.
4. The model must not alter, translate, expand, convert, or repeat the token.
5. The application validates required-token use.
6. The application replaces the token with the exact fragment.

Example deterministic treatment fragment:

```text
{drug} {strength} {dosage form}: {patient-friendly directions}. {duration statement when applicable}.
```

Do not ask the model to:

- Calculate age or eGFR.
- Convert mg to g.
- Calculate quantity.
- Create a calendar date from an interval.
- Convert professional SIG abbreviations.
- Choose a safer or simpler regimen.

SIG-to-patient-direction conversion must occur through a tested deterministic service or pharmacist-confirmed directions before handout generation.

## 17. Generation Input Contract

Create an immutable snapshot immediately before generation.

```ts
type HandoutContentCategory =
  | "CARE_PLAN"
  | "MEDICATION_COUNSELLING"
  | "SELF_CARE"
  | "EXPECTED_RESPONSE"
  | "FOLLOW_UP"
  | "WHEN_TO_SEEK_CARE";

type PatientHandoutSnapshotV1 = {
  schemaVersion: "patient-handout-source.v1";
  consultationId: string;
  consultationVersion: number;
  createdAt: string;
  jurisdiction: "AB" | string;
  locale: "en-CA" | string;
  outcomeMode: HandoutOutcomeMode;
  recipientMode: HandoutRecipientMode;

  patientHeader: {
    displayName?: string;
    includeName: boolean;
  };

  encounterHeader: {
    occurredAt: string;
    pharmacistName: string;
    pharmacyName: string;
    pharmacyPhone?: string;
    pharmacyAddress?: string;
  };

  careOutcome: {
    sourceId: string;
    certainty:
      | "CONFIRMED"
      | "CONSISTENT_WITH"
      | "SUSPECTED"
      | "UNSPECIFIED";
    patientFacingText?: string;
    protectedToken?: string;
  };

  treatmentRecords: Array<{
    treatmentId: string;
    treatmentType: "PRESCRIPTION" | "OTC" | "NON_DRUG";
    protectedTreatmentToken: string;
    required: boolean;
    sourceVersion: number;
  }>;

  contentItems: Array<{
    contentId: string;
    category: HandoutContentCategory;
    text?: string;
    protectedToken?: string;
    importance: "REQUIRED" | "RECOMMENDED" | "OPTIONAL";
    patientFacing: true;
    sourceType:
      | "PHARMACIST_ENTRY"
      | "CONFIRMED_AI_EXTRACTION"
      | "PATHWAY_CONTENT"
      | "TREATMENT_CONTENT"
      | "RULE_ENGINE"
      | "PRESCRIPTION"
      | "COUNSELLING_CONFIRMATION"
      | "FOLLOW_UP_PLAN";
    sourceRecordId: string;
    sourceVersion: number;
    approvalStatus: "APPROVED" | "PHARMACIST_CONFIRMED";
  }>;

  requiredConcepts: Array<{
    conceptId: string;
    category: HandoutContentCategory;
    satisfiedByContentIds: string[];
  }>;

  safetyState: HandoutSafetyState;
  protectedFragments: Record<string, string>;
  counsellingVersion: number;
  sourceSnapshotHash: string;
};
```

Before calling the model:

- Remove patient name and all direct identifiers.
- Remove pharmacist registration number and unnecessary provider identifiers.
- Exclude PHN, DOB, address, phone number, and raw consultation ID.
- Exclude content that is not approved or pharmacist-confirmed.
- Exclude items that failed deterministic applicability rules.
- Exclude empty items and `NOT_ASKED` or `NOT_APPLICABLE` facts.
- Replace high-risk values with protected tokens.
- Reject contradictory active items.
- Reject an unresolved or outdated safety state.
- Confirm that every required concept has at least one eligible source item.
- Confirm that every protected token has a server-side fragment.
- Confirm that the counselling version is the currently confirmed version.

## 18. Model Output Contract

Use Structured Outputs with a strict JSON schema. Do not request unrestricted HTML, Markdown, or PDF content from the model.

```ts
type PatientHandoutSectionId =
  | "CARE_PLAN"
  | "TREATMENT"
  | "HOW_TO_USE"
  | "EXPECTED_RESPONSE"
  | "SELF_CARE"
  | "SIDE_EFFECTS_PRECAUTIONS"
  | "FOLLOW_UP"
  | "WHEN_TO_GET_HELP";

type GeneratedPatientHandoutV1 = {
  schemaVersion: "patient-handout-output.v1";
  documentTitle: string;
  sections: Array<{
    sequence: number;
    sectionId: PatientHandoutSectionId;
    heading: string;
    introduction: string;
    items: Array<{
      text: string;
      sourceContentIds: string[];
      protectedTokens: string[];
    }>;
  }>;
  representedContentIds: string[];
  omittedContent: Array<{
    contentId: string;
    reason: "DUPLICATE" | "LOW_PRIORITY" | "NOT_NEEDED_FOR_CLARITY";
  }>;
  qualityFlags: Array<
    | "SOURCE_CONFLICT"
    | "UNSUPPORTED_INPUT"
    | "MISSING_REQUIRED_CONTENT"
    | "MISSING_REQUIRED_TOKEN"
    | "UNABLE_TO_GENERATE"
  >;
};
```

Schema rules:

- Use `additionalProperties: false` on every object.
- Require every schema key.
- Use empty strings and arrays only where the schema requires them; the renderer must not display empty content.
- Restrict `sectionId` and omission `reason` to enums.
- Reject any source content ID or token not present in the request.
- Reject omission of a non-duplicative required concept.
- Reject a refusal, incomplete response, parse failure, or non-empty `qualityFlags` as generation failure.
- Do not display partial output as a completed handout.

Structured Outputs constrains shape; it does not establish clinical truth. Application-side source validation is mandatory.

## 19. Model Prompt

Store and version this prompt outside application code where possible.

### 19.1 Developer/system instruction

```text
You create a patient-facing SafeScribe care handout from a closed set of
confirmed consultation content and approved patient-education items.

Your role is limited to organization, selection, deduplication, and careful
plain-language editing. You do not assess the patient, make clinical decisions,
calculate values, choose treatment, change urgency, or add medical knowledge.

Use only supplied content items and protected tokens. Every factual or
instructional statement must cite one or more supplied content IDs or contain
an allowed protected token. Do not use outside knowledge.

If optional information is absent, omit it silently. Do not leave blank
headings, labels, placeholders, questions, or statements that information was
not available. Do not infer that an unanswered, omitted, unknown, or
not-applicable item is absent, negative, normal, or safe.

Never invent symptoms, diagnoses, negative findings, examination findings,
vitals, laboratory results, medication details, allergies, side effects,
interactions, precautions, self-care, actions, counselling, consent, referrals,
urgency, monitoring, or follow-up.

Preserve uncertainty. Do not turn "consistent with" or "suspected" into "you
have" or a confirmed diagnosis.

Preserve every protected token exactly. Do not alter, translate, expand,
abbreviate, calculate, convert, or repeat a token.

Write concise Canadian patient-facing English. Use the supplied recipient mode:
- PATIENT: use "you" and "your";
- PARENT_OR_GUARDIAN: use "your child" where natural;
- CAREGIVER_OR_AGENT: use only the supplied approved recipient wording.
Do not infer gender or relationship.

Aim for a Grade 5-8 reading level. Use active voice, common words, short
sentences, short paragraphs, and action-oriented bullets. Explain an unavoidable
technical term in plain language. Do not simplify drug names or protected
clinical instructions.

Organize content only into the allowed sections. Omit sections that have no
supported content. Put medication effects and precautions under the appropriate
section, not under expected response. Put storage and administration guidance
under how to use or precautions, not self-care.

Do not reproduce raw clinical questions, internal workflow labels, DAP/SOAP
content, differential diagnoses, eligibility logic, clinician rationale, source
citations, Markdown, HTML, or tables.

Do not add generic phrases such as "take as directed", "complete the full
course", "monitor closely", or "follow up as needed" unless the exact idea is
explicitly supplied and adds useful information.

Return only the required structured output.
```

### 19.2 Request payload

```text
Create the Patient Handout from PATIENT_HANDOUT_CONTENT below.

OUTCOME_MODE:
{outcomeMode}

RECIPIENT_MODE:
{recipientMode and approved recipient wording, if needed}

PATIENT_HANDOUT_CONTENT:
{de-identified and filtered contentItems with allowed source IDs}

REQUIRED_CONCEPTS:
{required concept IDs and eligible source content IDs}

PROTECTED_TOKENS:
{allowed token names only; deterministic values remain server-side}
```

Do not include patient name, PHN, DOB, address, phone number, consultation ID, pharmacy address, pharmacist registration number, or other direct identifiers in the model request.

## 20. API and Privacy Requirements

- Use the OpenAI Responses API unless the approved SafeScribe integration uses another supported endpoint.
- Use strict Structured Outputs through `text.format` for Responses API implementations.
- Set `store: false` for requests containing consultation data.
- Use the approved documentation model route; do not hardcode a model in the UI.
- Use low and consistent variability where the selected model/API supports it.
- Do not provide web, file-search, database, or clinical tool access to the generation call.
- Do not send raw audio or transcript content.
- Do not log request bodies, response bodies, protected fragments, or final handout content in routine logs.
- Logs may contain non-PHI operational metadata: request ID, prompt version, model route, latency, token usage, success/failure code, and source snapshot hash.
- Encrypt consultation and generated document data in transit and at rest according to the approved privacy design.
- Apply the SafeScribe temporary-retention policy to drafts and versions.
- Do not include direct identifiers in analytics, error monitoring, or prompt-evaluation datasets.
- Use synthetic or formally de-identified cases for prompt and regression testing.

## 21. Post-Generation Validation

Run deterministic validation before displaying the handout.

Required checks:

1. JSON schema parsed successfully.
2. Response completed without refusal or truncation.
3. Section sequence is valid.
4. Section IDs are allowed for the current outcome mode.
5. No empty rendered section exists.
6. Every cited content ID exists in the request.
7. Every used protected token was allowed.
8. Every required protected token appears exactly once unless the manifest explicitly permits another count.
9. Every required semantic concept is represented.
10. Every omitted required or recommended item has a permitted omission reason.
11. No unknown placeholder remains.
12. No raw assessment question appears.
13. No direct patient identifier appears in model-generated text.
14. No unsupported number, unit, date, time, or dosage appears.
15. No medication direction is duplicated or converted into an equivalent value.
16. No diagnosis-certainty escalation appears.
17. No medication adverse effect appears under expected response.
18. No storage statement appears under self-care.
19. No urgent action is downgraded, upgraded, or moved into a routine section.
20. Final treatment, follow-up, and urgent-action fragments exactly match server values after token replacement.

Recommended additional checks:

- Flag professional abbreviations such as `BID`, `TID`, `QID`, `PRN`, and `PO` outside protected fragments.
- Flag internal terms such as `red flag`, `eligibility`, `contraindication`, `differential`, `rule engine`, and `pathway response`.
- Flag unsupported generic statements such as `you are safe`, `there are no concerns`, or `your results are normal`.
- Flag claims that the patient understood, agreed, declined, received counselling, or was referred unless supported.
- Flag repeated concepts using semantic similarity plus source-ID comparison.
- Calculate readability for review and flag content above the configured threshold.
- Confirm all URLs and QR codes come from the approved static-resource allowlist.

If validation fails:

- Do not silently remove a potentially unsafe statement and show the remainder as complete.
- Retry once only for technical or formatting failures.
- Do not retry to resolve a source conflict or missing approved content.
- After a second technical failure, show `Patient handout generation failed - create or edit the handout manually`.
- Preserve non-PHI error metadata and the source hash without logging document content.

## 22. Final Rendering and PDF Layout

The application, not the model, renders the document.

Render in this order:

1. SafeScribe/pharmacy brand area.
2. Deterministic document title.
3. Optional patient and encounter header.
4. Validated care-plan sections.
5. Deterministic pharmacy contact block.
6. Deterministic footer disclaimer, generated date, and page number.

### 22.1 Page format

- Standard page: US Letter, 8.5 x 11 inches.
- Target: one page for an uncomplicated consultation.
- Maximum default: two pages.
- Do not remove required safety or follow-up content to force one page.
- Use margins of approximately 0.55-0.7 inches.
- Avoid content splitting in the middle of a treatment block or urgent-care panel.
- Repeat the patient name only when configured and needed on page 2.
- Show `Page 1 of 2` when more than one page is rendered.

### 22.2 Typography and accessibility

- Body text: at least 11 pt for print; 12 pt preferred.
- Section headings: approximately 15-17 pt.
- Document title: approximately 20-22 pt.
- Line height: approximately 1.35-1.5.
- Use a highly legible sans-serif typeface.
- Maintain strong contrast.
- Do not rely on colour alone to communicate urgency.
- Use icons only as secondary cues and include a text heading.
- Keep bullet indentation shallow and consistent.
- Maintain visible white space between sections.
- Do not use dense tables for instructions.

### 22.3 Visual priority

Use stronger emphasis for:

1. Exact treatment directions.
2. Follow-up date or interval.
3. Urgent-care instructions.

Use a calm, high-contrast alert panel for `When to get help`. Reserve red styling for genuinely urgent or emergency content supplied by the rule engine. Routine return advice should use amber or neutral styling.

### 22.4 Deterministic footer

Use approved wording such as:

> This handout summarizes the care plan discussed with your pharmacist. Follow the directions on your prescription label. Contact the pharmacy if the information differs or if you have questions.

This footer is a product disclaimer, not a substitute for specific instructions. Do not use it to excuse missing treatment, follow-up, or safety content.

Include pharmacy name, phone number, and hours only when present in the approved pharmacy profile. Omit empty labels.

### 22.5 Output formats

- Primary: PDF suitable for printing.
- Optional: accessible HTML print view.
- Do not use a screenshot as the downloadable document.
- Generated PDFs must contain selectable text.
- Apply document language metadata.
- Use semantic HTML headings and lists in the accessible view.
- If tagged PDF support is available, preserve heading and list structure.

## 23. Example of Intended Style

This example demonstrates structure only. Tokens represent deterministic values supplied by the application.

```text
YOUR CARE PLAN

Today's plan
[[CARE_OUTCOME_1]]

Your treatment
[[TREATMENT_1]]

How to use it
- [Approved administration instruction.]
- [Approved treatment-specific precaution.]

What to expect
- [Approved expected-response statement.]
- [Approved instruction if improvement does not occur.]

What you can do
- [Approved self-care action.]

Follow-up
[[FOLLOWUP_1]]

When to get help
- [Approved routine return instruction.]
[[URGENT_ACTION_1]]

Questions? Contact [[PHARMACY_CONTACT_1]].
```

The final document must not show token names, square brackets, source IDs, or placeholder text.

## 24. Special Outcome Handling

### 24.1 Multiple treatments

- Render each treatment in its own block.
- Keep each exact direction attached to the correct treatment.
- Attach treatment-specific counselling by `treatmentId`.
- Put shared follow-up and safety instructions after the treatment blocks.
- Reject ambiguous advice that could apply to more than one medication.

### 24.2 Treatment declined

- Focus on the agreed next step.
- Do not use judgmental language.
- Do not state or infer the patient's reason unless explicitly confirmed and appropriate for the patient copy.
- Include any accepted alternative, referral, follow-up, and safety-net instructions.

### 24.3 Pharmacist declined to prescribe

- Do not say the patient was `ineligible` without explanation.
- Use the pharmacist-confirmed patient-facing disposition.
- Include the recommended alternative or referral.
- Include what the patient should do next and when urgency changes.

### 24.4 Urgent referral

- Place the urgent action immediately below the title.
- Render the action deterministically.
- Do not bury it on page 2.
- Do not let the model soften or expand the urgency statement.
- Omit routine self-care that could distract from or delay the urgent action unless explicitly approved as interim advice.

### 24.5 Pediatric or caregiver handout

- Use the confirmed recipient mode.
- Keep dose and age/weight-dependent values protected.
- Do not infer weight, caregiver relationship, or administration ability.
- Use `your child` only when the recipient is confirmed as a parent or guardian.

### 24.6 No medication selected

- Omit treatment and medication sections.
- Do not show `No medications` unless that statement is intentionally part of the confirmed care plan.
- Render self-care, observation, referral, follow-up, and safety sections as applicable.

## 25. Review and Editing Workflow

The initial document status is `REVIEW_REQUIRED`.

Selecting `Review & edit` opens the final rendered handout in an editable document view.

Required controls:

- Edit patient-facing text.
- Restore the current generated version.
- Save edits.
- Preview print layout.
- Mark reviewed.
- Print.
- Download PDF.

To mark reviewed, require explicit pharmacist confirmation:

> I have reviewed this patient handout and confirm that it accurately reflects the care plan, treatment instructions, follow-up, and safety advice discussed with the patient or caregiver.

Rules:

- Opening, previewing, printing, or downloading does not mark the handout reviewed.
- Final print and download remain disabled until the current version is reviewed.
- If draft export is supported, watermark every page `DRAFT - REVIEW REQUIRED`.
- Editing a reviewed handout creates a new version and returns it to `UPDATED_REVIEW_REQUIRED`.
- Regeneration creates a new version and returns it to `REVIEW_REQUIRED`.
- Relevant upstream changes set the current handout to `SOURCE_CHANGED`.
- Only the exact reviewed version may be printed or downloaded as final.

Pharmacist edits are clinical content. Preserve the edited version and review metadata for the active SafeScribe retention period.

## 26. Counselling and Handout States Must Remain Separate

Do not merge these concepts:

```ts
type CounsellingStatus =
  | "REVIEW_REQUIRED"
  | "CONFIRMED_WITH_PATIENT"
  | "SOURCE_CHANGED";

type HandoutReviewStatus =
  | "GENERATING"
  | "REVIEW_REQUIRED"
  | "UPDATED_REVIEW_REQUIRED"
  | "SOURCE_CHANGED"
  | "REVIEWED"
  | "GENERATION_FAILED";

type HandoutProvisionStatus =
  | "NOT_RECORDED"
  | "NOT_PROVIDED"
  | "PRINTED"
  | "DOWNLOADED"
  | "CONFIRMED_PROVIDED"
  | "PATIENT_DECLINED"
  | "NOT_APPLICABLE";
```

Rules:

- Counselling confirmation is completed in the Counselling and Follow-up step.
- Handout review confirms the document is accurate.
- Handout provision records whether the patient received or declined the document.
- None of these states automatically satisfies another state.
- A generated or reviewed handout does not prove counselling occurred.
- A printed or downloaded handout does not prove the patient received it.
- A provided handout does not prove patient understanding.

## 27. Recording Handout Provision

After final print or download, optionally prompt:

> Was the patient or caregiver provided with this handout?

Options:

- `Provided`
- `Patient/caregiver declined`
- `Not provided`

Record:

- Handout version ID.
- Provision status.
- Method when known: printed copy, secure approved digital route, or other approved method.
- Pharmacist/user ID.
- Date and time.

Do not claim digital delivery merely because a PDF was downloaded to the user's device. Email, SMS, or portal delivery requires a separate verified integration and delivery state.

Handout provision is not an Alberta requirement for every consultation. Do not block consultation completion solely because provision is unrecorded unless the active service/pathway configuration explicitly requires it. Required verbal counselling and the pharmacist's clinical documentation remain separate obligations.

## 28. Completion Gating

For a required Patient Handout document, the Consultation Documents screen may report it complete only when:

1. Generation succeeded.
2. Source snapshot remains current.
3. No unresolved safety conflict exists.
4. The current handout version was explicitly reviewed.

Provision status is recorded separately and is not part of the default document-review count.

If the handout is optional for a consultation outcome, the document manifest must identify it as optional. Do not silently count an optional document as required.

## 29. Invalidation Rules

Invalidate the Patient Handout when any content-affecting upstream value changes.

| Upstream change | Handout result |
| --- | --- |
| Pathway or care outcome | `SOURCE_CHANGED` |
| Treatment selection | `SOURCE_CHANGED`; regenerate treatment-dependent content |
| Dose, route, frequency, duration, quantity, or formulation | `SOURCE_CHANGED` |
| Counselling item or confirmation version | `SOURCE_CHANGED` |
| Expected-response guidance | `SOURCE_CHANGED` |
| Self-care recommendation | `SOURCE_CHANGED` |
| Follow-up plan | `SOURCE_CHANGED` |
| Referral urgency or destination | `SOURCE_CHANGED` |
| Safety rule result | `SOURCE_CHANGED` or immediate block |
| Recipient mode or output locale | `SOURCE_CHANGED` |
| Included patient name | Header version changes; require review again |
| PCP recipient or transmission status | No effect unless it changes the care plan |

When invalidating:

- Preserve the old version as historical during the retention period.
- Disable final print/download for the outdated version.
- Create a new source snapshot.
- Regenerate.
- Require pharmacist review again.

## 30. Versioning and Data Model

```ts
type PatientHandoutVersion = {
  id: string;
  consultationId: string;
  documentType: "PATIENT_CARE_SUMMARY";
  versionNumber: number;
  status: HandoutReviewStatus;
  outcomeMode: HandoutOutcomeMode;
  locale: string;
  recipientMode: HandoutRecipientMode;
  sourceSnapshotHash: string;
  sourceConsultationVersion: number;
  sourceCounsellingVersion: number;
  promptVersion: string;
  modelRoute: string;
  generatedContentJson: GeneratedPatientHandoutV1;
  generatedHtml: string;
  generatedPlainText: string;
  editedContentJson?: GeneratedPatientHandoutV1;
  editedHtml?: string;
  generatedAt: string;
  editedBy?: string;
  editedAt?: string;
  reviewedBy?: string;
  reviewedAt?: string;
  finalPdfObjectKey?: string;
  provisionStatus: HandoutProvisionStatus;
  provisionMethod?: "PRINT" | "APPROVED_DIGITAL" | "OTHER_APPROVED";
  provisionRecordedBy?: string;
  provisionRecordedAt?: string;
};
```

Rules:

- Never overwrite a reviewed version in place.
- Preserve generated and pharmacist-edited content separately.
- Tie review to an exact version ID and source hash.
- Do not store handout content in general application or audit logs.
- Do not treat a cached PDF as current after source invalidation.
- Remove temporary content and files according to the approved retention policy.
- If a handed-out document must be corrected, create a new corrected version; do not silently replace the record of what was previously generated.

## 31. Suggested Service Interface

```ts
generatePatientHandout(consultationId: string): Promise<{
  documentId: string;
  versionId: string;
  status: "REVIEW_REQUIRED" | "GENERATION_FAILED";
}>;

updatePatientHandout(
  versionId: string,
  editedContent: GeneratedPatientHandoutV1
): Promise<{
  newVersionId: string;
  status: "UPDATED_REVIEW_REQUIRED";
}>;

markPatientHandoutReviewed(
  versionId: string,
  pharmacistId: string
): Promise<{
  status: "REVIEWED";
  reviewedAt: string;
}>;

renderPatientHandoutPdf(versionId: string): Promise<{
  versionId: string;
  pdfDownloadToken: string;
}>;

recordPatientHandoutProvision(
  versionId: string,
  status:
    | "CONFIRMED_PROVIDED"
    | "PATIENT_DECLINED"
    | "NOT_PROVIDED",
  method?: "PRINT" | "APPROVED_DIGITAL" | "OTHER_APPROVED"
): Promise<{
  provisionStatus: HandoutProvisionStatus;
  recordedAt: string;
}>;
```

Backend authorization must confirm that the user belongs to the consultation tenant and is permitted to generate, edit, review, print, download, or record provision.

Use short-lived, authorized download tokens. Do not expose permanent public PDF URLs.

## 32. Failure Messages

### Required consultation information missing

> **Patient handout cannot be created**  
> Required care-plan information is incomplete. Return to the highlighted consultation step.

### Counselling no longer confirmed

> **Counselling review required**  
> Counselling content changed after it was confirmed. Review it again before creating the handout.

### Missing approved content

> **Approved handout content is incomplete**  
> Required patient instructions are not available for the selected pathway or treatment version. Complete the handout manually or contact a SafeScribe administrator.

### Treatment safety conflict

> **Treatment safety review required**  
> Resolve the current treatment conflict before creating the patient handout.

### Source conflict

> **Conflicting care-plan information**  
> SafeScribe found inconsistent source information. Return to the relevant step and confirm the correct value.

### Generation failed

> **Patient handout generation failed**  
> Try again once. If the issue continues, create or edit the handout manually.

### Source changed

> **Handout update required**  
> The care plan changed after this handout was created. Generate and review a new version.

### Not reviewed

> **Review required before final output**  
> Review the current handout before printing or downloading it as final.

### Unsupported language

> **Handout language not available**  
> This language is not yet clinically approved for Patient Handouts. Select an approved language.

### PDF rendering failed

> **PDF could not be created**  
> Your reviewed handout is still available. Try the PDF again or use the approved print view.

## 33. Acceptance Tests

### Purpose and structure

1. The document reads as a patient care summary, not a clinical note or PCP letter.
2. Only sections supported by source content are rendered.
3. Empty sections and labels never appear.
4. An uncomplicated consultation normally fits on one page without removing required content.
5. A complex handout may expand to two pages without splitting treatment or urgent-care blocks incorrectly.
6. Patient identifiers remain optional and PHN, DOB, and address are excluded by default.

### Source grounding

7. Every generated statement maps to an allowed content ID or protected token.
8. Unapproved, retired, or wrong-version content is rejected.
9. Raw transcripts never enter the handout generation request.
10. Raw assessment questions never appear in the handout.
11. Public web content and model memory are not used at runtime.
12. A required concept without approved content blocks generation.

### Missing data and non-invention

13. Missing blood pressure produces no blood-pressure sentence, blank, or `not available` statement.
14. Missing laboratory data produces no laboratory sentence or implied normal result.
15. `NOT_ASKED` never becomes a negative finding.
16. The model does not invent symptoms, diagnosis, treatment, self-care, side effects, or follow-up.
17. The model does not claim that the patient understood, agreed, declined, or received counselling without a supporting source.
18. Missing optional content removes the affected item or section without prompting the pharmacist at document generation.

### Exact treatment values

19. Drug name, strength, dosage form, dose, route, frequency, duration, quantity, and refills remain exact where displayed.
20. `2 g` is not repeated as `2,000 mg`.
21. Professional SIG abbreviations do not reach the patient handout unless part of an approved protected fragment, which should normally be rejected upstream.
22. The model does not calculate dose, quantity, age, eGFR, or dates.
23. Each treatment keeps its own directions and counselling when multiple treatments exist.
24. A changed medication direction invalidates the reviewed handout.

### Content classification and deduplication

25. Medication adverse effects appear under precautions, not expected response.
26. Storage instructions do not appear under self-care.
27. `Complete the full course` is not added when unsupported or redundant.
28. `Take as directed` is not added beside exact directions.
29. Equivalent counselling items are shown once.
30. Distinct urgent-care instructions are not incorrectly merged.

### Plain language and accessibility

31. The handout uses the confirmed recipient mode and does not infer gender or relationship.
32. Patient-mode output uses `you` and `your` naturally.
33. Parent/guardian mode uses `your child` only when confirmed.
34. Internal terms such as `eligibility`, `red flag`, `rule engine`, and `differential diagnosis` are absent from normal patient-facing output.
35. Unavoidable technical terms are explained without changing exact protected content.
36. Readability above the configured threshold creates a review flag, not an unsafe automatic rewrite.
37. PDF text is selectable and the print view remains legible in grayscale.
38. Urgency is communicated by text and hierarchy, not colour alone.

### Safety and referral

39. An unresolved medication-allergy conflict blocks handout generation.
40. An outdated safety state blocks generation until re-evaluated.
41. A confirmed routine referral generates routine referral instructions without medication sections unless applicable.
42. A confirmed urgent referral places deterministic urgent action near the top of page 1.
43. The model cannot upgrade or downgrade referral urgency.
44. A raw red-flag question is converted only when an approved patient-facing instruction exists; otherwise it is omitted or causes readiness failure if required.

### Review and lifecycle

45. A newly generated handout is `REVIEW_REQUIRED`.
46. Viewing or previewing does not mark it reviewed.
47. Final print and download are disabled until the current version is reviewed.
48. Editing reviewed content creates a new version and requires review again.
49. Regeneration creates a new version and requires review again.
50. Treatment, counselling, follow-up, recipient, or locale changes invalidate the current handout.
51. Only the exact reviewed version can be rendered as the final PDF.
52. Draft export, when supported, is visibly watermarked on every page.

### Counselling and provision states

53. A generated handout does not mark verbal counselling complete.
54. A reviewed handout does not mark it provided to the patient.
55. Printing or downloading does not prove provision.
56. `CONFIRMED_PROVIDED` requires an explicit user action tied to the document version.
57. `PATIENT_DECLINED` is recorded without judgmental wording.
58. Unrecorded handout provision does not block completion unless configuration explicitly makes it required.

### Privacy and technical controls

59. Direct identifiers are rendered outside the model response.
60. PHN, DOB, address, and phone number are absent from model requests.
61. `store: false` is set for the approved Responses API implementation.
62. Request bodies, response bodies, and final handout content are absent from routine logs.
63. Structured-output schema violations fail safely without showing partial content.
64. Protected tokens are validated and replaced server-side.
65. Download links are authorized and short-lived.
66. Temporary PDFs and document versions follow the approved retention policy.

## 34. Definition of Done

This feature is complete when:

- The Patient Handout is generated from an immutable, source-bound snapshot.
- Only approved and applicable patient-education content is eligible.
- The output uses plain, patient-friendly Canadian English.
- Treatment, follow-up, referral, and urgency values are deterministic and exact.
- Missing optional information disappears cleanly.
- Required missing or conflicting information fails before model generation.
- No unsupported clinical statement can pass post-generation validation.
- The current handout version requires explicit pharmacist review.
- Final PDF and print output are accessible, legible, and version-specific.
- Counselling, document review, and handout provision remain separate states.
- Upstream changes reliably invalidate the document.
- Privacy, logging, authorization, and retention controls are implemented.
- All acceptance tests pass with synthetic or formally de-identified data.

## 35. Superseded Requirements

For this document type, this specification supersedes older references to:

- A generic `patient counselling summary` generated from a transcript.
- A handout created directly from raw pathway questions.
- A handout that repeats the full consultation note.
- A fixed three-document final screen.
- A single `Generated` state with no review lifecycle.
- Treating print or download as proof that the handout was provided.
- Treating a written handout as proof of verbal counselling or patient understanding.
- Allowing the model to create missing medication, self-care, follow-up, or urgent-care advice.

The final product decision is a versioned, pharmacist-reviewed, patient-facing care summary assembled from confirmed consultation data and approved detailed handout content, with deterministic protection of all exact and safety-critical instructions.
