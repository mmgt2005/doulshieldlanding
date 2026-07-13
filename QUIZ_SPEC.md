# DoulaShield Quizzes — Questions & Scoring Spec

This document specifies the two quizzes on the DoulaShield marketing site so they can be
reimplemented in the product / backend. Both submit to `POST {base}/api/v1/public/leads/quiz`.

---

## Quiz 1 — PA Medicaid Doula Billing Self-Assessment (individual providers)

**Format:** 5 single-select questions, presented one at a time. Each option carries a
`complete` flag — whether choosing it means that pipeline stage is *done*.

### Questions

**Q1 — key `pcb_certification`** — "Do you hold a current PCB doula certification?"
| Option | complete |
|--------|----------|
| Yes, I'm certified | `true` |
| I've applied / it's in progress | `false` |
| Not yet | `false` |

**Q2 — key `npi`** — "Do you have an active NPI (registered in NPPES)?"
| Option | complete |
|--------|----------|
| Yes, I have my NPI | `true` |
| No, or I'm not sure | `false` |

**Q3 — key `promise_enrollment`** — "Are you enrolled as a provider in PA PROMISe™?"
| Option | complete |
|--------|----------|
| Yes, enrollment is complete | `true` |
| Started but not finished | `false` |
| No | `false` |

**Q4 — key `mco_contracting`** — "Have you contracted with any PA Medicaid MCOs?"
| Option | complete |
|--------|----------|
| Yes, with one or more MCOs | `true` |
| In progress | `false` |
| No | `false` |

**Q5 — key `cms1500_claim`** — "Have you ever successfully submitted a CMS-1500 claim?"
| Option | complete |
|--------|----------|
| Yes, and I've been reimbursed | `true` |
| I've tried but hit errors | `false` |
| Never | `false` |

### Scoring logic

The result maps the visitor to the **first pipeline stage they haven't completed**.
Q1–Q4 correspond to pipeline stages 1–4. Q5 only distinguishes the two "credentialed"
end states.

```
resultIdx = null
// walk the first 4 questions in order; stop at the first incomplete one
for i in 0..3:
    done = (answer[i] exists) AND (chosen option for Qi has complete == true)
    if not done:
        resultIdx = i        // 0,1,2,3 -> stages 1–4
        break

if resultIdx is null:        // Q1–Q4 all complete
    claimDone = (answer[4] exists) AND (chosen option for Q5 has complete == true)
    resultIdx = claimDone ? 5 : 4
```

`resultIdx` ∈ 0–5:

| resultIdx | Badge | Title | Meaning |
|-----------|-------|-------|---------|
| 0 | Stage 1 · Certification | Let's start with your PCB certification. | PCB not done |
| 1 | Stage 2 · NPI setup | Certified — next up is your NPI. | PCB done, NPI not |
| 2 | Stage 3 · PROMISe™ | You're close — PROMISe™ enrollment is next. | through NPI |
| 3 | Stage 4 · MCO contracting | Almost there — let's get you contracted. | through PROMISe™ |
| 4 | Ready to bill | You're credentialed — let's get claims paid. | all 4 stages done, no paid claim |
| 5 | Fully operational | You're already getting paid — nice work. | all 4 stages done + a paid claim |

### Result copy (body + next steps)

**0 — Stage 1 · Certification**
Body: "You're at the very beginning of the pipeline — which is exactly where DoulaShield is most useful. We'll guide your PCB application and handle everything that follows."
Next steps:
- Complete your PCB certification with our guided, pre-filled forms
- We register your NPI as your administrative delegate
- We handle PROMISe™ enrollment and MCO contracting for you

**1 — Stage 2 · NPI setup**
Body: "Your PCB certification is handled. The next hurdle is registering your NPI correctly, then PROMISe™ and MCO contracting. We take it from here."
Next steps:
- We register your NPPES / NPI as your delegate
- We complete your PROMISe™ provider enrollment
- We contract you with the right PA Medicaid MCOs

**2 — Stage 3 · PROMISe™**
Body: "Certification and NPI are in place. PROMISe™ enrollment is where many doulas stall. We'll finish it and get you contracted with MCOs."
Next steps:
- We complete and track your PROMISe™ enrollment
- We contract you with PA Medicaid MCOs
- We set you up to submit CMS-1500 claims

**3 — Stage 4 · MCO contracting**
Body: "You're enrolled in PROMISe™ and just need MCO contracts to start billing. We handle the contracting and get your claims flowing."
Next steps:
- We contract you with the PA Medicaid MCOs you serve
- We generate and submit your CMS-1500 claims via Availity
- We track every claim through to reimbursement

**4 — Ready to bill**
Body: "You've cleared the credentialing pipeline. If claims are still bouncing or slow, DoulaShield's claims engine and audit-proof vault clean up the billing side."
Next steps:
- Generate error-checked CMS-1500 claims automatically
- Submit 837P electronically through Availity
- Track submitted, paid, and denied claims in one queue

**5 — Fully operational**
Body: "You've made it through the whole pipeline and you're billing successfully. DoulaShield can still cut your admin time, catch denials early, and keep every credential current."
Next steps:
- Automate expiry reminders for NPI, CAQH, PROMISe™ & PCB
- Keep an audit-proof vault of every claim and document
- Add providers under a group NPI as you grow

### Lead payload (individual)

```json
{
  "first_name": "...",
  "last_name": "...",
  "email": "...",
  "provider_type": "independent",
  "answers": {
    "assessment_result": "<badge, e.g. 'Stage 2 · NPI setup'>",
    "recommendation": "<result title>",
    "pcb_certification": "<selected option label>",
    "npi": "<label>",
    "promise_enrollment": "<label>",
    "mco_contracting": "<label>",
    "cms1500_claim": "<label>"
  }
}
```

Unanswered questions send the string `"no answer"`.

---

## Quiz 2 — Doula Agency Compliance Audit Checklist (agencies)

**Format:** 9 checkboxes (multi-select, each independently on/off).

| # | Slug | Item |
|---|------|------|
| 1 | `group_npi` | We operate under a shared group NPI for all providers |
| 2 | `billing_admin` | A billing admin is assigned to review and submit claims |
| 3 | `availity_configured` | Availity credentials are configured for electronic 837P submission |
| 4 | `promise_enrolled` | Every provider has completed PROMISe™ enrollment |
| 5 | `expiry_tracking` | We track NPI, CAQH, PROMISe™ & PCB expiry dates |
| 6 | `client_verified` | Client Medicaid details are verified before billing |
| 7 | `zip4_enriched` | Addresses are ZIP+4 enriched for CMS-1500 accuracy |
| 8 | `audit_logging` | Sensitive actions are audit-logged for compliance |
| 9 | `surrogate_agreements` | Surrogate authorization agreements are on file per provider |

### Scoring logic

```
pct = round( checkedCount / 9 * 100 )

if pct >= 80:   band = "Strong"     // advice: "Your agency is in great shape…"
elif pct >= 50: band = "Some gaps"  // advice: "A solid foundation with a few gaps…"
else:           band = "At risk"    // advice: "Several core requirements are missing…"
```

Advice copy per band:
- **Strong (≥80%):** "Your agency is in great shape. DoulaShield keeps you there — automating expiry tracking, audit logs, and claim submission."
- **Some gaps (50–79%):** "A solid foundation with a few gaps to close. These are quick wins that reduce denials and audit risk."
- **At risk (<50%):** "Several core requirements are missing. These are the gaps that most often stall reimbursement — let's close them together."

### Lead payload (agency)

```json
{
  "first_name": "...",
  "last_name": "...",
  "email": "...",
  "provider_type": "agency",
  "organization_name": "...(optional)",
  "answers": {
    "compliance_score": "78%",
    "compliance_band": "Some gaps",
    "group_npi": "yes",
    "billing_admin": "no",
    "availity_configured": "yes",
    "promise_enrolled": "yes",
    "expiry_tracking": "no",
    "client_verified": "yes",
    "zip4_enriched": "no",
    "audit_logging": "yes",
    "surrogate_agreements": "no"
  }
}
```

Each checklist slug is `"yes"` (checked) or `"no"` (unchecked).

---

## API endpoints (reference)

All three use the same pattern — `POST {base}/api/v1/public/leads/{quiz|webinar|contact}`,
`Content-Type: application/json`, rate-limited **5 req/min per IP**, response
`{"status": "ok", "id": "<lead-uuid>"}`.

- **quiz** — required `first_name`, `last_name`, `email`; optional `phone`,
  `provider_type` (`"independent"` | `"agency"` | `"unknown"`), `organization_name`,
  `answers` (free-form object as specified above).
- **webinar** — same required fields; optional `phone`, `organization_name`, `webinar_topic`.
- **contact** — same required fields; optional `phone`, `organization_name`, `message`.

Empty optional fields are stripped client-side before sending. The marketing site's origin
must be in the backend `CORS_EXTRA_ORIGINS` allowlist for the browser `fetch()` to succeed.
