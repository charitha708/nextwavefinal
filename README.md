# Next Wave Project Lab

A Next Wave initiative: an AI-driven software project assessment platform and the **Next Wave 60-Minute AI Project Making Workshop** funnel.

Students submit a public GitHub repository, take a timed open-ended technical exam generated from the actual code, receive a scored report, and convert into workshop registrations with full campaign attribution, email automation, and an admin dashboard.

## Implemented components (not yet production-verified)

The project contains the components listed below, but their full workflow has not been verified in a live environment. The frontend requires a valid Supabase project URL and anon key. Once those are configured and the Edge Functions are deployed, it can run in mock-AI development mode without an AI provider key. Real AI grading, email delivery, and Google Sheets writes require server-side credentials. Missing integrations are reported as incomplete—not as successful sends or syncs.

| Feature | Status without keys | Status with keys |
|---|---|---|
| Landing, workshop blog, privacy, terms pages | Implemented; requires frontend build and Supabase config | Same |
| Campaign attribution (utm + ref) capture & preservation | Works | Works |
| Analytics events → Supabase + Sheets | Requires configured Supabase Edge Functions | Real event rows and sync jobs |
| Assessment submit + GitHub repo analysis | Implemented; requires deployed Edge Function and public repo | Same |
| AI question generation + grading | Mock provider | Real LLM |
| Timed exam, autosave, refresh restore | Implemented through server actions; requires deployed Edge Functions and end-to-end verification | Same |
| Results report with category breakdown + roadmap | Works (mock scoring) | Works (real scoring) |
| Registration + duplicate protection | Implemented; requires deployed Edge Function and migrations | Same |
| Confirmation email | Not sent; explicitly marked failed | Sent via Resend |
| Google Sheets sync | Logged to server (dev) | Synced via Apps Script |
| Admin dashboard (auth + metrics + filters + export) | Implemented; requires Supabase Auth, allowlist, migrations and deployed function | Same |
| Email/sheet retry controls | Works (dev mode) | Works (real retries) |

## Tech stack

- **Frontend:** Vite + React + TypeScript + Tailwind CSS + React Router
- **Backend:** Supabase Edge Functions (Deno)
- **Database:** Supabase Postgres (RLS enabled on all tables)
- **AI:** Modular provider interface — OpenAI, Anthropic, Gemini, or mock
- **Email:** Resend (with dev-mode logging)
- **Sheets:** Google Apps Script Web App (with dev-mode skip)

## Getting started

```bash
npm ci
npm run dev
```

Create `.env` from `.env.example`, then replace the placeholder `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY` with values from your Supabase project. The uploaded ZIP does not include actual credentials. Keep `.env` out of Git. All non-`VITE_` values in the example are server-side references: configure them as Supabase Edge Function secrets, not as frontend-exposed variables.

## Environment variables

Copy `.env.example` to `.env`. The Supabase values are placeholders and must be replaced. Server-side integration values must be set in Supabase Edge Function secrets:

| Variable | Purpose | Required? |
|---|---|---|
| `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` / `GEMINI_API_KEY` | Real AI examiner; configure as Edge Function secrets | No (mock runs otherwise) |
| `RESEND_API_KEY` | Real email sending; configure as an Edge Function secret | No (email is marked not sent otherwise) |
| `RESEND_FROM` | Sender address | With Resend |
| `RESEND_WEBHOOK_SECRET` | Webhook signature verification | With Resend |
| `SHEETS_WEBAPP_URL` | Google Sheets sync; configure as an Edge Function secret | Required for real sync |
| `SHEETS_WEBAPP_SECRET` | Shared secret for Apps Script receiver; configure as an Edge Function secret | Required for real sync |
| `WORKSHOP_DATE` / `WORKSHOP_TIME` / `WORKSHOP_JOIN_URL` | Confirmation email content | No (defaults to "To be announced") |

## Database

Apply every SQL file in `supabase/migrations/` to your Supabase project before deployment. Do not assume a migration in the repository has already been applied remotely. Tables:

- `assessments` — project submission + exam metadata + report
- `assessment_questions` — AI-generated questions
- `assessment_answers` — autosaved answers (one per question)
- `registrations` — workshop registrations (unique per workshop+email)
- `analytics_events` — funnel + campaign events (first-party, session-based)
- `email_outbox` — email queue with delivery state
- `integration_jobs` — email + sheet sync job tracking
- `admins` — admin allowlist (email-based)

All tables have RLS enabled. The later hardening migration revokes browser access to application tables; validated Edge Functions use the service role and enforce session/email ownership server-side. Apply all migrations in order.

## Admin setup

To access the admin dashboard at `/admin`:

1. Create an admin user in Supabase Auth (email/password).
2. Add their email to the `admins` table:
   ```sql
   INSERT INTO admins (email) VALUES ('you@example.com');
   ```
3. Sign in at `/admin`.

You can run this SQL via the Supabase dashboard or ask the assistant to run it for you.

## Google Sheets setup (optional)

1. Create a Google Sheet and open **Extensions → Apps Script**.
2. Copy `automation/google-apps-script/Code.gs` into the Apps Script editor.
3. In **Project Settings → Script Properties**, add `SPREADSHEET_ID` and `SHEETS_WEBAPP_SECRET`. Use a long random secret and set the exact same value as the Supabase Edge Function secret.
4. Run `setupSheets()` once from the editor and authorize it. This creates the three tabs and headers.
5. Deploy as a Web App, execute as **Me**, with access set to **Anyone** (the shared secret is still required in the POST body). Copy the deployment URL.
6. Set Supabase secrets `SHEETS_WEBAPP_URL` and `SHEETS_WEBAPP_SECRET`, then redeploy `register-workshop`.
7. A registration row is only marked synced when the receiver returns HTTP success and `{ ok: true }`. The Apps Script uses registration IDs to avoid duplicate rows. Keep the secret private.

## Resend setup (optional)

1. Create a Resend account and verify your sending domain.
2. Set `RESEND_API_KEY` and `RESEND_FROM` (e.g. `Next Wave <workshop@yourdomain.com>`).
3. Configure the Resend webhook to point to the `resend-webhook` edge function and set `RESEND_WEBHOOK_SECRET`.

## Campaign links

Use tracked links to attribute registrations:

```
https://YOUR_DOMAIN/?utm_source=whatsapp&utm_medium=community&utm_campaign=ai_workshop&utm_content=community_a
https://YOUR_DOMAIN/?utm_source=instagram&utm_medium=social&utm_campaign=ai_workshop&utm_content=reel_01
https://YOUR_DOMAIN/?utm_source=telegram&utm_medium=community&utm_campaign=ai_workshop&utm_content=group_a
```

UTM parameters and referral codes are stored in browser storage and submitted with registration. Use distinct links for each community/post. Verify first-touch versus last-touch behavior before relying on campaign reports.

## Scripts

- `npm run dev` — start dev server
- `npm run build` — production build
- `npm run typecheck` — TypeScript check
- `npm run lint` — ESLint

Run `npm run build`, `npm run typecheck`, and `npm run lint` after dependencies are installed. These checks have not been verified in this workspace because dependency installation failed.

## Deployment

1. Copy `.env.example` to `.env` and set the two `VITE_SUPABASE_*` values for the frontend build.
2. Apply every SQL migration in `supabase/migrations/` in filename order.
3. Set Edge Function secrets in Supabase Dashboard → Edge Functions → Secrets (or the Supabase CLI): `RESEND_API_KEY`, `RESEND_FROM`, `RESEND_WEBHOOK_SECRET`, `SHEETS_WEBAPP_URL`, `SHEETS_WEBAPP_SECRET`, and one of `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, or `GEMINI_API_KEY` if real AI grading is required. Also configure `WORKSHOP_DATE`, `WORKSHOP_TIME`, and `WORKSHOP_JOIN_URL` when finalized.
4. Deploy functions: `supabase functions deploy ai-examiner`, `register-workshop`, `registration-status`, `track-event`, `admin-api`, and `resend-webhook`.
5. Configure the Resend webhook URL as `https://YOUR_PROJECT_REF.supabase.co/functions/v1/resend-webhook` and use the webhook signing secret generated by Resend.
6. Install dependencies, then run `npm run build`, `npm run typecheck`, and `npm run lint`.
7. Deploy the frontend build to Vercel, Netlify, or Cloudflare Pages. Configure SPA fallback/rewrite to `index.html` so routes such as `/register` work after refresh.
8. Create an admin user in Supabase Auth and insert that email into `public.admins` using the SQL shown above.
9. Register a test address you control. Confirm the registration row, Resend provider message ID, actual inbox receipt, webhook-updated delivery status, Google Sheets row, campaign-event rows, and admin retry behavior before distributing campaign links.

Frontend `VITE_*` variables belong in the hosting provider. All provider keys and shared secrets belong only in Supabase Edge Function secrets. Never put Resend, LLM, service-role, or Apps Script shared-secret values in frontend variables.

## Important verification status

- Without an AI provider key, assessment question generation and grading use the built-in mock provider; this is not a real AI evaluation.
- A missing Resend key is recorded as failed/not sent. Real delivery requires a verified sender domain and webhook configuration.
- Google Sheets sync requires both `SHEETS_WEBAPP_URL` and `SHEETS_WEBAPP_SECRET`. The supplied Apps Script handles idempotent registration/event rows and computes a campaign summary, but must be deployed and tested against your own spreadsheet.
- The assessment actions now use server-side ownership and expiry checks. Apply the new table-privilege migration and deploy all Edge Functions before testing.
- A successful local syntax parse is not a substitute for a full build, database migration, live email receipt, or live spreadsheet write.
