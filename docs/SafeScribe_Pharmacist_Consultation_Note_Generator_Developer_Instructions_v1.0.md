# SafeScribe Pharmacist Consultation Note Generator - Developer Instructions

**Document:** v1.0  
**Module:** Consultation Documents - Pharmacist Consultation Note  
**Primary use:** Pharmacist review, editing, and plain-text copy into the patient's Kroll profile  
**Jurisdictional baseline:** Alberta  
**Status:** Implementation specification

## 1. Objective

Implement a Pharmacist Consultation Note generator that automatically assembles the completed SafeScribe consultation into a concise, clinically complete note.

The note must:

1. Follow DAP principles internally.
2. Read as a natural clinical narrative externally.
3. Never display `D:`, `A:`, `P:`, `Data`, `Assessment`, or `Plan` headings.
4. Use only information already captured or confirmed in the SafeScribe workflow.
5. Omit optional information that was not captured.
6. Never invent, assume, normalize, or imply missing clinical facts.
7. Remain editable and require explicit pharmacist review.
8. Produce clean plain text that can be copied directly into Kroll.

This feature does not conduct a new assessment. It documents the assessment that has already been completed.

## 2. Controlling Product Decision

DAP is the hidden organizational framework, not the visible output format.

The generated note should normally contain:

1. A compact encounter and patient-information header.
2. One or two short paragraphs describing the reason for care, relevant history, and confirmed findings.
3. One short paragraph documenting the pharmacist's assessment and rationale.
4. One or two short paragraphs documenting the care provided, counselling, monitoring, follow-up, referrals, and communications.

The final note must read from beginning to end as one coherent clinical account. It must not resemble a questionnaire, workflow export, transcript, or list of database fields.

Typical uncomplicated notes should usually be approximately 250-500 words. This is a target, not a hard limit. Do not add filler to reach it and do not remove clinically important content to stay within it.

## 3. Regulatory Purpose

The note should allow another pharmacist or an ACP reviewer to understand:

- Why the patient sought care.
- What relevant information was collected.
- What the pharmacist concluded.
- Why the decision was reasonable.
- What treatment or other care was provided.
- What counselling, monitoring, and follow-up were arranged.
- Who provided the care, when, where, and by what method.

The Alberta College of Pharmacy's current standards require documentation that supports professional accountability, collaboration, and continuity of care. Appendix E describes patient-record and record-of-care elements, including the reason for care, goals, information collected, decisions and rationale, options considered, care provided, monitoring, follow-up, and the responsible pharmacist.

Important distinction:

- ACP's Appendix E applies to the complete patient record.
- A single consultation note does not need to repeat every unrelated field already held in the Kroll patient profile.
- The generated note must include core patient identifiers and every captured patient fact relevant to this encounter.
- Do not dump unrelated profile information into the clinical narrative merely because it exists.

Official references:

- [ACP Standards of Practice for Pharmacists and Pharmacy Technicians - Domain 7 and Appendix E](https://abpharmacy.ca/wp-content/uploads/Standards_SPPPT.pdf)
- [OpenAI Structured Outputs guidance](https://developers.openai.com/api/docs/guides/structured-outputs)
- [OpenAI Responses API migration and storage guidance](https://developers.openai.com/api/docs/guides/migrate-to-responses)

## 4. Scope

This ticket includes:

- Source-data projection from the completed consultation.
- Missing-data behaviour.
- DAP-informed narrative generation.
- AI prompt and structured-output contract.
- Deterministic rendering of identifiers and exact clinical values.
- Review and editing.
- Kroll copy formatting.
- Versioning, invalidation, and audit metadata.
- Quality controls and acceptance tests.

This ticket does not include:

- Reopening the clinical interview to collect missing optional data.
- Asking new assessment questions during document generation.
- Making a diagnosis or treatment decision.
- Re-running medication-safety rules inside the language model.
- Generating the prescription, prescriber communication, or patient handout.
- Direct integration with Kroll unless a separate approved integration is implemented.

## 5. Source-of-Truth Rule

The document generator may transform, organize, deduplicate, and narrate confirmed consultation data. It must not introduce new clinical facts.

Only the following sources may feed the note:

1. Pharmacist-entered or pharmacist-confirmed structured fields.
2. Final responses to pathway questions.
3. Deterministic eligibility, red-flag, and medication-safety results.
4. The pharmacist-confirmed clinical impression and rationale.
5. Final selected treatment and prescription fields.
6. Counselling items confirmed as provided.
7. The finalized monitoring and follow-up plan.
8. Confirmed referrals and communications.
9. Patient and encounter metadata collected earlier in the workflow.
10. AI-extracted facts only after the pharmacist has confirmed them.

Do not send the raw transcript directly to the final-note generator. A transcript may contain speech-recognition errors, hypothetical discussion, negated statements, or information that the pharmacist rejected. Only confirmed structured facts derived from it are eligible.

When duplicate or conflicting values exist, apply this precedence:

1. Most recent pharmacist edit.
2. Explicit final workflow selection.
3. Deterministic rule-engine result.
4. Confirmed AI extraction.
5. Earlier structured value.

If a clinically material conflict remains unresolved, block generation. Do not ask the language model to choose which value is correct.

## 6. Upstream Validation Versus Document Generation

Mandatory clinical information must be enforced before the user reaches document generation.

Examples include:

- A mandatory pathway question.
- An unresolved red flag.
- A required treatment-eligibility criterion.
- An unresolved medication-allergy conflict.
- A missing value required to calculate a selected dose.
- A missing monitoring plan when one is required for the care decision.

The document stage must not ask for this information and must not invent a substitute.

Minimum source requirements depend on the encounter outcome:

| Outcome | Required structured source data before generation |
| --- | --- |
| Every consultation | Reason for care, pharmacist's clinical conclusion or disposition, care outcome, encounter date/time, and pharmacist identity |
| Prescription issued | Indication, pharmacist assessment summary, prescribing rationale, exact prescription record, relevant safety result, and monitoring/follow-up plan |
| Referral | Reason or trigger, urgency, destination or action, safety-net direction, and follow-up responsibility when applicable |
| Treatment declined | Option offered, confirmed reason declined, resulting care decision, and how the patient's needs were accommodated |
| Pharmacist declined to prescribe | Criteria not met or confirmed reason, alternative recommendation/referral, and follow-up or safety-net plan |

These values must be collected in the relevant earlier workflow step. The note generator reports a readiness failure; it does not open a document-generation questionnaire.

Generation behaviour:

| Source state | Document behaviour |
| --- | --- |
| Required clinical field missing | Block upstream; do not call the model |
| Optional field missing or `null` | Omit it completely |
| Field not asked | Omit it; do not convert it to a negative |
| Explicitly not applicable | Omit it unless the status itself is clinically material |
| Explicitly unavailable and clinically material | Include a precise statement that it was unavailable |
| Confirmed negative finding | Include it when relevant to eligibility, referral, or rationale |
| Unresolved contradiction | Block generation and return to the relevant workflow step |

## 7. Missing-Information Rules

These rules are non-negotiable.

### 7.1 Optional missing information

If optional data is absent, silently omit it.

For example, if no blood pressure or laboratory result was entered:

- Do not leave a blank field.
- Do not show a placeholder.
- Do not ask for the value.
- Do not write `not available` by default.
- Do not state or imply that the value was normal.

### 7.2 Missing is not negative

The following states are distinct and must remain distinct:

```ts
type AnswerState =
  | "CONFIRMED_PRESENT"
  | "CONFIRMED_ABSENT"
  | "EXPLICITLY_UNAVAILABLE"
  | "NOT_ASKED"
  | "NOT_APPLICABLE";
```

`NOT_ASKED`, `NOT_APPLICABLE`, an empty string, and `null` must never render as `denies`, `none`, `normal`, `negative`, or `no concerns`.

### 7.3 Explicitly unavailable information

Include unavailable information only when both conditions are true:

1. The pharmacist explicitly recorded that it was unavailable; and
2. Its absence materially affected or qualified the decision.

Acceptable:

> Current renal function was not available for review; the pharmacist's documented decision and rationale were [confirmed rationale].

Do not create the rationale. It must already be present in the consultation data.

### 7.4 No invented objective findings

Never generate:

- Physical-examination findings that were not performed and recorded.
- Otoscopy findings based only on symptoms.
- Temperatures, blood pressures, weights, laboratory values, or calculated values not present in source data.
- Statements such as `patient appears well` unless that observation was explicitly documented.

If the encounter was based on reported symptoms and records, simply narrate those sources. Do not add a generic statement that no physical examination occurred unless the pharmacist recorded it and it is relevant to interpreting the assessment.

## 8. Patient and Encounter Header

The application must render the header deterministically. Do not send direct identifiers to the language model and do not ask the model to reproduce them.

Header order:

```text
PHARMACIST CONSULTATION - {SERVICE TYPE}
Patient: {patient name}
DOB: {YYYY-MM-DD}
PHN / Patient ID: {identifier}
Address: {single-line address}
Date/time: {local date and time with time zone}
Encounter: {in person / virtual / telephone / other confirmed method}
Pharmacy: {practice-site name and location}
Pharmacist: {name, role, and registration number if configured}
Consultation ID: {SafeScribe consultation ID}
```

Rules:

- Render only fields that have values. Do not render an empty label.
- Use the correct dynamic service type, such as `INITIAL ACCESS PRESCRIBING`, `RENEWAL`, `ADAPTATION`, `ASSESSMENT ONLY`, or `REFERRAL`.
- Include all captured core identifiers designated for the consultation note.
- Include the captured address in both the full note and Kroll-copy header. The label disappears completely when no address is present and the encounter is eligible to proceed without it.
- Do not place direct identifiers inside the AI-generated paragraphs.
- Patient pronouns, sex assigned at birth, phone number, and other profile fields belong in the complete patient record; include them in the note only when clinically relevant to the encounter.
- Include a caregiver or authorized agent when the person's involvement was captured.
- Include consent only when consent was explicitly captured. Do not assume consent from workflow completion.

If the consultation is allowed to remain de-identified, render `Consultation ID` and omit missing patient labels. Do not finalize a prescription-related record when earlier workflow rules require patient identity.

## 9. Hidden DAP Content Model

The generator must internally assign each paragraph a DAP role. The renderer must not display the role.

### 9.1 Data content

Include confirmed, relevant information such as:

- Reason for seeking care and indication.
- Patient's stated goal or expectation, when captured.
- Symptom history: onset, duration, severity, progression, location, associated features, and prior episodes.
- Clinically important positive findings.
- Clinically important negative findings that were explicitly confirmed.
- Relevant red-flag screening results.
- Treatments already tried and response.
- Current medications, non-prescription drugs, and natural health products when relevant.
- Allergies, sensitivities, and reactions when relevant.
- Relevant health conditions and contraindications.
- Pregnancy/lactation, weight, vitals, renal function, or laboratory values only when captured and relevant.
- Adherence, preferences, affordability, access, or other barriers when captured.
- Information sources reviewed, such as patient report, caregiver report, Kroll, or Netcare, when recorded.

Do not render every pathway question. Combine related findings into natural sentences.

### 9.2 Assessment content

Include confirmed pharmacist decisions such as:

- Working assessment or clinical impression.
- Whether the presentation met pathway and treatment-eligibility criteria.
- Severity or classification when recorded.
- Material differential diagnoses considered.
- Referral criteria or red flags identified, or specific relevant criteria confirmed absent.
- Actual or potential drug-therapy problems.
- Relevant allergy, interaction, contraindication, renal, or other safety findings.
- Treatment options considered.
- Treatments declined by the patient or denied by the pharmacist, including confirmed reasons.
- Rationale for the final decision.
- Approved pathway or clinical resource and version used.

Do not upgrade tentative language. If the pharmacist selected `possible`, `suspected`, `consistent with`, or `working assessment`, preserve that degree of certainty. Do not convert it to `confirmed diagnosis`.

### 9.3 Plan content

Include confirmed actions such as:

- Prescription issued, including exact drug, strength, dosage form, dose, route, frequency, duration, quantity, and refills.
- Non-prescription and non-drug recommendations.
- Counselling actually provided.
- Expected benefit and expected time to improvement.
- Relevant adverse effects, precautions, administration, and adherence advice that were confirmed as counselled.
- Safety-net instructions and when additional or urgent care should be sought.
- Monitoring parameters and their rationale.
- Follow-up interval, expected outcome, and person responsible.
- Referral actions.
- Communication with another healthcare professional, including date and method when captured.
- Patient agreement, understanding, preferences, and consent when explicitly captured.

The prescription must appear in the note even when SafeScribe also generates a separate prescription document. The prescription proves the order; the consultation note explains the assessment and rationale.

## 10. Natural Narrative Rules

The output must:

- Use three to five short paragraphs for a typical consultation.
- Follow the order `encounter context -> relevant findings -> assessment and rationale -> care provided -> monitoring and follow-up`.
- Use natural transitions.
- Combine related facts.
- Remove duplicate statements.
- Prefer concise professional sentences.
- Use Canadian spelling.
- Use clear medication terminology and standard units.
- Preserve the pharmacist's documented degree of certainty.
- Remain understandable when pasted as plain text.

The output must not:

- Display DAP or SOAP labels.
- Use Markdown headings, bullets, tables, icons, bold, or decorative separators inside the Kroll-copy body.
- Reproduce raw questions such as `Does the patient have fever?`.
- Repeat the same medication directions in equivalent forms such as `2 g` and `2,000 mg`.
- Use generic filler such as `the patient was counselled appropriately` when specific confirmed counselling exists.
- Use legalistic claims such as `all required information was reviewed` unless every relevant element is traceably confirmed.
- Add stock phrases such as `complete the full course` when they are inapplicable or redundant.
- State that a communication, referral, prescription, or follow-up occurred unless it was captured.

## 11. Exact-Value Protection

The language model must not freely rewrite high-risk values.

Render these fields deterministically or protect them with immutable tokens:

- Patient identifiers.
- Dates and times.
- Drug names.
- Strengths and dosage forms.
- Dose, route, frequency, duration, quantity, and refills.
- Allergy names and reactions.
- Laboratory values, units, and dates.
- Vital signs and units.
- Follow-up dates or intervals.
- Pharmacist and pharmacy identifiers.
- Pathway name and version.

Recommended approach:

1. The server creates an exact text fragment for each protected fact.
2. The model receives placeholders such as `[[RX_1]]`, `[[LAB_1]]`, and `[[FOLLOWUP_1]]`.
3. The model may position the placeholder but must not edit it.
4. The application validates that each required placeholder appears exactly once.
5. The application replaces the placeholder with the deterministic fragment.

Example deterministic fragment:

```text
Prescription issued: {drug} {strength} {dosage form}, {directions}; quantity {quantity}; {refill statement}.
```

Never ask the model to calculate age, eGFR, dose, quantity, duration, or unit conversion.

## 12. Generation Input Contract

Create an immutable consultation snapshot immediately before generation.

```ts
type ConsultationNoteSnapshotV1 = {
  schemaVersion: "consultation-note-source.v1";
  consultationId: string;
  consultationVersion: number;
  createdAt: string;
  jurisdiction: "AB" | string;
  serviceType: string;

  patientHeader: {
    displayName?: string;
    dateOfBirth?: string;
    identifierType?: string;
    identifierValue?: string;
    addressLine?: string;
    caregiverOrAgent?: string;
  };

  encounterHeader: {
    occurredAt: string;
    method?: string;
    pharmacyName?: string;
    pharmacyLocation?: string;
    pharmacistName: string;
    pharmacistRole: string;
    pharmacistRegistrationNumber?: string;
    consentStatus?: "CONFIRMED" | "NOT_CAPTURED";
  };

  facts: Array<{
    factId: string;
    dapRole: "DATA" | "ASSESSMENT" | "PLAN";
    factType: string;
    answerState:
      | "CONFIRMED_PRESENT"
      | "CONFIRMED_ABSENT"
      | "EXPLICITLY_UNAVAILABLE"
      | "NOT_ASKED"
      | "NOT_APPLICABLE";
    normalizedText?: string;
    protectedToken?: string;
    clinicallyRelevant: boolean;
    renderPolicy: "REQUIRED" | "INCLUDE_IF_RELEVANT" | "OMIT";
    source:
      | "PHARMACIST_ENTRY"
      | "CONFIRMED_AI_EXTRACTION"
      | "PATHWAY_RESPONSE"
      | "RULE_ENGINE"
      | "PRESCRIPTION"
      | "COUNSELLING_CONFIRMATION"
      | "FOLLOW_UP_PLAN";
    sourceRecordId: string;
    sourceVersion: number;
  }>;

  pathway?: {
    id: string;
    displayName: string;
    version: string;
    jurisdiction: string;
  };

  protectedFragments: Record<string, string>;
  sourceSnapshotHash: string;
};
```

Before calling the model:

- Remove direct identifiers from `patientHeader` and `encounterHeader`.
- Exclude facts with `NOT_ASKED` or `NOT_APPLICABLE` unless their status is itself explicitly marked clinically material.
- Exclude empty facts and facts marked `OMIT`.
- Convert exact high-risk facts to protected tokens.
- Reject contradictory active facts.
- Reject any token whose deterministic fragment is missing.
- Confirm that every source fact required for the applicable encounter outcome is present and marked `REQUIRED`.

## 13. Model Output Contract

Use Structured Outputs with a strict JSON schema. Do not request unrestricted plain text from the model.

```ts
type GeneratedConsultationNoteV1 = {
  schemaVersion: "consultation-note-output.v1";
  paragraphs: Array<{
    sequence: number;
    dapRole: "DATA" | "ASSESSMENT" | "PLAN";
    text: string;
    sourceFactIds: string[];
    protectedTokens: string[];
  }>;
  omittedFactIds: string[];
  qualityFlags: Array<
    | "SOURCE_CONFLICT"
    | "UNSUPPORTED_INPUT"
    | "MISSING_REQUIRED_TOKEN"
    | "UNABLE_TO_GENERATE"
  >;
};
```

Schema rules:

- Use `additionalProperties: false` on every object.
- Require every schema key. Use empty arrays when there are no items.
- Keep optional source values out of the prompt rather than asking the model to emit `null` prose.
- Reject a response containing a source fact ID or token that was not present in the request.
- Reject a response that omits a non-duplicative fact marked `REQUIRED`.
- Treat a model refusal, incomplete response, parse failure, or non-empty `qualityFlags` as generation failure.
- Do not display partial output as a completed clinical note.

Structured Outputs constrains the response shape, not the clinical truth of the prose. Application-side validation remains mandatory.

## 14. Model Prompt

Store and version this prompt outside application code where possible.

### 14.1 Developer/system instruction

```text
You draft a pharmacist consultation note from a closed set of confirmed
SafeScribe consultation facts.

Your role is limited to organization, selection, deduplication, and clear
clinical narration. You do not assess the patient, make clinical decisions,
calculate values, or add medical knowledge.

Use DAP principles internally, but write the final note as a natural clinical
narrative. Never display D, A, P, SOAP, Data, Assessment, or Plan headings.

Follow this narrative order:
1. reason for care and relevant confirmed history/findings;
2. pharmacist's confirmed clinical impression, eligibility, safety findings,
   options considered, and rationale;
3. confirmed care provided, counselling, monitoring, follow-up, referral, and
   communication.

Use only the supplied facts and protected tokens. Every factual clause must be
supported by one or more supplied fact IDs. Do not use outside clinical
knowledge. Do not infer that an unanswered, omitted, unknown, or not-applicable
item is absent, negative, or normal.

If optional information is absent, omit it silently. Do not leave a blank,
placeholder, question, or generic statement that the information was not
available. Mention unavailable information only when an explicitly supplied
fact says that it was unavailable and clinically material.

Never invent symptoms, negative findings, diagnoses, examination findings,
vitals, laboratory results, medications, allergies, actions, counselling,
consent, communications, referrals, monitoring, or follow-up.

Preserve every protected token exactly. Do not alter, translate, calculate,
expand, abbreviate, or repeat a protected token.

Write concise Canadian clinical English in three to five short paragraphs when
the available facts support that length. Combine related facts and remove
repetition. Do not reproduce raw assessment questions, database labels,
Markdown, bullets, or tables.

Preserve uncertainty exactly. Do not turn "possible", "suspected", or
"consistent with" into a confirmed diagnosis.

Return only the required structured output.
```

### 14.2 Request payload

```text
Create the pharmacist consultation note from CONSULTATION_FACTS below.

CONSULTATION_FACTS:
{de-identified and filtered ConsultationNoteSnapshotV1 facts}

PROTECTED_TOKENS:
{allowed token names only; deterministic values remain server-side}
```

Do not include patient name, PHN, address, phone number, consultation ID, pharmacist registration number, or other direct identifiers in the model request.

## 15. API and Privacy Requirements

- Use the OpenAI Responses API for the generation request unless the current approved SafeScribe integration uses another supported endpoint.
- Use strict Structured Outputs through `text.format` for Responses API implementations.
- Set `store: false` for requests containing consultation data.
- Use the existing approved documentation model route; do not hardcode a model in the UI.
- Use low and consistent generation variability where the selected model/API supports that control.
- Do not give the note generator web, file-search, database, or clinical tool access.
- Do not send raw audio or transcript content to this generation step.
- Do not log request bodies, response bodies, protected fragments, or final note content in routine application logs.
- Logs may contain non-PHI operational metadata such as request ID, prompt version, model route, latency, token usage, success/failure code, and source snapshot hash.
- Encrypt consultation data in transit and at rest according to the application's approved privacy design.
- Apply the SafeScribe retention policy to drafts and generated versions. Kroll remains the permanent patient record unless a separate system-of-record decision is approved.

## 16. Post-Generation Validation

Run deterministic validation before displaying a note.

Required checks:

1. JSON schema parsed successfully.
2. Response completed without refusal or truncation.
3. Paragraph sequence is valid.
4. DAP roles appear internally in the order `DATA -> ASSESSMENT -> PLAN`.
5. No visible DAP/SOAP heading appears in paragraph text.
6. Every cited `sourceFactId` exists in the request.
7. Every used protected token was allowed.
8. Every required protected token appears exactly once.
9. No unknown placeholder remains.
10. No raw assessment question appears.
11. No empty paragraph appears.
12. No duplicate prescription direction or repeated equivalent fact appears.
13. No direct patient identifier appears in AI-generated prose.
14. No statement uses a missing or `NOT_ASKED` fact.
15. The exact deterministic prescription and clinical-value fragments remain unchanged after token replacement.

Recommended additional controls:

- Maintain a list of prohibited unsupported phrases such as `vitals normal`, `labs normal`, `no concerns`, and `physical examination unremarkable` when those facts are not present.
- Compare numeric values and units in final text against the protected-fragment catalog.
- Flag diagnosis-certainty escalation.
- Flag statements such as `patient understands`, `patient agrees`, `consent obtained`, `counselling provided`, or `prescriber notified` unless a matching fact exists.

If validation fails:

- Do not silently remove a potentially unsupported clinical sentence and present the remainder as complete.
- Retry once only when the failure is technical or formatting-related.
- If the second attempt fails, show `Generation failed - create or edit note manually`.
- Preserve the source snapshot and error metadata without logging PHI.

## 17. Rendering the Final Note

The application renders:

1. Deterministic title.
2. Deterministic patient and encounter header.
3. Validated AI-generated narrative paragraphs.
4. Deterministic footer metadata if required.

Kroll copy format:

```text
PHARMACIST CONSULTATION - {SERVICE TYPE}
Patient: {value when present}
DOB: {value when present}
PHN / Patient ID: {value when present}
Address: {value when configured for this output and present}
Date/time: {value}
Encounter: {value when present}
Pharmacy: {value when present}
Pharmacist: {value}
Consultation ID: {value}

{natural narrative paragraph 1}

{natural narrative paragraph 2}

{natural narrative paragraph 3}
```

Rules:

- Use plain text and standard line breaks.
- No Markdown characters.
- No bullets or tables.
- No empty labels.
- No leading spaces or tabs.
- Normalize repeated whitespace.
- Use a maximum of one blank line between blocks.
- Preserve clinically meaningful punctuation and units.
- Copy the currently reviewed version, not an older generated version.

## 18. Example of the Intended Style

This is a style example only. The actual content must come from the consultation snapshot.

```text
PHARMACIST CONSULTATION - INITIAL ACCESS PRESCRIBING
Patient: [captured patient name]
DOB: [captured DOB]
Date/time: [captured date and time]
Encounter: In person
Pharmacist: [captured pharmacist identity]
Consultation ID: [captured consultation ID]

The patient sought assessment for [confirmed presenting concern] that began [confirmed onset/duration]. Relevant associated symptoms and prior treatment response were reviewed. [Specific confirmed negative findings relevant to referral or eligibility.] Relevant medications, health conditions, and allergies were reviewed as recorded.

The presentation was assessed as [pharmacist-confirmed clinical impression] and met the documented pathway and treatment-eligibility criteria. [Confirmed differential, safety finding, treatment options considered, and rationale.] The assessment used [confirmed pathway/resource and version].

[[RX_1]] [Confirmed non-drug advice and counselling provided.] The patient was advised to monitor for [confirmed effectiveness and safety parameters] and to seek further care for [confirmed safety-net instructions]. Follow-up is planned [confirmed interval] with [confirmed responsible person] to assess [confirmed expected outcomes].
```

Do not use the square-bracketed example text in production. It illustrates sequencing only.

## 19. Non-Prescribing Outcomes

The same generator must support:

- Assessment only.
- Supportive care or non-drug care only.
- Non-prescription treatment.
- Referral.
- Urgent referral.
- No treatment required.
- Patient declined treatment.
- Pharmacist declined to prescribe because criteria were not met.
- Treatment deferred pending additional assessment.

Do not force a prescription sentence into these notes. The narrative should document the confirmed decision, rationale, accommodation of the patient's needs, safety-net advice, and follow-up or referral actions.

## 20. Review and Editing Workflow

The initial output status is `REVIEW_REQUIRED`.

Selecting `Review & edit` opens the full plain-text note in an editable clinical-document view.

Required controls:

- Edit note.
- Restore current generated version.
- Save edits.
- Mark reviewed.
- Copy to Kroll.
- Download PDF, when enabled.

To mark reviewed, require explicit pharmacist confirmation:

> I have reviewed this consultation note and confirm that it accurately reflects the assessment, decisions, care provided, and follow-up plan.

Viewing, downloading, or copying the note does not by itself mark it reviewed.

If the pharmacist edits the note after reviewing it:

- Create a new document version.
- Clear the prior review state.
- Return status to `UPDATED_REVIEW_REQUIRED`.

If a clinically relevant upstream value changes:

- Mark the current note `SOURCE_CHANGED`.
- Prevent it from being copied as current.
- Generate a new version from a new source snapshot.
- Require review again.

## 21. Kroll Transfer Confirmation

`Copy to Kroll` copies the reviewed plain-text version to the clipboard.

The application cannot prove that clipboard content was pasted into Kroll. Keep these states separate:

```ts
type RecordTransferStatus =
  | "NOT_COPIED"
  | "COPY_INITIATED"
  | "CONFIRMED_SAVED_TO_PATIENT_RECORD";
```

After copying, display:

> Note copied. Paste it into the correct patient's Kroll profile, then confirm it has been saved to the patient record.

Require an explicit confirmation:

> I confirm that this note has been saved in the correct patient's permanent pharmacy record.

Do not treat `COPY_INITIATED` as proof of record creation. If SafeScribe deletes consultation content under its retention policy, finishing the consultation should be blocked until record transfer is confirmed or an approved alternative record-storage route has succeeded.

## 22. Versioning and Data Model

Do not continue using a SOAP-specific table with `subjective`, `objective`, `assessment`, and `plan` fields as the primary schema for this document. The finalized product decision is a generic consultation note with hidden DAP roles and a natural narrative.

Recommended storage model:

```ts
type ConsultationNoteVersion = {
  id: string;
  consultationId: string;
  documentType: "PHARMACIST_CONSULTATION_NOTE";
  versionNumber: number;
  status:
    | "GENERATING"
    | "REVIEW_REQUIRED"
    | "UPDATED_REVIEW_REQUIRED"
    | "SOURCE_CHANGED"
    | "REVIEWED"
    | "FINALIZED"
    | "GENERATION_FAILED";
  sourceSnapshotHash: string;
  sourceConsultationVersion: number;
  promptVersion: string;
  modelRoute: string;
  generatedContentJson: GeneratedConsultationNoteV1;
  generatedPlainText: string;
  editedPlainText?: string;
  generatedAt: string;
  editedBy?: string;
  editedAt?: string;
  reviewedBy?: string;
  reviewedAt?: string;
  finalizedBy?: string;
  finalizedAt?: string;
  recordTransferStatus: RecordTransferStatus;
  recordTransferConfirmedBy?: string;
  recordTransferConfirmedAt?: string;
};
```

Rules:

- Never update a finalized version in place.
- Preserve the originally generated text and the pharmacist-edited text for the active retention period.
- Record the source snapshot hash, prompt version, model route, and reviewing pharmacist.
- Do not store clinical note content in a general audit-log table.
- If a finalized record must be corrected, create a dated addendum or corrected version while preserving the original according to the approved record policy.

## 23. Suggested Service Interface

```ts
generateConsultationNote(consultationId): Promise<{
  documentId: string;
  versionId: string;
  status: "REVIEW_REQUIRED" | "GENERATION_FAILED";
}>;

updateConsultationNote(versionId, editedPlainText): Promise<{
  newVersionId: string;
  status: "UPDATED_REVIEW_REQUIRED";
}>;

markConsultationNoteReviewed(versionId, pharmacistId): Promise<{
  status: "REVIEWED";
  reviewedAt: string;
}>;

copyConsultationNote(versionId): Promise<{
  plainText: string;
  transferStatus: "COPY_INITIATED";
}>;

confirmSavedToPatientRecord(versionId, pharmacistId): Promise<{
  transferStatus: "CONFIRMED_SAVED_TO_PATIENT_RECORD";
  confirmedAt: string;
}>;
```

Backend authorization must confirm that the user belongs to the consultation's tenant and is permitted to perform each action.

## 24. Failure Messages

Use clear, actionable messages.

### Required upstream information missing

> This consultation is not ready for documentation. Return to the highlighted assessment step and complete the required clinical information.

### Source conflict

> Conflicting consultation information must be resolved before the note can be generated.

### Generation failed

> The consultation note could not be generated. Try again or create the note manually from the confirmed consultation summary.

### Source changed

> Consultation information changed after this note was generated. Generate and review an updated note before copying it to the patient record.

### Not reviewed

> Review and confirm the current note before copying it to Kroll.

## 25. Acceptance Tests

### Content and flow

1. A complete uncomplicated consultation produces a coherent natural narrative with no visible DAP/SOAP labels.
2. The narrative follows findings, assessment/rationale, then plan/follow-up.
3. A typical note contains three to five short paragraphs without filler.
4. Raw assessment questions never appear in the note.
5. Duplicate medication directions are removed.

### Missing data

6. No blood pressure entered: no blood-pressure statement or blank label appears.
7. No laboratory data entered: no laboratory statement or blank label appears.
8. A symptom question was not asked: the note does not state that the symptom was denied.
9. Renal function explicitly recorded as unavailable and clinically material: the precise qualified statement appears.
10. An optional patient identifier is absent: its header label is omitted.

### Non-invention

11. No physical examination recorded: no examination finding is generated.
12. No counselling confirmation recorded: the note does not claim that counselling was provided.
13. No consent confirmation recorded: the note does not claim consent was obtained.
14. No prescriber communication recorded: the note does not claim the prescriber was notified.
15. Tentative assessment recorded: the generated note does not convert it into a confirmed diagnosis.

### Exact values

16. Prescription details in the final note exactly match the prescription record.
17. Laboratory values, units, and dates exactly match source data.
18. Follow-up interval and responsible person exactly match the finalized follow-up plan.
19. Every required protected token appears once and only once.
20. The model never calculates or converts a clinical value.

### Safety and completeness

21. A required pathway response is missing: generation is blocked before the model call.
22. An unresolved allergy-treatment conflict exists: generation is blocked.
23. Contradictory active facts exist: generation is blocked.
24. A referral-only consultation generates an appropriate note without prescription language.
25. A treatment declined by the patient includes the confirmed option, reason, accommodation, and follow-up when captured.

### Review and lifecycle

26. A newly generated note is `REVIEW_REQUIRED`.
27. Previewing, downloading, or copying does not automatically mark the note reviewed.
28. Pharmacist review applies only to the current version.
29. Editing a reviewed note creates a new version and requires review again.
30. Changing a clinically relevant upstream field marks the existing note `SOURCE_CHANGED`.
31. A finalized note cannot be overwritten in place.
32. Copying changes transfer status to `COPY_INITIATED`, not `CONFIRMED_SAVED_TO_PATIENT_RECORD`.
33. Explicit pharmacist confirmation is required to mark the note saved to Kroll.

### Privacy and technical controls

34. Direct patient identifiers are absent from the model request.
35. API requests set `store: false`.
36. Routine logs contain no prompt, response, patient identifier, protected fragment, or note body.
37. Model output that fails schema validation is never displayed as a completed note.
38. Model refusal or truncated output produces a controlled failure state.
39. Kroll copy contains plain text only and preserves paragraph breaks.
40. The source snapshot, prompt version, model route, and pharmacist review metadata are traceable for the active retention period.

## 26. Definition of Done

This feature is complete when:

- The note is assembled only from confirmed SafeScribe consultation data.
- Missing optional information disappears cleanly.
- Missing information is never converted to a negative or normal finding.
- Required missing or conflicting information is blocked upstream.
- DAP organization is preserved internally without visible DAP labels.
- The final note reads naturally and is concise enough for routine Kroll use.
- Exact prescription and clinical values are protected from model rewriting.
- The pharmacist can review, edit, and explicitly approve the current version.
- The reviewed note can be copied as clean plain text.
- Kroll transfer is explicitly confirmed rather than inferred from clipboard use.
- Finalized content is versioned, auditable, and not silently overwritten.
- Privacy controls prevent direct identifiers and note content from entering model storage or routine logs.

## 27. Superseded Requirements

For this document type, these instructions supersede older SafeScribe references to:

- A visible SOAP note.
- Separate `S`, `O`, `A`, and `P` output blocks.
- A SOAP-specific persistence schema as the final consultation-note model.
- Asking for optional missing data during document generation.
- Rendering placeholders such as `not provided` or `not available` for every empty field.

The current requirement is a pharmacist consultation note that is DAP-informed internally, natural in external presentation, source-bound, non-inventive, pharmacist-reviewed, and optimized for Kroll copy.
