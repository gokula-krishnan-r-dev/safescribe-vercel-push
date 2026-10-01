# Treatment Warning Reasons — Pharmacy Authoring Guide

**Audience:** Pharmacist Admins / clinical pathway authors  
**Feature:** Pathway treatment Yes/No safety flags + pharmacist-authored **reason** sentences  
**Version:** 1.0

---

## What changed

Every treatment safety flag that can show a **warning in consultation** now has a matching **reason** field. Pharmacists see *why* the caution applies — not only that a flag is set.

| Flag (Yes / No) | Reason field | Shown in consultation when |
|---|---|---|
| Pregnancy / lactation | Pregnancy reason | Flag = **Yes** *and* patient pregnancy answer = **Yes** |
| Renal adjustment | Renal reason | Flag = **Yes** |
| Hepatic adjustment | Hepatic reason | Flag = **Yes** |
| Monitoring | Monitoring reason | Flag = **Yes** |

If the flag is **Yes** but the reason is left blank, SafeScribe uses a standard clinical fallback sentence. Authors should always fill the reason so the message matches your pathway.

---

## How it works (end-to-end)

```
Pathway author sets flag = Yes + reason sentence
        ↓
Saved on ClinicalTreatment (database)
        ↓
Consultation Step 7 loads treatments
        ↓
SafeScribe builds warning banners using the authored reason
        ↓
Pregnancy banner only if the patient is pregnant
```

1. **Author** a treatment in the pathway (UI form, Excel import, or ChatGPT import).
2. Set each relevant flag to **Yes** or **No**.
3. When **Yes**, enter a short pharmacist-facing **reason** (1–2 sentences).
4. **Approve** the treatment (incomplete if a Yes flag has no reason).
5. During a consultation, Step 7 shows a banner with **Reason:** … for each active caution.

---

## Authoring in the UI

**Pathways → [Pathway] → Treatments → Add / Edit**

1. Under safety flags, choose **Yes** or **No** for Renal, Hepatic, Pregnancy / lactation, Monitoring.
2. When **Yes**, a **reason** text box appears — fill it before saving.
3. Expand a treatment card to review flag + reason together.
4. Approve only when required fields (including reasons) are complete.

**Good reason examples**

- Renal: `Reduce dose when eGFR is below 30 mL/min — risk of accumulation and neurotoxicity`
- Pregnancy: `Limited human data in pregnancy — confirm benefit outweighs risk; prefer specialist advice in first trimester`
- Monitoring: `Recheck symptoms in 48–72 hours; seek care if lesions worsen or fever develops`

**Avoid**

- Only writing `Yes` or `Caution` without clinical meaning
- Copy-pasting the entire product monograph — keep it actionable and short

---

## Excel / CSV import (new)

Use **Excel import** on the Treatments tab (or download the template).

### Required columns for warnings

| Column | Values | Notes |
|---|---|---|
| `pregnancy_flag` | Yes / No | Optional |
| `pregnancy_reason` | Free text | **Required if** `pregnancy_flag` = Yes |
| `renal_flag` | Yes / No | Optional |
| `renal_reason` | Free text | **Required if** `renal_flag` = Yes |
| `hepatic_flag` | Yes / No | Optional |
| `hepatic_reason` | Free text | **Required if** `hepatic_flag` = Yes |
| `monitoring_flag` | Yes / No | Optional |
| `monitoring_reason` | Free text | **Required if** `monitoring_flag` = Yes |

Also include core columns such as `medication_name`, `category`, `dose`, `route`, etc. See the template’s **Column_Guide** sheet.

### Sample file

A ready-to-edit sample is in the repository:

`docs/samples/pathway-treatments-sample.csv`

Or download **pathway-treatments-template.xlsx** from the Treatments tab (includes two sample rows).

### Import steps

1. Download the template (or copy the sample CSV into Excel).
2. Fill one row per treatment option.
3. Set flags and reasons.
4. On the pathway Treatments tab → **Excel import** → choose file → preview → **Import treatments**.
5. Review imported options (they arrive **unapproved**) → edit if needed → approve.

Validation fails if any **Yes** flag is missing its reason.

---

## ChatGPT import

When pasting ChatGPT treatment markdown, use:

```
Pregnancy: Yes
Pregnancy reason: …
Renal adjustment: Yes
Renal reason: …
Hepatic adjustment: No
Monitoring: Yes
Monitoring reason: …
```

The ChatGPT prompt template for Treatments already asks for these fields.

---

## What pharmacists see in consultation

For each selected pathway treatment:

- **Renal / Hepatic / Monitoring** banners show immediately when the flag is Yes, with your reason.
- **Pregnancy** banner shows only when the patient is recorded as pregnant **and** the pregnancy flag is Yes.
- The collapsed treatment row may also surface the primary caution reason under “Review required”.

CDS / treatment-safety panels reuse the same authored reasons (not a bare “Yes”).

---

## Operational checklist for pharmacy teams

- [ ] Update existing treatments: set Yes/No flags and write reasons where Yes.
- [ ] Prefer Excel bulk update for large pathways using the sample/template.
- [ ] Do not approve treatments that still show “Incomplete” for missing reasons.
- [ ] Spot-check one consultation with a pregnant test patient to confirm pregnancy reasons appear only when appropriate.
- [ ] Keep reasons province- and pathway-specific where dosing differs.

---

## Support notes

- Clearing a flag to **No** clears its stored reason on save.
- Legacy free-text in old flag fields is treated as **Yes** so existing cautions are not lost; rewrite them into Yes + reason when you next edit.
- Questions? Contact your SafeScribe Pharmacist Admin or platform support.
