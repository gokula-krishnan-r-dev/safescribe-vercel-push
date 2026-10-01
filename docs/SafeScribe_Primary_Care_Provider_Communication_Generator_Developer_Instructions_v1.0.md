# SafeScribe Primary Care Provider Communication Generator - Developer Instructions

**Document:** v1.0  
**Module:** Consultation Documents - Primary Care Provider Communication  
**Primary use:** Pharmacist-reviewed communication to a physician, nurse practitioner, or other regulated health professional whose care may be affected  
**Jurisdictional baseline:** Alberta  
**Status:** Implementation specification

## 1. Objective

Implement a Primary Care Provider Communication generator that automatically converts the completed SafeScribe consultation into a concise, provider-facing clinical communication.

The communication must:

1. Identify the patient, recipient, pharmacist, pharmacy, consultation, and communication purpose.
2. Summarize only the encounter information needed for continuity of care.
3. State the pharmacist's confirmed assessment or disposition without overstating diagnostic certainty.
4. Clearly describe the prescribing decision, treatment, rationale, patient instructions, monitoring, follow-up, and any requested provider action.
5. Use only confirmed information from the completed SafeScribe workflow.
6. Silently omit optional information that was not captured.
7. Never invent, infer, or normalize missing clinical or administrative facts.
8. Protect exact medication, laboratory, vital-sign, date, and follow-up values from language-model rewriting.
9. Remain editable and require explicit pharmacist review.
10. Keep document review separate from transmission and delivery status.
11. Produce a clean PDF and a plain-text copy suitable for an approved secure communication workflow.

This feature does not conduct a new assessment, choose a recipient, make a prescribing decision, or send the communication autonomously. It documents and communicates decisions already made by the pharmacist.

## 2. Controlling Product Decision

The PCP communication is not a duplicate of the Pharmacist Consultation Note.

The consultation note is the detailed pharmacy record. The PCP communication is a shorter clinical handoff designed so the recipient can quickly determine:

- Who the communication concerns.
- Why it was sent.
- What the pharmacist assessed and concluded.
- What medication or other care was provided.
- Why the decision was made.
- What the patient was told.
- What will be monitored and by whom.
- Whether the recipient must take any action.

The output should normally fit on one page. A typical uncomplicated communication should usually contain approximately 150-350 words, excluding the deterministic header, medication block, and footer. This is a target, not a hard limit. Do not add filler to reach it or remove clinically important information to stay within it.

Use a semi-structured professional format with short narrative paragraphs and clear clinical labels. Unlike the Kroll consultation note, visible labels are appropriate because the recipient must scan the communication quickly.

Recommended visible structure:

1. Deterministic sender, recipient, patient, and date header.
2. Prominent communication purpose and action status.
3. Brief encounter and assessment summary.
4. Deterministic care or prescription block.
5. Concise rationale.
6. Patient instructions, monitoring, and follow-up.
7. Explicit provider action or `For information only` statement.
8. Deterministic pharmacist signature and contact block.

## 3. Regulatory Purpose

ACP Standard 3.3.2(c) requires a pharmacist, when adapting, prescribing in an emergency, prescribing at initial access, or managing ongoing therapy of a Schedule 1 drug, to communicate as soon as reasonably possible with regulated health professionals whose care may be affected by the prescribing decision.

For those prescribing decisions, the communication must include:

1. The type and amount of drug prescribed.
2. The rationale for prescribing the drug.
3. The date the drug was prescribed.
4. Any non-pharmaceutical recommendations associated with the prescription.
5. The monitoring plan.
6. Instructions given to the patient.

ACP Appendix E also requires the patient record to capture the date and method of notification for pharmacist adaptations and pharmacist prescribing or deprescribing.

Product implications:

- These elements are deterministic readiness requirements for applicable Alberta prescribing communications.
- The model must not be asked to decide whether the regulatory communication requirement applies.
- The application must determine the requirement from the service type, prescribing action, jurisdictional configuration, affected-provider decision, and documented exceptions.
- Generating a document does not prove communication occurred.
- Downloading, printing, or copying does not prove communication occurred.
- Transmission date and method belong in a separate communication-attempt record.
- A successful send does not prove that the recipient reviewed or acknowledged the communication.

Official references:

- [ACP Standards of Practice for Pharmacists and Pharmacy Technicians - Standards 3.3.1, 3.3.2, 7.9 and Appendix E](https://abpharmacy.ca/wp-content/uploads/Standards_SPPPT.pdf)
- [OpenAI Structured Outputs guidance](https://developers.openai.com/api/docs/guides/structured-outputs)
- [OpenAI Responses API migration and storage guidance](https://developers.openai.com/api/docs/guides/migrate-to-responses)
- [OpenAI API data controls](https://developers.openai.com/api/docs/guides/your-data)

## 4. Terminology

Use `Primary Care Provider Communication` as the SafeScribe user-facing document name for the current module.

Do not assume every recipient is a family physician. The actual document must use the recipient's confirmed professional title and name, for example:

- Physician
- Nurse practitioner
- Specialist
- Pharmacist
- Other regulated health professional

Internally, prefer the broader type name:

```ts
type DocumentType = "REGULATED_HEALTH_PROFESSIONAL_COMMUNICATION";
```

The first product implementation may retain:

```ts
type DocumentType = "PRIMARY_CARE_PROVIDER_COMMUNICATION";
```

if changing the shared manifest is outside this ticket. Do not let the internal name cause the generated content to call an unverified recipient the patient's PCP.

Use `pharmacist` or `prescribing pharmacist` for the sender. Do not use `prescriber` alone where it could be mistaken for the receiving clinician.

## 5. Scope

This ticket includes:

- Determining the communication template from the confirmed communication purpose.
- Projecting source data from the completed consultation.
- Required-versus-optional source validation.
- Missing-information behaviour.
- Recipient-specific document generation.
- AI prompt and strict structured-output contract.
- Deterministic rendering of identifiers and exact clinical values.
- One-page PDF and plain-text rendering.
- Review and editing.
- Versioning and upstream-change invalidation.
- Transmission-state recording.
- Communication-attempt audit metadata.
- Privacy, quality controls, and acceptance tests.

This ticket does not include:

- Selecting or guessing the correct recipient.
- Searching for providers without an approved source and workflow.
- Reopening the clinical assessment to collect optional data.
- Asking new clinical questions during document generation.
- Making a diagnosis, treatment, referral, or monitoring decision.
- Re-running medication-safety rules inside the language model.
- Autonomous fax, email, or secure-message transmission.
- Treating ordinary email or text messaging as an approved secure channel.
- Direct integration with a provincial EHR, EMR, fax vendor, or Kroll unless separately approved.
- Confirming that the recipient read, understood, or accepted the communication.

## 6. Communication-Purpose Model

Every communication must have exactly one primary purpose selected or derived before generation.

```ts
type CommunicationPurpose =
  | "INITIAL_ACCESS_PRESCRIBING_NOTIFICATION"
  | "ONGOING_THERAPY_PRESCRIBING_NOTIFICATION"
  | "RENEWAL_NOTIFICATION"
  | "ADAPTATION_NOTIFICATION"
  | "EMERGENCY_PRESCRIBING_NOTIFICATION"
  | "DEPRESCRIBING_NOTIFICATION"
  | "REFERRAL"
  | "URGENT_REFERRAL"
  | "ASSESSMENT_UPDATE"
  | "CARE_COORDINATION"
  | "OTHER_CONFIRMED_PURPOSE";
```

Every communication must also have an action expectation:

```ts
type RecipientActionExpectation =
  | "FOR_INFORMATION_ONLY"
  | "ACTION_REQUESTED"
  | "URGENT_ACTION_REQUESTED";
```

Rules:

- The application determines these values from pharmacist-confirmed workflow state.
- The model must not choose or change them.
- `FOR_INFORMATION_ONLY` must visibly state that no response is requested unless the pharmacist documented a conditional request.
- `ACTION_REQUESTED` must include a specific confirmed action and requested timing when captured.
- `URGENT_ACTION_REQUESTED` must not rely on a generated letter alone. The workflow must require a real-time or otherwise appropriate urgent handoff process and record the communication attempt.
- Do not use `URGENT` merely to make the document more noticeable.

## 7. Communication-Requirement Decision

The application, not the model, must determine whether communication is required, optional, not required, or unresolved.

```ts
type CommunicationRequirementStatus =
  | "REQUIRED"
  | "OPTIONAL"
  | "NOT_REQUIRED"
  | "PENDING_DETERMINATION";
```

The decision should be produced by a versioned jurisdictional rule, using at least:

- Jurisdiction.
- Service type.
- Whether a Schedule 1 drug was initiated, renewed, adapted, deprescribed, or prescribed in an emergency.
- Whether another regulated health professional's care may be affected.
- Applicable documented exception.
- Referral or care-coordination outcome.

Do not infer that a PCP is affected solely because one exists in the patient profile. Conversely, do not suppress required communication solely because no PCP is stored.

If `PENDING_DETERMINATION`:

- Block final communication generation.
- Return the pharmacist to the communication decision step.
- Do not ask the model to resolve the requirement.

If no affected provider is identified after an explicit pharmacist decision:

- Record the structured status and pharmacist decision in the consultation record.
- Do not create a generic letter pretending it was sent to a PCP.
- Apply the approved Alberta operational policy for completing the consultation and any follow-up task.

## 8. Recipient Requirements

The recipient must be selected and verified before the communication can be marked ready for transmission.

```ts
type RecipientSnapshot = {
  recipientId?: string;
  status: "VERIFIED" | "UNVERIFIED" | "NONE_IDENTIFIED";
  displayName?: string;
  professionalTitle?: string;
  clinicOrOrganization?: string;
  secureFax?: string;
  secureMessagingAddress?: string;
  mailingAddress?: string;
  verificationSource?: string;
  verifiedBy?: string;
  verifiedAt?: string;
};
```

Rules:

- Never infer the recipient from a first name, last encounter, clinic name, or free-text note.
- Do not call the recipient `the patient's PCP` unless that relationship is confirmed.
- Do not place an unverified fax number or address on a transmission-ready document.
- If the recipient is unverified, a draft may be generated for pharmacist review only, but transmission and completion remain blocked when communication is required.
- If the recipient is `NONE_IDENTIFIED`, do not render empty `To`, clinic, or fax labels.
- Generate a separate document version for each recipient when more than one provider must be informed.
- Do not place multiple unrelated recipients in one `To` field.
- Record which recipient received which document version.

Recipient verification and clinical-document review are different controls. Reviewing the content does not verify the recipient.

## 9. Source-of-Truth Rule

The generator may select, organize, deduplicate, and narrate confirmed consultation facts. It must not introduce new clinical or administrative facts.

Permitted sources are:

1. Pharmacist-entered or pharmacist-confirmed structured fields.
2. Final pathway responses.
3. Deterministic eligibility, red-flag, and medication-safety results.
4. Pharmacist-confirmed clinical impression and rationale.
5. Final prescription, adaptation, renewal, deprescribing, or care record.
6. Counselling and patient instructions confirmed as provided.
7. Final monitoring and follow-up plan.
8. Confirmed non-drug recommendations.
9. Confirmed referral and requested-provider-action fields.
10. Patient, recipient, pharmacist, pharmacy, and encounter metadata.
11. AI-extracted facts only after pharmacist confirmation.

Do not send the raw consultation transcript to the communication generator. Use the finalized structured consultation snapshot.

When duplicate or conflicting values exist, use this precedence:

1. Most recent pharmacist edit.
2. Explicit final workflow selection.
3. Deterministic rule-engine result.
4. Confirmed AI extraction.
5. Earlier structured value.

If a clinically or operationally material conflict remains unresolved, block generation. Do not ask the model to choose a value.

## 10. Upstream Validation Versus Document Generation

Mandatory data must be enforced before the final document is marked ready.

### 10.1 Requirements for every communication

- Patient full legal name.
- At least one reliable secondary patient identifier: DOB or approved patient identifier. Prefer DOB and PHN when both are captured.
- Encounter date.
- Communication purpose.
- Pharmacist identity.
- Pharmacy identity and return contact information.
- Pharmacist-confirmed clinical conclusion or disposition.
- Confirmed recipient-action expectation.
- Source consultation version.

### 10.2 Additional requirements for prescribing notifications

- Exact prescribed drug and amount.
- Exact prescribing date.
- Confirmed prescribing rationale.
- Associated non-drug recommendations, when any were made.
- Monitoring plan.
- Instructions actually given to the patient.
- Relevant affected-provider determination.

If no non-drug recommendation was made, do not invent one. Represent its structured state as `NONE_CONFIRMED`, not as missing data.

If no patient instruction was given in a situation where instructions were required upstream, block earlier. Do not have the generator create counselling after the fact.

### 10.3 Additional requirements for referral

- Reason for referral.
- Urgency.
- Relevant supporting findings.
- Specific requested action, when applicable.
- Patient safety-net direction.
- Responsibility for follow-up.

### 10.4 Source-state behaviour

| Source state | Behaviour |
| --- | --- |
| Required clinical element missing | Block before model call and return to the relevant workflow step |
| Required recipient not verified | Permit draft review only; block transmission and required-communication completion |
| Optional value missing or `null` | Omit completely |
| Field not asked | Omit; never convert to a negative |
| Explicitly not applicable | Omit unless the status is materially important |
| Explicitly unavailable and material | Include only the pharmacist-confirmed qualification |
| Confirmed none | Render only when needed to satisfy an explicit communication element |
| Unresolved contradiction | Block generation |

The document stage must not present a new clinical questionnaire.

## 11. Missing-Information Rules

These rules are non-negotiable.

### 11.1 Optional missing information

If optional data is absent, silently omit it.

For example, if no blood pressure or laboratory value was entered:

- Do not leave a blank field.
- Do not show a placeholder.
- Do not ask for the value.
- Do not state `not available` by default.
- Do not imply that it was normal or reviewed.

### 11.2 Missing is not negative

```ts
type AnswerState =
  | "CONFIRMED_PRESENT"
  | "CONFIRMED_ABSENT"
  | "CONFIRMED_NONE"
  | "EXPLICITLY_UNAVAILABLE"
  | "NOT_ASKED"
  | "NOT_APPLICABLE";
```

`NOT_ASKED`, `NOT_APPLICABLE`, an empty string, and `null` must never render as `denies`, `none`, `normal`, `negative`, `no concerns`, or `not applicable`.

### 11.3 Explicitly unavailable information

Mention unavailable information only when:

1. The pharmacist explicitly recorded it as unavailable; and
2. Its absence materially affected or qualified the decision; and
3. The pharmacist supplied the decision or qualification that should accompany it.

The model must not create the qualification.

### 11.4 No invented objective findings or actions

Never generate:

- Examination findings that were not performed and recorded.
- Vitals, laboratory values, calculated values, or dates not present in source data.
- Statements that records, labs, or medication histories were reviewed unless recorded.
- Claims that counselling, consent, a referral, follow-up, or communication occurred unless confirmed.
- A requested PCP action that the pharmacist did not select.
- A recipient, clinic, fax number, or provider relationship from context clues.

## 12. Document Header

Render the header deterministically. Do not ask the model to write it.

Recommended PDF/plain-text header order:

```text
PRIMARY CARE PROVIDER COMMUNICATION
Purpose: {display purpose}
Action: {For information only / Action requested / Urgent action requested}

To: {recipient title and full name}
Clinic/organization: {value when present}
Secure fax/address: {value when present and verified}

Re: {patient full legal name}
DOB: {YYYY-MM-DD}
PHN / Patient ID: {value when present}

Encounter date: {local date}
Communication created: {local date and time}
SafeScribe consultation ID: {value}
```

Rules:

- The PDF should use `Re:` for patient identification, not place the patient name in the greeting.
- Use at least patient name plus DOB or approved patient identifier before transmission.
- Include both DOB and PHN when captured and appropriate for the approved communication channel.
- Patient address is not required in the communication unless the configured use case needs it. Do not include it by default.
- Render only labels with values.
- Never include an unverified destination in a transmission-ready version.
- Distinguish encounter date, document-created date, prescribing date, and transmission date. Do not substitute one for another.
- Transmission date is not available until a communication attempt occurs and does not belong in the initial generated content.

## 13. Content Structure

### 13.1 Purpose and action banner

The purpose and action expectation must be visible without reading the body.

Examples:

```text
Purpose: Pharmacist initial-access prescribing notification
Action: For information only
```

```text
Purpose: Referral following pharmacist assessment
Action: Assessment requested within the documented timeframe
```

The application renders this block from structured values. The model may not rewrite it.

### 13.2 Encounter and assessment summary

Use one concise paragraph containing only relevant confirmed information:

- Reason for the encounter.
- Relevant history and findings needed to understand the decision.
- Material positive and explicitly confirmed negative findings.
- Pharmacist's working assessment or disposition.
- Eligibility, severity, red flags, or differentials only when material.
- Relevant safety findings.

Do not reproduce the full pathway or every screening question. The recipient can request the detailed pharmacy record if needed.

### 13.3 Care provided or prescribing decision

Render exact care details deterministically.

For a prescription:

```text
PRESCRIBING DECISION
Date prescribed: {date}
Medication: {drug, strength, dosage form}
Directions: {dose, route, frequency, duration}
Quantity: {value}
Refills: {value}
```

For an adaptation:

```text
ADAPTATION
Date adapted: {date}
Original therapy: {exact confirmed record}
Adapted therapy: {exact confirmed record}
Nature of adaptation: {confirmed type}
```

For a referral or assessment-only outcome, do not render prescription labels.

### 13.4 Rationale

Use a concise paragraph or short labelled statement explaining the pharmacist-confirmed reason for the decision.

Include only rationale already confirmed in the workflow. The model may improve readability but may not add clinical evidence, guideline claims, comparative benefits, or risk statements that were not provided.

Preserve the degree of certainty. Do not convert `consistent with`, `suspected`, or `working assessment` into a confirmed diagnosis.

### 13.5 Non-drug recommendations and patient instructions

Include:

- Confirmed non-pharmaceutical recommendations associated with the prescription.
- Confirmed medication administration instructions.
- Confirmed counselling about important adverse effects or precautions.
- Confirmed safety-net directions.

Do not include every patient-handout detail. Select only items relevant to coordination and safe follow-up.

If the structured state is `CONFIRMED_NONE` for non-drug recommendations, the Alberta prescribing template may render:

```text
No associated non-drug recommendations were made.
```

Use this only when needed to demonstrate that the required element was considered. Do not infer it from an empty list.

### 13.6 Monitoring and follow-up

Include:

- What effectiveness and safety parameters will be monitored.
- Timing or interval.
- Expected outcome, when confirmed.
- Person responsible for follow-up.
- Escalation or referral threshold, when confirmed.

Use exact protected fragments for dates, intervals, measurements, and responsible-person labels.

Do not imply shared responsibility. If the pharmacist remains responsible for follow-up, state that. If the PCP is asked to assume or coordinate a task, state the specific confirmed request.

### 13.7 Provider action

End with exactly one deterministic action block.

For information only:

```text
PROVIDER ACTION
For information and continuity of care. No response is requested.
```

For action requested:

```text
PROVIDER ACTION
{specific confirmed request and timing}
```

For urgent action requested:

```text
URGENT PROVIDER ACTION
{specific confirmed urgent request}
```

Do not generate vague requests such as `Please assess as appropriate`, `Please advise`, or `Please follow up` unless that exact level of discretion was deliberately selected and confirmed by the pharmacist.

Do not state that the PCP agreed, accepted responsibility, or will follow up unless that response was actually received and separately recorded.

## 14. Sender and Signature Block

Render the sender block deterministically:

```text
{pharmacist full name}, {credentials/role}
ACP registration number: {value when configured}
{pharmacy name}
{pharmacy address}
Telephone: {value}
Secure fax: {value when present}
```

Rules:

- Use the pharmacist who made or approved the clinical decision.
- If another pharmacist prepared the communication, preserve authorship and reviewer roles separately in metadata.
- Do not let the model generate credentials, registration numbers, contact details, or signatures.
- A typed signature block does not replace any separate authentication requirement for the selected transmission method.

## 15. Natural Language Rules

The provider-facing narrative must:

- Use concise Canadian clinical English.
- Use one to three short paragraphs for a typical communication.
- Follow the order `encounter and assessment -> rationale -> relevant instructions and follow-up`.
- Use objective, professional wording.
- Combine related facts and remove repetition.
- Preserve documented uncertainty.
- Clearly distinguish patient-reported information from pharmacist-observed or record-derived information when that distinction is material.
- Remain understandable in both PDF and plain text.

The narrative must not:

- Reproduce the entire Pharmacist Consultation Note.
- Display DAP or SOAP headings.
- Reproduce raw assessment questions.
- Use Markdown, decorative icons, or conversational language.
- Repeat deterministic prescription directions in prose.
- Add generic filler such as `the patient was counselled appropriately`.
- Include a long patient-education handout.
- Introduce new diagnoses, evidence, recommendations, or provider requests.
- Say `all labs were normal`, `no concerns`, or similar unsupported conclusions.
- Claim the provider was notified, received the document, acknowledged it, or accepted the plan.

## 16. Exact-Value Protection

The language model must not freely rewrite high-risk or operational values.

Render these values deterministically or protect them with immutable tokens:

- Patient identifiers.
- Recipient identity and destination.
- Pharmacist and pharmacy identity.
- Encounter, prescribing, adaptation, document, and follow-up dates.
- Drug names, strengths, and dosage forms.
- Dose, route, frequency, duration, quantity, and refills.
- Original and adapted prescription details.
- Allergy names and reactions.
- Laboratory values, units, and dates.
- Vital signs and units.
- Monitoring intervals and responsible person.
- Requested action and requested timing.
- Communication purpose and action expectation.
- Pathway name and version, if included.

Recommended process:

1. Server creates exact deterministic fragments.
2. Model receives placeholders such as `[[RX_1]]`, `[[MONITORING_1]]`, and `[[ACTION_1]]`.
3. Model may position an allowed placeholder but may not edit it.
4. Application validates required placeholder presence and uniqueness.
5. Application replaces placeholders with exact fragments.

Never ask the model to calculate age, eGFR, dose, amount, quantity, duration, date, or unit conversion.

## 17. Generation Input Contract

Create an immutable communication snapshot immediately before generation.

```ts
type ProviderCommunicationSnapshotV1 = {
  schemaVersion: "provider-communication-source.v1";
  consultationId: string;
  consultationVersion: number;
  sourceSnapshotHash: string;
  createdAt: string;
  jurisdiction: "AB" | string;
  serviceType: string;

  requirement: {
    status: CommunicationRequirementStatus;
    ruleId: string;
    ruleVersion: string;
    affectedProviderDecision: string;
    decidedBy: string;
    decidedAt: string;
  };

  purpose: CommunicationPurpose;
  actionExpectation: RecipientActionExpectation;
  requestedActionToken?: string;

  patientHeader: {
    displayName: string;
    dateOfBirth?: string;
    identifierType?: string;
    identifierValue?: string;
  };

  recipient: RecipientSnapshot;

  senderHeader: {
    pharmacistName: string;
    pharmacistRole: string;
    pharmacistRegistrationNumber?: string;
    pharmacyName: string;
    pharmacyAddress?: string;
    pharmacyTelephone: string;
    pharmacySecureFax?: string;
  };

  encounter: {
    occurredAt: string;
    method?: string;
  };

  facts: Array<{
    factId: string;
    communicationRole:
      | "ENCOUNTER"
      | "ASSESSMENT"
      | "RATIONALE"
      | "CARE_PROVIDED"
      | "PATIENT_INSTRUCTION"
      | "MONITORING"
      | "FOLLOW_UP"
      | "PROVIDER_ACTION";
    factType: string;
    answerState: AnswerState;
    normalizedText?: string;
    protectedToken?: string;
    providerRelevant: boolean;
    renderPolicy: "REQUIRED" | "INCLUDE_IF_RELEVANT" | "OMIT";
    source:
      | "PHARMACIST_ENTRY"
      | "CONFIRMED_AI_EXTRACTION"
      | "PATHWAY_RESPONSE"
      | "RULE_ENGINE"
      | "PRESCRIPTION"
      | "ADAPTATION"
      | "COUNSELLING_CONFIRMATION"
      | "MONITORING_PLAN"
      | "FOLLOW_UP_PLAN"
      | "REFERRAL_PLAN";
    sourceRecordId: string;
    sourceVersion: number;
  }>;

  protectedFragments: Record<string, string>;
};
```

Before calling the model:

- Confirm requirement status is not `PENDING_DETERMINATION`.
- Confirm the service-specific required facts exist.
- Reject unresolved clinical conflicts.
- Exclude `NOT_ASKED`, `NOT_APPLICABLE`, empty, and `OMIT` facts.
- Exclude direct patient, recipient, pharmacist, and pharmacy identifiers from the model request.
- Convert exact clinical and operational values to protected tokens.
- Reject tokens without deterministic fragments.
- De-duplicate facts already represented in deterministic blocks.
- Send only provider-relevant facts.

## 18. Model Output Contract

Use strict Structured Outputs. Do not request unrestricted letter text.

```ts
type GeneratedProviderCommunicationV1 = {
  schemaVersion: "provider-communication-output.v1";
  assessmentSummary: {
    text: string;
    sourceFactIds: string[];
    protectedTokens: string[];
  };
  rationaleSummary: {
    text: string;
    sourceFactIds: string[];
    protectedTokens: string[];
  };
  continuitySummary: {
    text: string;
    sourceFactIds: string[];
    protectedTokens: string[];
  };
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

- Use `additionalProperties: false` for every object.
- Require every key. Use an empty string or empty array only where the schema explicitly allows it.
- Omit absent optional source facts from the request rather than asking the model to narrate `null`.
- Reject unknown source fact IDs or protected tokens.
- Reject omission of a non-duplicative `REQUIRED` fact.
- Reject unsupported direct identifiers in model-generated text.
- Treat refusal, incomplete response, parse failure, or non-empty `qualityFlags` as generation failure.
- Do not display partial output as a completed communication.

Structured Outputs constrains response shape; application validation must still establish source support and exact-value integrity.

## 19. Model Prompt

Store and version the prompt outside UI code where possible.

### 19.1 Developer/system instruction

```text
You draft the narrative portions of a clinical communication from a pharmacist
to another regulated health professional using a closed set of confirmed
SafeScribe facts.

Your role is limited to selection, organization, deduplication, and concise
clinical narration. You do not assess the patient, make clinical decisions,
calculate values, select a recipient, decide whether communication is required,
or add medical knowledge.

Write concise Canadian clinical English for a busy healthcare professional.
The application separately renders the sender, recipient, patient identifiers,
communication purpose, action status, exact care or prescription block,
provider-action block, and signature. Do not reproduce those elements unless an
allowed protected token is explicitly supplied for placement.

Use only supplied facts and protected tokens. Every factual clause must be
supported by one or more supplied fact IDs. Do not infer that an unanswered,
omitted, unknown, or not-applicable item is absent, negative, normal, or
unavailable.

If optional information is absent, omit it silently. Do not leave blanks,
questions, placeholders, or generic missing-information statements. Mention
unavailable information only when a supplied fact explicitly says it was
unavailable and clinically material.

Never invent symptoms, findings, diagnoses, examination results, vitals,
laboratory results, medications, allergies, rationale, recommendations,
counselling, monitoring, follow-up, referrals, requested actions, consent,
recipient identity, communication attempts, delivery, or acknowledgement.

Preserve uncertainty exactly. Do not turn "possible", "suspected", "working
assessment", or "consistent with" into a confirmed diagnosis.

Preserve every protected token exactly. Do not alter, translate, calculate,
expand, abbreviate, or repeat a protected token.

Write:
1. a brief assessment summary containing only provider-relevant facts;
2. a concise pharmacist-confirmed rationale;
3. a concise continuity summary containing only confirmed patient instructions,
   monitoring, follow-up, or care-coordination details not already rendered in
   deterministic blocks.

Avoid repetition, raw assessment questions, database labels, DAP/SOAP headings,
Markdown, greetings, signatures, and filler. Do not claim the recipient was
notified, received the communication, agreed with the plan, or accepted
follow-up responsibility.

Return only the required structured output.
```

### 19.2 Request payload

```text
Create the provider-facing narrative from COMMUNICATION_FACTS below.

COMMUNICATION_FACTS:
{de-identified and filtered ProviderCommunicationSnapshotV1 facts}

PROTECTED_TOKENS:
{allowed token names only; deterministic values remain server-side}
```

Do not include patient name, DOB, PHN, address, recipient identity, clinic, destination, pharmacist identity, registration number, pharmacy contact information, or consultation ID in the model request.

## 20. API and Privacy Requirements

- Use the OpenAI Responses API unless the approved SafeScribe integration uses another supported endpoint.
- Use strict Structured Outputs through `text.format` for Responses API implementations.
- Set `store: false` for every request containing consultation data.
- Use the approved documentation model route; do not hardcode a model in the UI.
- Use low and consistent generation variability where supported.
- Do not give the generator web, file-search, database, or clinical-tool access.
- Do not send raw audio or transcript content.
- Do not use a multi-turn model conversation for this document. Each version should be generated from its immutable snapshot.
- Do not log model request bodies, response bodies, protected fragments, or final communication content in routine logs.
- Logs may contain non-PHI operational metadata such as request ID, prompt version, model route, latency, token use, success/failure code, and source snapshot hash.
- Encrypt consultation and communication data in transit and at rest according to the approved privacy design.
- Apply the SafeScribe retention policy to drafts and versions.
- Do not use live web search for clinical content generation.

## 21. Post-Generation Validation

Run deterministic validation before displaying the communication.

Required checks:

1. Strict JSON schema parsed successfully.
2. Response completed without refusal or truncation.
3. Every cited source fact exists in the request.
4. Every used protected token was allowed.
5. Every required protected token appears exactly once in the assembled document.
6. No unknown placeholder remains.
7. No direct patient, recipient, pharmacist, or pharmacy identifier appears in model-generated text.
8. No source fact marked `NOT_ASKED`, `NOT_APPLICABLE`, or `OMIT` supports a sentence.
9. No raw assessment question appears.
10. No empty visible section or label appears.
11. No duplicate prescription direction or equivalent fact appears.
12. Exact prescription, adaptation, laboratory, vital, date, monitoring, and action fragments remain unchanged.
13. Diagnostic certainty is not escalated.
14. No unsupported claim of counselling, consent, notification, transmission, delivery, acknowledgement, or provider agreement appears.
15. Required Alberta prescribing elements are present for the applicable purpose.
16. `ACTION_REQUESTED` contains a specific confirmed action.
17. `FOR_INFORMATION_ONLY` does not contain a contradictory request.
18. Recipient status and document readiness are consistent.

If validation fails:

- Do not silently remove a potentially unsupported sentence and present the remainder as complete.
- Retry once only for technical or formatting failure.
- If the retry fails, show `Generation failed - create or edit the communication manually`.
- Preserve non-PHI error metadata and the source snapshot hash.

## 22. Final Document Rendering

Assemble the document in this order:

1. Deterministic title.
2. Deterministic purpose and action banner.
3. Deterministic recipient and patient header.
4. Validated assessment summary.
5. Deterministic care or prescription block.
6. Validated rationale summary.
7. Deterministic non-drug, instruction, monitoring, and follow-up blocks where exact content is required.
8. Validated continuity summary for relevant non-duplicative prose.
9. Deterministic provider-action block.
10. Deterministic pharmacist signature and contact block.
11. Optional deterministic confidentiality footer approved by privacy/legal review.

Plain-text output example:

```text
PRIMARY CARE PROVIDER COMMUNICATION
Purpose: {value}
Action: {value}

To: {verified recipient}
Clinic/organization: {value when present}

Re: {patient full legal name}
DOB: {value}
PHN / Patient ID: {value when present}
Encounter date: {value}
SafeScribe consultation ID: {value}

CLINICAL SUMMARY
{validated concise assessment summary}

{deterministic prescription, adaptation, care, or referral block}

RATIONALE
{validated concise rationale}

PATIENT INSTRUCTIONS, MONITORING AND FOLLOW-UP
{deterministic confirmed content plus validated non-duplicative continuity text}

PROVIDER ACTION
{deterministic action statement}

{deterministic pharmacist and pharmacy contact block}
```

Rendering rules:

- No empty labels or sections.
- No placeholder text in a final document.
- No decorative icons in the printable document.
- Use plain-language section labels and professional typography.
- Preserve exact units, punctuation, and medication directions.
- Normalize repeated whitespace.
- Use one blank line between plain-text sections.
- Keep the PDF legible in grayscale.
- Avoid splitting the prescription or provider-action block across pages.
- Display `DRAFT - NOT YET REVIEWED` on unreviewed PDF versions.
- Remove the draft watermark only after current-version pharmacist review.

## 23. Example of Intended Style

This is a style example only. Square-bracketed text must never appear in production.

```text
PRIMARY CARE PROVIDER COMMUNICATION
Purpose: Pharmacist initial-access prescribing notification
Action: For information only

To: [verified recipient]
Re: [captured patient name]
DOB: [captured DOB]
Encounter date: [captured encounter date]

CLINICAL SUMMARY
The patient was assessed for [confirmed reason for care]. The reported history and relevant confirmed findings were consistent with [pharmacist-confirmed working assessment]. [Only provider-relevant eligibility or safety findings.]

PRESCRIBING DECISION
[[RX_1]]
Date prescribed: [[PRESCRIBING_DATE_1]]

RATIONALE
[Concise pharmacist-confirmed rationale supported by source facts.]

PATIENT INSTRUCTIONS, MONITORING AND FOLLOW-UP
[[INSTRUCTIONS_1]] [[MONITORING_1]] [[FOLLOWUP_1]]

PROVIDER ACTION
For information and continuity of care. No response is requested.

[deterministic pharmacist and pharmacy contact block]
```

## 24. Non-Prescribing Communications

Support communications where no prescription was issued:

- Referral.
- Urgent referral.
- Assessment update.
- Care-coordination request.
- Treatment declined.
- Pharmacist declined to prescribe.
- Supportive care or non-prescription treatment only.

Rules:

- Do not force prescription headings into these documents.
- Clearly state the confirmed disposition.
- Include only findings needed to understand the referral or update.
- For a referral, make the requested action and urgency explicit.
- For urgent referrals, require an appropriate active handoff workflow; the document is supporting information, not the whole urgent action.
- Do not claim that a referral was accepted or booked unless confirmed.

## 25. Review and Editing Workflow

The initial document status is `REVIEW_REQUIRED`.

`Review & edit` opens the assembled provider communication in an editable clinical-document view.

Required controls:

- Edit the communication.
- Restore the current generated version.
- Save edits as a new version.
- Mark the current version reviewed.
- Copy communication.
- Download PDF.
- Record or initiate an approved transmission action when configured.

Require explicit pharmacist confirmation before review:

> I have reviewed this communication and confirm that it accurately identifies the patient and recipient, reflects the assessment and care provided, and clearly states the monitoring, follow-up, and requested provider action.

Viewing, copying, downloading, or printing does not mark the document reviewed.

Editing after review must:

- Create a new document version.
- Clear the prior review state.
- Return the current version to `UPDATED_REVIEW_REQUIRED`.

Changing a clinically relevant upstream value must:

- Mark the current version `SOURCE_CHANGED`.
- Prevent the outdated version from being transmitted as current.
- Generate a new version from a new source snapshot.
- Require review again.

Changing recipient or destination must:

- Create a recipient-specific version or regenerate the deterministic header.
- Require recipient verification again when destination data changes.
- Require review again because the pharmacist must confirm the intended recipient.

## 26. Transmission and Delivery Status

Keep content lifecycle separate from transmission lifecycle.

```ts
type DocumentReviewStatus =
  | "GENERATING"
  | "REVIEW_REQUIRED"
  | "UPDATED_REVIEW_REQUIRED"
  | "SOURCE_CHANGED"
  | "REVIEWED"
  | "FINALIZED"
  | "GENERATION_FAILED";

type CommunicationTransmissionStatus =
  | "NOT_READY"
  | "READY_TO_SEND"
  | "SEND_INITIATED"
  | "SENT_EXTERNALLY_UNCONFIRMED"
  | "DELIVERED"
  | "FAILED"
  | "CANCELLED";
```

Status rules:

- Newly generated: `REVIEW_REQUIRED` + `NOT_READY`.
- Reviewed with verified recipient: `REVIEWED` + `READY_TO_SEND`.
- Copied or downloaded for external sending: do not mark `DELIVERED`.
- If the pharmacist sends outside SafeScribe, require explicit confirmation of date, time, method, and destination before setting `SENT_EXTERNALLY_UNCONFIRMED`.
- A direct integration may set `SEND_INITIATED`, `DELIVERED`, or `FAILED` only from the vendor response and approved status mapping.
- `DELIVERED` means the destination system accepted delivery; it does not mean the clinician read or agreed with the content.
- Never use `NOTIFIED`, `ACKNOWLEDGED`, or `ACCEPTED` as inferred states.

## 27. Communication Attempt Record

Create an append-only communication-attempt record.

```ts
type CommunicationAttempt = {
  id: string;
  consultationId: string;
  documentVersionId: string;
  recipientId: string;
  destinationType: "SECURE_FAX" | "SECURE_MESSAGE" | "PRINT" | "OTHER_APPROVED";
  destinationMasked: string;
  initiatedBy: string;
  initiatedAt: string;
  status: CommunicationTransmissionStatus;
  vendorReference?: string;
  statusUpdatedAt: string;
  failureCode?: string;
  externalSendConfirmedBy?: string;
  externalSendConfirmedAt?: string;
};
```

Rules:

- Do not store full fax numbers or addresses in general audit logs.
- Preserve the actual approved destination in the protected clinical/communication record as required by policy.
- Do not overwrite failed attempts; append a new attempt for retries.
- Link every attempt to the exact reviewed document version.
- If the document changes after a failed attempt, send only the newly reviewed version.
- Record notification date and method in the patient record or approved record export.

## 28. Completion Gating

When communication is `REQUIRED`, SafeScribe must not present the consultation as fully complete merely because the communication document was generated.

Minimum completion conditions:

- Current document version reviewed.
- Recipient verified.
- Current version linked to the intended recipient.
- Required communication attempt recorded through an approved method, or approved external-send confirmation completed.
- No failed attempt left without a documented retry or escalation plan.

If the product permits deferred communication, implement a separate approved workflow with:

- Reason for deferral.
- Responsible pharmacist.
- Due date/time consistent with `as soon as reasonably possible`.
- Visible outstanding-task state.
- Escalation if not completed.

Do not implement silent deferral. Do not let end-of-day SafeScribe deletion erase the only copy of an outstanding required communication.

## 29. Versioning and Data Model

```ts
type ProviderCommunicationVersion = {
  id: string;
  consultationId: string;
  documentType: "PRIMARY_CARE_PROVIDER_COMMUNICATION";
  recipientId?: string;
  versionNumber: number;
  status: DocumentReviewStatus;
  sourceSnapshotHash: string;
  sourceConsultationVersion: number;
  promptVersion: string;
  modelRoute: string;
  communicationPurpose: CommunicationPurpose;
  actionExpectation: RecipientActionExpectation;
  requirementStatus: CommunicationRequirementStatus;
  generatedContentJson: GeneratedProviderCommunicationV1;
  generatedPlainText: string;
  generatedPdfReference?: string;
  editedPlainText?: string;
  generatedAt: string;
  editedBy?: string;
  editedAt?: string;
  reviewedBy?: string;
  reviewedAt?: string;
  finalizedBy?: string;
  finalizedAt?: string;
  transmissionStatus: CommunicationTransmissionStatus;
};
```

Rules:

- Never update a finalized or transmitted version in place.
- Preserve generated and pharmacist-edited content for the approved active retention period.
- Review applies only to one exact document version and recipient.
- Record source snapshot hash, prompt version, model route, reviewer, purpose, action expectation, and requirement decision.
- Do not store document content in a general audit-log table.
- Corrections after transmission require a corrected communication linked to the original, not silent replacement.

## 30. Suggested Service Interface

```ts
generateProviderCommunication(
  consultationId: string,
  recipientId?: string
): Promise<{
  documentId: string;
  versionId: string;
  reviewStatus: "REVIEW_REQUIRED" | "GENERATION_FAILED";
  transmissionStatus: "NOT_READY";
}>;

updateProviderCommunication(
  versionId: string,
  editedPlainText: string
): Promise<{
  newVersionId: string;
  reviewStatus: "UPDATED_REVIEW_REQUIRED";
  transmissionStatus: "NOT_READY";
}>;

markProviderCommunicationReviewed(
  versionId: string,
  pharmacistId: string
): Promise<{
  reviewStatus: "REVIEWED";
  transmissionStatus: "READY_TO_SEND" | "NOT_READY";
}>;

recordExternalCommunication(
  versionId: string,
  recipientId: string,
  method: string,
  occurredAt: string,
  pharmacistId: string
): Promise<{
  attemptId: string;
  transmissionStatus: "SENT_EXTERNALLY_UNCONFIRMED";
}>;

sendProviderCommunication(
  versionId: string,
  recipientId: string,
  channel: string
): Promise<{
  attemptId: string;
  transmissionStatus: "SEND_INITIATED" | "DELIVERED" | "FAILED";
}>;
```

Backend authorization must confirm tenant membership, pharmacist permissions, recipient access, current version, and approved communication channel.

## 31. Failure Messages

### Required consultation information missing

> This communication is not ready. Return to the highlighted consultation step and complete the required clinical information.

### Communication requirement unresolved

> Confirm whether another regulated health professional's care may be affected before generating the communication.

### Recipient not verified

> Verify the intended recipient and destination before sending this communication.

### Source conflict

> Conflicting consultation information must be resolved before the communication can be generated.

### Generation failed

> The provider communication could not be generated. Try again or create it manually from the confirmed consultation summary.

### Source changed

> Consultation information changed after this communication was generated. Generate and review an updated version before sending it.

### Not reviewed

> Review and confirm the current communication before copying, downloading as final, or sending it.

### Transmission failed

> The communication was not delivered. Verify the destination and retry using an approved method.

### Urgent referral handoff incomplete

> The urgent referral document is supporting information only. Complete and record the required urgent handoff before finishing the consultation.

## 32. Acceptance Tests

### Purpose and content

1. Initial-access prescribing produces a communication with a visible purpose and action expectation.
2. Alberta prescribing communication contains drug and amount, rationale, prescribing date, non-drug recommendation state, monitoring plan, and patient instructions.
3. The document is shorter than the detailed Pharmacist Consultation Note and avoids duplicating the full assessment.
4. Provider action is explicit and appears at the end.
5. `FOR_INFORMATION_ONLY` clearly states that no response is requested.
6. `ACTION_REQUESTED` contains the exact confirmed action and timing.
7. A non-prescribing referral does not display prescription headings.

### Missing data and non-invention

8. No blood pressure entered: no blood-pressure statement or blank label appears.
9. No lab entered: no lab statement or blank label appears.
10. A symptom was not asked: the communication does not state it was denied.
11. No examination recorded: no examination finding is generated.
12. No counselling confirmation: the communication does not claim counselling was provided.
13. No communication attempt: the document does not state the PCP was notified.
14. Tentative assessment: the output preserves tentative wording.
15. Optional information missing: the associated section or sentence disappears cleanly.

### Exact values

16. Prescription details exactly match the final prescription record.
17. Prescribing date is distinct from document-created and transmission dates.
18. Monitoring values, intervals, and responsible person exactly match the source plan.
19. Requested action and timing exactly match pharmacist-confirmed values.
20. Every required protected token appears once.
21. The model does not calculate or convert any clinical value.

### Recipient safety

22. Unverified recipient: draft preview allowed, final transmission blocked.
23. Recipient changed: current reviewed status is cleared.
24. Destination changed: recipient verification and review are required again.
25. Multiple affected providers: separate recipient-specific versions are generated.
26. Missing recipient relationship: document does not call the recipient the patient's PCP.
27. Unverified fax or address never appears in a transmission-ready document.

### Requirement and safety gating

28. `PENDING_DETERMINATION`: model is not called.
29. Required prescribing element missing: generation is blocked upstream.
30. Unresolved allergy-treatment conflict: generation is blocked.
31. Contradictory active facts: generation is blocked.
32. Urgent referral: a supporting document alone cannot complete the handoff.
33. No affected provider identified: system records the decision and does not create a generic sent letter.

### Review and version lifecycle

34. New communication is `REVIEW_REQUIRED` and watermarked as a draft.
35. Previewing, copying, or downloading does not mark it reviewed.
36. Review applies only to the current version and recipient.
37. Editing a reviewed communication creates a new version and requires review again.
38. Clinically relevant upstream change sets `SOURCE_CHANGED`.
39. Transmitted version cannot be overwritten in place.
40. Corrected communication links to the prior transmitted version.

### Transmission lifecycle

41. Copying or downloading does not set `DELIVERED`.
42. External-send confirmation captures date, time, method, recipient, destination, and pharmacist.
43. Direct send failure creates an append-only failed attempt.
44. Retry creates a new attempt linked to the same or newer reviewed version.
45. `DELIVERED` does not imply read, acknowledged, or accepted.
46. Required communication prevents normal completion until sent or placed into an approved deferred workflow.
47. Notification date and method are available to the permanent patient record.

### Privacy and technical controls

48. Direct identifiers are absent from the model request.
49. API requests set `store: false`.
50. Routine logs contain no prompt, response, identifiers, destination, protected fragment, or document body.
51. Schema-invalid, refused, or truncated output is never shown as completed.
52. Source fact IDs and protected tokens are validated against the request.
53. Plain-text and PDF versions contain the same clinical content.
54. Unreviewed PDFs visibly display `DRAFT - NOT YET REVIEWED`.
55. The source snapshot, requirement rule, prompt version, model route, pharmacist review, recipient, and transmission attempt remain traceable for the approved retention period.

## 33. Definition of Done

This feature is complete when:

- The communication is assembled only from confirmed SafeScribe data.
- It is clearly shorter and more action-oriented than the consultation note.
- Required Alberta prescribing elements are enforced deterministically.
- Missing optional information disappears without placeholders or assumptions.
- Required missing or conflicting information is blocked upstream.
- Exact prescription, monitoring, date, and action values cannot be rewritten by the model.
- Recipient identity and destination must be verified before transmission.
- The pharmacist can review, edit, and explicitly approve the current recipient-specific version.
- PDF and plain-text outputs are professionally formatted and consistent.
- Document generation, pharmacist review, transmission, delivery, and acknowledgement are separate states.
- Communication attempts are append-only and linked to exact document versions.
- Required communication cannot be silently skipped or lost under the end-of-day retention process.
- Direct identifiers and document content are excluded from model storage and routine logs.

## 34. Superseded Requirements

For this document type, these instructions supersede older SafeScribe references to:

- A generic physician letter containing the full consultation note.
- Hardcoding every recipient as `family physician` or `PCP`.
- Treating `Generated`, `Copied`, `Downloaded`, or `Printed` as proof the provider was notified.
- Asking the document-generation model to decide whether communication is required.
- Letting the model rewrite prescription, monitoring, or provider-action values.
- Filling every empty field with `not available` or `not provided`.
- Sending an urgent referral document without an active handoff workflow.

The current requirement is a concise, recipient-specific, source-bound, non-inventive, pharmacist-reviewed provider communication with explicit action status, deterministic clinical values, verified routing, and auditable transmission.
