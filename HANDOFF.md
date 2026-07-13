# DoulaShield Landing Page — Handoff Notes

Living notes for the marketing/landing-page project. If it's written here it survives into
future chats; conversation history does not. Companion doc: **QUIZ_SPEC.md** (full quiz
questions + scoring logic for backend reimplementation).

---

## What this project is

A single-file, multi-view marketing site for **DoulaShield** — a SaaS platform for
PA Medicaid doula providers and agencies. Built as one Design Component:

- **`DoulaShield.dc.html`** — the entire site (landing + webinar + quiz + contact).
- **`QUIZ_SPEC.md`** — quiz questions, options, scoring logic, and lead payloads.
- **`assets/`** — `doulashield-logo.png` (full logo) and `doulashield-mark.png` (cropped shield graphic used in nav).
- **`support.js`** — DC runtime (auto-generated, do not edit).

### Views (client-side, no router library)
`home` · `webinar` · `quiz` (two tabs: individual self-assessment + agency audit) · `contact`.
Brand palette pulled from the logo: magenta `#BF206F`, orange `#F88017`, yellow `#FCC11C`.
Warm/maternal treatment, cream background `#FBF5ED`, Newsreader (serif) + Mulish (sans).

---

## Configurable settings (Tweaks panel → props on the root DC)

| Prop | Section | Purpose |
|------|---------|---------|
| `apiBaseUrl` | Integrations | Base URL of the DoulaShield API, e.g. `https://api.doulashield.com`. Blank = prototype mode (payloads logged to console only, no network call). |
| `individualWebinarVideoUrl` | Webinar videos | Share/embed URL for the Individual Provider recording. |
| `agencyWebinarVideoUrl` | Webinar videos | Share/embed URL for the Agency recording. |

Video URLs accept YouTube / Vimeo / Loom **share** links (or a raw embed URL) — the page
auto-converts them to an autoplaying embed for the matching track. Until set, a labeled
placeholder shows.

---

## Lead capture — API wiring

All forms POST JSON to `{apiBaseUrl}/api/v1/public/leads/{quiz|webinar|contact}`.
Rate limit: **5 requests/min per IP**. Success response: `{"status":"ok","id":"<uuid>"}`.
Empty optional fields are stripped client-side before sending.

- **Webinar form** → `/webinar` — `first_name`, `last_name`, `email`, optional
  `organization_name`, `webinar_topic` (the selected session title).
- **Quiz (individual result)** → `/quiz` — name/email, `provider_type: "independent"`,
  free-form `answers` (see QUIZ_SPEC.md).
- **Quiz (agency audit)** → `/quiz` — name/email, optional `organization_name`,
  `provider_type: "agency"`, free-form `answers`.
- **Contact form** → `/contact` — name/email, optional `organization_name`, `message`.

Lead POST is **fire-and-forget** — the UI advances to the success/video state immediately
and does not wait for or gate on the API response.

### CORS (required for live submissions)
The browser calls the API cross-origin, so the landing page's origin must be in the backend
allowlist. On Railway this is the **`CORS_EXTRA_ORIGINS`** env var (comma-separated, no
trailing slashes), in addition to `FRONTEND_ORIGIN`. Example:
```
CORS_EXTRA_ORIGINS=https://get.doulashield.com,https://doulashield.netlify.app
```
Origins must match exactly (scheme + host; apex ≠ www; https ≠ http). Preview deploys are
different origins. Confirm in DevTools → Network: `OPTIONS` → 204, then `POST` → 200.

### Admin email / prospect email (backend, not this page)
Per the API docs, all three endpoints send an **admin** notification (via Resend). The page
copy promises the *visitor* an email ("We'll email your results" / "Watch Now link").
**Open item:** backend should send a second Resend email to the prospect's `email`, or the
page copy should be softened. See open decisions below.

---

## Webinar = immediate on-demand video + Q&A chat

The webinars are framed as **on-demand** (not live): no scheduled times, no live Q&A. Copy
reads "Free on-demand webinar," "45 min · watch instantly," CTA "Get instant access."

After registering, the visitor lands on a "You're registered — Watch [session] now" panel
with the recording embedded (16:9), driven by the per-track video URL props above. Access is
immediate and unconditional (not gated on the API 201).

**AI Q&A chat** sits below the video (replaces the live-host Q&A). It uses the built-in
Claude helper (`window.claude.complete` — no API key/backend needed), grounded with a system
prompt covering the 4-stage pipeline, DoulaShield's delegate model, and exact pricing.
Answers are short and non-salesy; it defers to the contact form for human matters. Shared
rate limit: 15 questions/min per viewer.

**Chat questions are logged to the API.** Registrant details (first/last/email/org) are
captured at webinar submit and stored in state; each question is POSTed to
`/api/v1/public/leads/contact` (fire-and-forget) with `message` = `"[Webinar Q&A — <session title>] <question>"`.
So every question a viewer asks becomes a contact lead attributed to the registrant.

---

## Deep links (hash routes)

Append to the deployed page URL (the `#` matters; hash routing works on any static host):

| Link | Lands on |
|------|----------|
| `#webinar` or `#webinar-individual` | Webinar page, Individual track selected |
| `#webinar-agency` | Webinar page, Agency track selected |
| `#quiz` or `#quiz-individual` | Quiz, individual self-assessment tab |
| `#quiz-agency` | Quiz, agency audit tab |
| `#contact` | Contact form |
| `#home` or bare URL | Home |

The hash is read on load and updates live in the address bar as visitors navigate. Unknown
hash → falls back to home. Case-insensitive.

---

## Pricing shown on the landing page

**Independent doulas** — two paths:
- Already credentialed: **$39/month** (claims, billing & audit vault; start right away).
- Full enrollment service: **$99** deposit + **$400** escrow (charged only after first claim
  is paid in DoulaShield) → then **$39/month** once earning.

**Doula agencies** — two tiers:
- Billing tier: **$55 / seat / month**, 3-seat minimum ($165/mo floor), auto-adjusting.
- Enrollment tier (add-on): **+$15 / seat / month** to use DoulaShield's enrollment system
  to credential providers.

---

## Open decisions / TODO

1. **Video gating:** currently the video shows immediately on submit (forgiving). Claude Code
   suggested gating on the API's 201 response — not implemented. Decide which behavior you want.
2. **~~"Live" vs on-demand copy~~ — DONE:** webinars reframed as on-demand (times removed,
   "live" language dropped, CTA "Get instant access"). Revisit only if you start running live
   sessions again.
3. **Prospect confirmation email:** see CORS/email section — backend needs a Resend email to
   the prospect, or soften the page's "we'll email you" copy.
4. **Webinar video URLs** must be set in Tweaks (individual + agency) or a placeholder shows.
5. **Hero image** is a drag-and-drop `<image-slot id="ds-hero">` — the user drops their own
   warm/diverse hero photo onto it and it persists. No stock photo is bundled (can't ship a
   licensed one).
6. **MCO logos as social proof** — ADDED as a muted strip below the hero ("Claims routed to
   every PA Medicaid MCO") with **placeholder** wordmarks + a trademark disclaimer. Swap in
   real logo files when available (see note below); the MCO list lives in the `mcos` array in
   the logic class.

---

## Editing notes

- The whole site is one Design Component (`DoulaShield.dc.html`). Edit via the DC tools; the
  logic class holds all content arrays (`stages`, `features`, `webinarData`, `quizQuestions`,
  `auditItems`) and handlers.
- No persistence: registration/quiz state is in-memory only. **Reload = clean reset.** No
  "already registered" lock; forms can be resubmitted freely.
