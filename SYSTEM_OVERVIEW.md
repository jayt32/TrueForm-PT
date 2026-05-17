# TrueForm PT — System Overview

## How Everything Connects

```
Visitor fills out form on website
        ↓
Supabase Edge Function (notify-inquiry)
        ↓                    ↓
Saves to database     Sends email via Resend
        ↓                    ↓
contact_submissions   alisha@trueform-pt.com
    table               inbox (Gmail)
```

---

## The Website

**File:** `index.html` (single file, all styles inline)
**Hosted locally:** `http://localhost:3000` via `node serve.mjs`
**Future hosting:** GitHub → Vercel (see Deployment section below)

The website is a static HTML page — no server needed to display it. All the backend work (database, email) happens through Supabase, which the page calls directly from the visitor's browser.

---

## Supabase

**Project:** `thrjdvkevrmajzclfxht`
**Dashboard:** https://supabase.com/dashboard/project/thrjdvkevrmajzclfxht

Supabase handles two things: the database and the edge function.

### Database — `contact_submissions` table

Every form submission is stored here with these fields:
- `id` — unique identifier (auto-generated)
- `name` — visitor's name
- `email` — visitor's email
- `phone` — visitor's phone (optional)
- `message` — their message
- `created_at` — timestamp of submission

**To view submissions:** Supabase dashboard → Table Editor → contact_submissions

### Edge Function — `notify-inquiry`

A small serverless function that runs on Supabase's servers (not the website). When someone submits the contact form:
1. The browser sends the form data to this function
2. The function saves it to the database
3. The function calls Resend to send an email notification to Alisha

Because this runs server-side, sensitive keys (Resend API key, database service key) are never visible to anyone visiting the website.

---

## Resend

**Dashboard:** https://resend.com
**Purpose:** Sends email notifications to Alisha when someone submits the contact form
**Verified domain:** `trueform-pt.com` (DNS records added via Squarespace)
**Sends from:** `noreply@trueform-pt.com`
**Sends to:** `alisha@trueform-pt.com`

Resend required the domain `trueform-pt.com` to be verified via DNS records before it would send emails from that address. Those records were added in Squarespace (where the domain is now managed after Google sold Google Domains to Squarespace in 2023).

**API key location:** Supabase dashboard → Edge Functions → notify-inquiry → Secrets → `RESEND_API_KEY`

---

## Security

### What's protected and how

**No API keys in the website code**
The only thing visible in the page source is the Supabase edge function URL. No API keys, no service credentials. All sensitive keys live inside the edge function as Supabase secrets.

**Database row-level security (RLS)**
RLS is enabled on the `contact_submissions` table. There are zero public-facing policies — meaning no visitor can read, update, or delete any submissions using the public API key. The only way data gets written to the table is through the edge function, which uses a server-side service key that bypasses RLS in a controlled way.

**Input length limits**
Database constraints prevent abuse:
- Name: 1–200 characters
- Email: 3–320 characters
- Phone: up to 30 characters
- Message: 1–5,000 characters

**No sensitive data stored**
The table only holds contact form submissions — no passwords, no payment info, no user accounts. There is nothing valuable to steal.

**Email not exposed in page source**
Alisha's email (`alisha@trueform-pt.com`) does not appear anywhere in the website HTML. Contact goes through the form only.

**Why this site is lower risk than most**
The common Supabase security vulnerabilities (subscription manipulation, rate limit abuse, user data leakage) don't apply here because there are no user accounts, no subscriptions, and no sensitive personal data beyond what visitors voluntarily submit.

---

## Deployment (GitHub + Vercel)

*This step hasn't been done yet — this is the plan for when the site is ready to go live.*

### GitHub
- The project files (`index.html`, assets, etc.) get pushed to a GitHub repository
- GitHub acts as version control — every change is tracked and reversible
- Vercel watches the GitHub repo for changes

### Vercel
- Vercel hosts the website publicly on the internet
- When a change is pushed to GitHub, Vercel automatically re-deploys the site within seconds
- Vercel provides a free URL (e.g. `trueform-pt.vercel.app`) and can be connected to `trueform-pt.com` so the site lives at the real domain

### The relationship
```
You edit index.html locally
        ↓
Push to GitHub
        ↓
Vercel detects the change and redeploys automatically
        ↓
Live site at trueform-pt.com updates within ~30 seconds
```

The contact form will work identically whether the site is on localhost or live on Vercel — it calls Supabase directly from the browser regardless of where the HTML is hosted.

---

## Credentials & Where Things Live

| Thing | Where to find it |
|---|---|
| Website file | `/Users/jaytrivedi/Desktop/Website Builder/index.html` |
| Supabase dashboard | supabase.com → project `thrjdvkevrmajzclfxht` |
| Form submissions | Supabase → Table Editor → contact_submissions |
| Edge function | Supabase → Edge Functions → notify-inquiry |
| Resend API key | Supabase → Edge Functions → notify-inquiry → Secrets |
| Resend dashboard | resend.com (logged in as Jefferson account) |
| Domain DNS | account.squarespace.com → trueform-pt.com → DNS |
| Google Workspace | admin.google.com (manages alisha@trueform-pt.com inbox) |
