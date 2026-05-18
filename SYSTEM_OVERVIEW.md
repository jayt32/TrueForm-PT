# TrueForm Physical Therapy — System Overview

**Last updated:** May 2026  
**Purpose:** Full handoff reference for the TrueForm PT website and its backend infrastructure.

---

## Project Summary

TrueForm Physical Therapy is a concierge PT private practice based in Tampa, FL, run by Alisha Trivedi, DPT. This document covers the website, backend, email notifications, deployment pipeline, and security model.

---

## Live URLs

| Environment | URL |
|---|---|
| Production | https://trueform-pt.com |
| www redirect | https://www.trueform-pt.com → trueform-pt.com |
| GitHub repo | https://github.com/jayt32/TrueForm-PT |
| Vercel dashboard | https://vercel.com (project: `true-form-pt`) |

---

## File Structure

```
Website Builder/
├── index.html              ← Main website (single file, all styles inline)
├── flyer.html              ← Print/referral flyer (680×880px, letter size)
├── SYSTEM_OVERVIEW.md      ← This document
├── CLAUDE.md               ← AI assistant configuration (internal)
├── og-card.html            ← Source file used to generate the OG preview image
├── .gitignore
├── website_resources/
│   ├── trueform_logo2.png          ← Primary logo (used in nav, footer, flyer)
│   ├── Trueform_logo.png           ← Alternate logo asset
│   ├── alisha_trueform_profpic.jpg ← Therapist headshot (About section + flyer)
│   ├── tampa_skyline.png           ← Hero background image
│   ├── og-image.png                ← Open Graph preview card (1200×630px)
│   └── flyer_example.png           ← Reference image (not used in production)
├── serve.mjs               ← Local dev server (node serve.mjs → localhost:3000)
├── screenshot.mjs          ← Puppeteer screenshot utility (dev only)
├── package.json            ← Node deps (puppeteer for local dev only)
└── node_modules/           ← Local dev only, not in git
```

**What's in the repo (GitHub):** `index.html`, `flyer.html`, `website_resources/` (production assets only), `SYSTEM_OVERVIEW.md`, `.gitignore`

**Not in the repo:** `node_modules/`, `temporary screenshots/`, `og-card.html`, screenshot/serve utilities, reference images

---

## How the Contact Form Works

```
Visitor fills out contact form on website
              ↓
   Browser POSTs to Supabase Edge Function
   (notify-inquiry) via fetch()
              ↓                    ↓
   Saves to database         Sends email via Resend
              ↓                    ↓
   contact_submissions        alisha@trueform-pt.com
       table                      inbox
```

The edge function runs on Supabase's servers. The browser never touches the database directly, and no API keys are exposed in the HTML.

---

## The Website (`index.html`)

**Tech stack:** Pure HTML, inline CSS, Tailwind CSS via CDN, vanilla JavaScript  
**Fonts:** Playfair Display (headings) + DM Sans (body) via Google Fonts  
**Maps:** Leaflet.js (Tampa coverage polygon)  
**No build step required** — open the file in a browser or serve it statically.

### Sections (top to bottom)
1. **Nav** — Sticky, cream background, logo + links (Services, Areas Served, About, Contact)
2. **Hero** — Tampa skyline background, headline, service type icons, CTA button
3. **CTA Band** — Dark olive strip with "Send a Message" button
4. **Services** — 5-card grid: Gym, Office, Condo, Outdoor, Home Visits
5. **What We Treat** — 2×2 grid: Post-Op Rehab, Chronic Pain, Balance & Fall Prevention, Return-to-Function
6. **Why TrueForm** — 3-column differentiators
7. **Areas Served** — Location list + Leaflet map (Tampa coverage polygon)
8. **About** — Alisha's bio, credentials, headshot
9. **Footer** (`id="contact"`) — Logo, nav links, "Send a Message" button

### Brand Colors
| Token | Hex | Usage |
|---|---|---|
| Cream | `#F5F0E8` | Page background, nav |
| Cream Alt | `#EDE7D9` | Alternate section backgrounds |
| Olive | `#4B5535` | Accents, bullet dots, map polygon |
| Dark | `#2A2E22` | CTA band, footer background |
| Charcoal | `#1A1A17` | Body text, headings |

### Contact Form (JavaScript)
The form is a modal overlay triggered by `openContactModal()`. On submit, `submitContactForm()` POSTs to the Supabase edge function. Fields: Name (required), Email (required), Phone (optional), Message (required).

The edge function URL is the only backend reference in the HTML:
```
https://thrjdvkevrmajzclfxht.supabase.co/functions/v1/notify-inquiry
```

### Open Graph / Link Previews
Meta tags in `<head>` control how the URL appears when shared via SMS, iMessage, email, or social media:
- **Image:** `https://trueform-pt.com/website_resources/og-image.png` (1200×630px branded card)
- **Title:** TrueForm Physical Therapy
- **Description:** Concierge physical therapy in Tampa, FL

---

## The Flyer (`flyer.html`)

A single-page print/referral flyer designed to be printed on standard 8.5×11 letter paper or shared as a PDF screenshot. Fixed at 680×880px.

**Sections:** Header (logo + badge) → Hero (title + headshot) → What We Treat (4 items) → Ideal Patients → Credentials strip → Footer (email, phone, website)

**Contact info on flyer:**
- Email: `alisha@trueform-pt.com`
- Phone: `(817) 879-3316`
- Website: `trueform-pt.com`

---

## Supabase

**Project ref:** `thrjdvkevrmajzclfxht`  
**Dashboard:** https://supabase.com/dashboard/project/thrjdvkevrmajzclfxht

### Database — `contact_submissions` table

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key, auto-generated |
| `name` | text NOT NULL | Max 200 chars |
| `email` | text NOT NULL | Max 320 chars |
| `phone` | text | Optional, max 30 chars |
| `message` | text NOT NULL | Max 5,000 chars |
| `created_at` | timestamptz | Auto-set on insert |

**To view submissions:** Supabase dashboard → Table Editor → contact_submissions

### Edge Function — `notify-inquiry`

A serverless Deno function (deployed at version 3) that:
1. Receives the form POST from the browser
2. Validates the payload
3. Inserts the row into `contact_submissions` using the service role key (bypasses RLS)
4. Calls Resend to send an email notification to Alisha
5. Returns `{ success: true }` — even if the email fails, the DB save is treated as the source of truth

**Runtime:** Deno (Supabase Edge Functions use Deno, not Node.js)  
**JWT verification:** Disabled (`verify_jwt: false`) — this is a public form endpoint  
**Secrets stored in Supabase (not in code):**
- `RESEND_API_KEY`
- `SUPABASE_URL` (injected automatically)
- `SUPABASE_SERVICE_ROLE_KEY` (injected automatically)

---

## Resend (Email)

**Purpose:** Sends email notifications to Alisha when a contact form is submitted  
**Sends from:** `noreply@trueform-pt.com`  
**Sends to:** `alisha@trueform-pt.com`  
**Dashboard:** https://resend.com (logged in as Jefferson account)

**Domain verification:** `trueform-pt.com` is verified in Resend. Three DNS records were added in Squarespace to prove domain ownership:
- MX record (send subdomain)
- Two TXT records (DKIM + SPF)

The API key lives in Supabase → Edge Functions → notify-inquiry → Secrets → `RESEND_API_KEY`. It is never in the HTML or the GitHub repo.

---

## Security Model

### What's protected and how

**No API keys in the website code**  
The HTML only contains the edge function URL — a public endpoint. All sensitive credentials (Resend API key, Supabase service role key) live as Supabase secrets inside the edge function.

**Row Level Security (RLS)**  
RLS is enabled on `contact_submissions`. There are zero public-facing policies — no visitor can read, update, or delete any submissions through the public API. The only way data is written is through the edge function using the service role key server-side.

**Input length constraints**  
Database-level constraints prevent oversized payloads:
- Name: 1–200 characters
- Email: 3–320 characters
- Phone: up to 30 characters
- Message: 1–5,000 characters

**No sensitive data stored**  
The table only holds voluntary contact form submissions. No passwords, payment info, or user accounts exist.

**Alisha's email not in page source**  
`alisha@trueform-pt.com` does not appear anywhere in the HTML. Contact is form-only.

### What is intentionally public
- The Supabase edge function URL (it's a public form endpoint — no key required to call it)
- The Supabase project ref in the URL (this is normal and not a secret)

---

## Deployment

### Stack
```
Local editing (index.html)
        ↓
git push → GitHub (jayt32/TrueForm-PT)
        ↓
Vercel detects push → auto-deploys within ~30 seconds
        ↓
Live at trueform-pt.com
```

### DNS (managed in Squarespace)
The domain `trueform-pt.com` was originally on Google Domains, which was acquired by Squarespace in 2023. DNS is managed at `account.squarespace.com`.

**Vercel DNS records (added in Squarespace → Custom Records):**
| Type | Name | Value |
|---|---|---|
| A | @ | `216.198.79.1` |
| CNAME | www | `0ced5b596ba65d5c.vercel-dns-017.com` |

**Email DNS records (Resend + Google Workspace — do not remove):**
| Type | Name | Purpose |
|---|---|---|
| MX | send | Resend sending |
| TXT | google_domainkey | DKIM for Resend |
| TXT | resend_domainkey | Resend domain verification |
| TXT | send | SPF for Resend |
| TXT | @ | SPF for Google |
| MX | @ | Google Workspace Gmail |

### Local Development
```bash
# Serve locally
node serve.mjs          # → http://localhost:3000

# Take a screenshot
node screenshot.mjs http://localhost:3000

# Access from phone (same WiFi)
http://192.168.1.104:3000
```

---

## Credentials Reference

| Thing | Where to find it |
|---|---|
| Website + flyer files | `/Users/jaytrivedi/Desktop/Website Builder/` |
| GitHub repo | github.com/jayt32/TrueForm-PT |
| Vercel dashboard | vercel.com → project `true-form-pt` |
| Supabase dashboard | supabase.com → project `thrjdvkevrmajzclfxht` |
| Form submissions | Supabase → Table Editor → contact_submissions |
| Edge function code | Supabase → Edge Functions → notify-inquiry |
| Resend API key | Supabase → Edge Functions → notify-inquiry → Secrets |
| Resend dashboard | resend.com (Jefferson account) |
| Domain DNS | account.squarespace.com → trueform-pt.com → DNS |
| Google Workspace | admin.google.com (manages alisha@trueform-pt.com) |

---

## Known Limitations / Future Considerations

- **No booking system** — the contact form sends an email inquiry only; scheduling is handled manually by Alisha
- **No CMS** — content changes require editing `index.html` directly and pushing to GitHub
- **Rate limiting** — the edge function has no rate limiting implemented; for a higher-traffic site this should be added (e.g. via Supabase's built-in rate limiting or an API gateway)
- **No spam filtering** — no CAPTCHA or honeypot on the contact form; could be added if spam becomes an issue
- **Flyer is HTML only** — `flyer.html` is designed to be printed from the browser or screenshotted; it is not automatically generated as a PDF
