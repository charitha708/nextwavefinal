# Next Wave Project Lab — fastest path to a verified launch

This project has code for the main workflow, but it is **not yet production-verified**. Do these steps in order; do not distribute a public campaign link until the live checks at the end pass.

## 1. Configure the frontend locally

1. Copy `.env.example` to `.env`.
2. In Supabase → Project Settings → API, copy the Project URL and anon/public key into `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY`.
3. Keep the service-role key, AI keys, Resend key, webhook secret, and Sheets secret out of `VITE_*` variables and out of Git.
4. Run:

   ```bash
   npm ci
   npm run typecheck
   npm run lint
   npm run build
   npm run dev
   ```

If `npm ci` fails, resolve the network/package-registry issue first. A successful syntax parse is not a substitute for the full build.

## 2. Apply database migrations

In Supabase → SQL Editor, run every file in `supabase/migrations/` in filename order:

1. `20261027180329_001_core_schema.sql`
2. `20261028100000_002_analytics_jobs.sql`
3. `20261028110000_003_lock_down_public_tables.sql`

The third migration removes direct browser access to application tables. Do not skip it.

## 3. Deploy Edge Functions

Install/use the Supabase CLI, authenticate, and link the project:

```bash
supabase login
supabase link --project-ref YOUR_PROJECT_REF
```

Set server-side secrets in Supabase Dashboard → Edge Functions → Secrets. At minimum:

- `SUPABASE_URL`
- `SUPABASE_SERVICE_ROLE_KEY`
- One AI provider key: `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, or `GEMINI_API_KEY`
- `RESEND_API_KEY`
- `RESEND_FROM`
- `RESEND_WEBHOOK_SECRET`
- `SHEETS_WEBAPP_URL`
- `SHEETS_WEBAPP_SECRET`
- Optional but recommended when finalized: `WORKSHOP_DATE`, `WORKSHOP_TIME`, `WORKSHOP_JOIN_URL`

Deploy the functions:

```bash
supabase functions deploy ai-examiner
supabase functions deploy register-workshop
supabase functions deploy registration-status
supabase functions deploy track-event
supabase functions deploy admin-api
supabase functions deploy resend-webhook --no-verify-jwt
```

`resend-webhook` uses Resend's signed webhook verification rather than a Supabase user JWT, so deploy it with `--no-verify-jwt`. Keep its signing secret configured.

## 4. Configure Google Sheets

1. Create a Google Sheet.
2. In Extensions → Apps Script, paste `automation/google-apps-script/Code.gs`.
3. In Apps Script Project Settings → Script Properties, set `SPREADSHEET_ID` and `SHEETS_WEBAPP_SECRET`.
4. Run `setupSheets()` once and authorize it.
5. Deploy as a Web App, execute as you, with access set to Anyone. The shared secret is checked on every POST.
6. Copy the Web App URL into the Supabase `SHEETS_WEBAPP_URL` secret. Set the exact same shared secret as `SHEETS_WEBAPP_SECRET` in Supabase.

## 5. Configure email and admin access

1. Verify a sender domain in Resend and set `RESEND_FROM` to an address on that verified domain.
2. Configure the Resend webhook URL: `https://YOUR_PROJECT_REF.supabase.co/functions/v1/resend-webhook`.
3. Use the webhook signing secret from Resend as `RESEND_WEBHOOK_SECRET`.
4. Create an admin user in Supabase Auth.
5. Using a trusted SQL/editor session, allowlist that email:

   ```sql
   INSERT INTO public.admins (email) VALUES ('your-admin-email@example.com');
   ```

Do not expose the service-role key or add it to frontend environment variables.

## 6. Deploy the frontend

Deploy to Vercel, Netlify, or another static host. Configure the two `VITE_SUPABASE_*` variables in the host's build environment. Configure an SPA rewrite/fallback to `index.html` so direct refreshes on `/assessment`, `/register`, and other routes work.

## 7. Required live smoke test

Use a public repository and an email address you control. Confirm each step, not just the UI:

- [ ] Landing page loads on desktop and mobile.
- [ ] A tracked UTM link preserves source/campaign through registration.
- [ ] A public GitHub repo is retrieved and the question set is non-empty.
- [ ] Timer begins only after start, answers save, refresh restores, expiry is enforced, and a report is persisted.
- [ ] A real AI key produces a valid AI-generated report (mock mode is not a production pass).
- [ ] Registration appears once in Supabase; duplicate submission is handled safely.
- [ ] A real confirmation email arrives in the inbox.
- [ ] Resend webhook changes the email status to delivered/bounced/failed as appropriate.
- [ ] Registration and campaign events appear in the correct Sheets tabs; Campaign Summary updates.
- [ ] `/admin` rejects anonymous access and an allowlisted admin can view real metrics and retry failed jobs.
- [ ] `npm run typecheck`, `npm run lint`, and `npm run build` all pass.

## Known remaining work

Rate limiting and bot protection are not fully implemented in this snapshot. Add them before a large public campaign. The code has not been deployed or tested against live Supabase, AI, Resend, Sheets, or hosting credentials in this environment.
