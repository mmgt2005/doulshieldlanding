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
| `turnstileSiteKey` | Integrations | Cloudflare Turnstile site key (public, safe to embed). Blank = bot protection off — all 4 lead forms submit exactly as before, no `turnstile_token` sent. See "Bot protection" below. |
| `individualWebinarPart1VideoUrl` / `Part2VideoUrl` | Webinar videos | Share/embed URL for each part of the Individual Provider series (2 parts). |
| `agencyWebinarPart1VideoUrl` / `Part2VideoUrl` | Webinar videos | Share/embed URL for each part of the Agency series (2 parts). |

Video URLs accept YouTube / Vimeo / Loom / **Google Drive** share links (or a raw embed
URL) — the page auto-converts them to an embeddable iframe `src` for the matching part
(autoplaying for YouTube/Vimeo/Loom; Drive doesn't support an autoplay param, so Drive
embeds start paused). A Drive share link (`drive.google.com/file/d/<id>/view?usp=sharing`)
converts to `drive.google.com/file/d/<id>/preview` — the raw `/view` link does **not**
embed. **The Drive file's sharing setting must be "Anyone with the link can view"**, or
visitors will see an access-denied page inside the iframe instead of the video — this
can't be verified from the code, check it manually after publishing a Drive link. Until a
part's video URL is set, a labeled placeholder shows for that part.

**Each track is a series of parts**, not one video — see `webinarData` in the logic class
(search for `parts:`). Part *count* and *labels* are a code edit (hardcoded in
`webinarData`, same as `title`/`desc`/`tag`); only the video URL per part is Tweaks-editable.
Tracks can have different part counts (Individual and Agency aren't required to match). To
add or remove a part: add/remove both the `webinarData[i].parts` entry **and** its matching
`{track}WebinarPart{N}VideoUrl` Tweaks prop together — they're tied by the `videoProp`
string on each part object, so a mismatch silently shows the empty-video placeholder rather
than erroring.

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

**Webinar engagement timestamps:** if a visitor registered for a webinar earlier in the
session, the `/quiz` and `/contact` payloads above also carry `webinar_reached_at` (ISO
timestamp — when the webinar video/success view first appeared), `webinar_cta_clicked_at`
(ISO timestamp — when they clicked "Book a setup call" or "Take the free quiz next" from
that panel), and `webinar_part_at_cta` (which part's label, e.g. `"Part 2"`, they were on
when they clicked). Backend can diff the two timestamps for elapsed watch time. All three
are `null`/omitted for leads that didn't come through the webinar flow. This is a
time-on-page approximation, not proof the video was actually watched — the page doesn't
integrate any video-provider SDK to detect real playback/completion (embeds can be
YouTube, Vimeo, or Loom depending on what's set in Tweaks, so there's no single generic
"video ended" event without picking one provider's JS SDK).

Lead POST is **fire-and-forget** — the UI advances to the success/video state immediately
and does not wait for or gate on the API response.

### Bot protection (Cloudflare Turnstile) — frontend wired, backend enforcement pending

All 4 lead forms (webinar, quiz individual, quiz agency, contact) optionally render a
Cloudflare Turnstile widget just above their submit button, driven by the `turnstileSiteKey`
Tweaks prop above. The Turnstile script loads in **explicit render mode**
(`<script ...?onload=onloadTurnstileApi&render=explicit">` in `<helmet>`), and each widget
is rendered on demand via `mountTurnstileWidgets()` (called from `componentDidMount`/
`componentDidUpdate`, plus the script's own `onload` callback) rather than auto-rendering —
this is required because each form lives inside an `<sc-if>` that genuinely unmounts the
form (switching quiz tabs, "Retake the quiz", a successful submit), so a widget is destroyed
and freshly re-rendered every time its form reappears. Verified safe against React wiping
the widget's injected DOM on unrelated re-renders (e.g. clicking between the webinar's
track cards) — the containers are declared with zero template children, which this
templating system's compiled output never touches once rendered.

When a widget completes, its token lands in `state.turnstileTokens` and rides along on
that form's existing `sendLead(...)` payload as `turnstile_token` — stripped like any other
empty field when blank (feature off, or not yet completed).

**Server-side verification is NOT implemented in this repo and must be added on the
Railway backend** before this actually blocks anything: for each of the 4 lead endpoints,
call `https://challenges.cloudflare.com/turnstile/v0/siteverify` with a **secret key**
(`TURNSTILE_SECRET_KEY` env var or similar — never exposed client-side, separate from the
public `turnstileSiteKey` above) and reject the lead if verification fails. Until that's
built, the token is collected but nothing checks it — bots aren't actually blocked yet,
only the client-side half is done. `sendLead` stays fire-and-forget on purpose: a legit
visitor's token just verifies and they see nothing different; a bot's missing/forged token
gets silently dropped server-side once that half exists — no new error UI needed here.

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
with the recording embedded (16:9), driven by the per-part video URL props above. Access is
immediate and unconditional (not gated on the API 201).

**Each track can be a multi-part series.** If a track has more than one part (see
`webinarData` above), a pill row ("Part 1 / Part 2 / ...") appears above the video so the
visitor can jump between parts freely, plus a "Next part →" button for convenience (fades
out on the last part). Registration is a one-time, track-level action — it is **not**
re-gated per part, and there's no enforced watch order; all parts are open immediately
after the single registration. A track with only one part shows no pill row at all (falls
back to the original single-video layout).

**AI Q&A chat** sits below the video (replaces the live-host Q&A), grounded with a system
prompt covering the 4-stage pipeline, DoulaShield's delegate model, and exact pricing
(see `chatSystemPrompt()` in the logic class — edit the prompt text there, nowhere else).
Answers are short and non-salesy; it defers to the contact form for human matters.

**Requires a backend endpoint — this was broken on the deployed site until fixed.** The
page previously called `window.claude.complete(...)`, a bridge that only exists inside the
Claude Design Canvas editor tool itself; every deployed visitor hit an error ("Something
went wrong reaching the assistant") on every question. The frontend now POSTs to
`{apiBaseUrl}/api/v1/public/webinar-chat`:

```
POST {apiBaseUrl}/api/v1/public/webinar-chat
Content-Type: application/json

{ "system": "<the system prompt string>", "messages": [{ "role": "user"|"assistant", "content": "..." }, ...] }
```

Expected response: `200 OK` with `{ "reply": "<assistant's reply text>" }`. On any non-2xx
response, network error, or missing `apiBaseUrl`, the page shows a fallback message and
points the visitor to the contact form.

**This endpoint needs to be built on the Railway backend** (not in this repo) — it should
proxy `system`/`messages` to an LLM API (e.g. Anthropic's Messages API) server-side and
return `{ reply }`. **Never call an LLM API directly from the browser with an API key** —
that exposes the key to every visitor. Same CORS requirement as the leads endpoints
(`CORS_EXTRA_ORIGINS`). Consider the same per-IP rate limit as leads, plus a per-viewer cap
on chat turns to control LLM cost (previously documented as "15 questions/min per viewer"
as a UX-level target; not yet enforced anywhere).

**Chat questions are logged to the API.** Registrant details (first/last/email/org) are
captured at webinar submit and stored in state; each question is POSTed to
`/api/v1/public/leads/contact` (fire-and-forget) with `message` = `"[Webinar Q&A — <session title>] <question>"`
— or `"[Webinar Q&A — <session title> — <part label>] <question>"` when the track has more
than one part, so the logged lead shows which part prompted the question. So every question
a viewer asks becomes a contact lead attributed to the registrant — this part already
worked and is unrelated to the chat-reply bug above.

---

## Deep links (hash routes)

Append to the deployed page URL (the `#` matters; hash routing works on any static host):

| Link | Lands on |
|------|----------|
| `#webinar` or `#webinar-individual` | Webinar page, Individual track, Part 1 |
| `#webinar-agency` | Webinar page, Agency track, Part 1 |
| `#webinar-individual-partN` | Webinar page, Individual track, Part N |
| `#webinar-agency-partN` | Webinar page, Agency track, Part N |
| `#quiz` or `#quiz-individual` | Quiz, individual self-assessment tab |
| `#quiz-agency` | Quiz, agency audit tab |
| `#contact` | Contact form |
| `#home` or bare URL | Home |

The hash is read on load and updates live in the address bar as visitors navigate. Unknown
hash → falls back to home. Case-insensitive. A part-link still requires registering first
(it only pre-selects which part shows once they do); an out-of-range part number (e.g.
`-part9` on a 3-part track) clamps to that track's last valid part instead of erroring.

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
4. **Webinar video URLs** must be set in Tweaks (one per part per track) or a placeholder
   shows for that part. Currently all are blank.
5. **Part count is a code+Tweaks pair.** If a track's part count changes later, add/remove
   both the `webinarData[i].parts` entry and its matching Tweaks prop together — see the
   "Configurable settings" section above. They're linked only by the `videoProp` string on
   each part object, so a typo there silently shows the empty-video placeholder rather than
   erroring loudly.
6. **Hero image** is a drag-and-drop `<image-slot id="ds-hero">` — the user drops their own
   warm/diverse hero photo onto it and it persists. No stock photo is bundled (can't ship a
   licensed one).
7. **MCO logos as social proof** — ADDED as a muted strip below the hero ("Claims routed to
   every PA Medicaid MCO") with **placeholder** wordmarks + a trademark disclaimer. Swap in
   real logo files when available (see note below); the MCO list lives in the `mcos` array in
   the logic class.
8. **Turnstile bot protection (frontend wired, backend verification pending):** set
   `turnstileSiteKey` in Tweaks to turn the widgets on across all 4 lead forms. Backend must
   add `siteverify` checks on all 4 lead endpoints using a secret key before this actually
   blocks anything — see "Bot protection" under "Lead capture — API wiring" above.

---

## Editing notes

- The whole site is one Design Component (`DoulaShield.dc.html`). Edit via the DC tools; the
  logic class holds all content arrays (`stages`, `features`, `webinarData`, `quizQuestions`,
  `auditItems`) and handlers.
- No persistence: registration/quiz state is in-memory only. **Reload = clean reset.** No
  "already registered" lock; forms can be resubmitted freely.
