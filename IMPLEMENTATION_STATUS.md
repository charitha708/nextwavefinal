# Next Wave Project Lab — implementation status

This is an implementation snapshot, not a production certification. The app uses the existing Vite + React + TypeScript stack with Supabase Edge Functions; it was not rebuilt into Next.js because the uploaded project already had a working structure to preserve.

## Changes in this working copy

- Kept the existing application and added/retained all primary routes: landing, assessment, results, workshop article, registration, registration success, admin, privacy, and terms.
- Kept the Supabase schema migrations and six Edge Functions for AI assessment, registration, registration status, analytics events, admin operations, and Resend webhooks.
- Fixed a duplicate `getSessionId` import in the results page that could cause a TypeScript build error.
- Fixed the no-LLM-key development path: the mock provider now produces a clearly labeled development assessment instead of returning an empty question array and failing to create an exam.
- Added schema checks for AI-generated questions. A configured AI provider returning malformed or insufficient questions now produces a visible retryable error instead of silently replacing them with generic questions.
- Added structured validation for AI grading output. Invalid/incomplete reports no longer silently fall back to heuristic mock scoring while claiming to be real AI evaluation.
- Added server-side allowlist validation for assessment duration, category, experience level, and session ID length.
- Made manual assessment submission flush all locally entered answers before grading; the server remains authoritative about expiry.
- Added explicit error handling if a generated report cannot be persisted or another request finalizes the assessment first.
- Changed the admin's `totalVisits` metric to count landing-page views rather than every analytics event.
- Expanded `.gitignore` to ignore `.env.*` files while allowing `.env.example` to remain tracked.
- Kept the Google Apps Script receiver idempotent for registration/event IDs and the email/sheet status UI honest about incomplete integrations.

## Checks actually run in this working copy

- TypeScript syntax transpilation: 27 `.ts`/`.tsx` application and Edge Function source files parsed with no syntax diagnostics. The declaration file `src/vite-env.d.ts` was excluded from transpilation because it is a declaration-only file.
- Duplicate module-import scan: passed; no duplicate module imports remain in the scanned source files.
- Google Apps Script syntax check with Node: passed.
- `npm run build`: **blocked / not verified** because Vite is not installed in the environment.
- `npm run typecheck`: **blocked / not verified** because dependencies such as React, React Router, and their types are unavailable. The compiler reports missing modules/JSX types rather than a usable project typecheck.
- Dependency installation: `npm ci` timed out; offline installation also failed because required packages are not cached. This is an environment/network limitation, not evidence that the application builds.
- Supabase migrations: not applied to a live project.
- Edge Functions: not deployed or exercised against live Supabase credentials.
- AI: no live provider key was supplied; mock mode is for development only and must not be presented as real AI evaluation.
- Email: no real email was sent or received in this session. Resend sender-domain verification and webhook setup remain unverified.
- Google Sheets: Apps Script syntax passed, but the script was not deployed and no live row was verified.
- Production website: not deployed or verified.

## Required before real student traffic

1. Install dependencies in a network-enabled environment using `npm ci`, then run `npm run typecheck`, `npm run lint`, and `npm run build`; resolve any remaining diagnostics.
2. Configure `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY` in the local `.env` and hosting settings.
3. Apply all migrations in `supabase/migrations/` in filename order, including `20261028110000_003_lock_down_public_tables.sql`.
4. Configure Supabase Edge Function secrets: `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, one AI provider key (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, or `GEMINI_API_KEY`), `RESEND_API_KEY`, `RESEND_FROM`, `RESEND_WEBHOOK_SECRET`, `SHEETS_WEBAPP_URL`, `SHEETS_WEBAPP_SECRET`, and optional `WORKSHOP_DATE`, `WORKSHOP_TIME`, `WORKSHOP_JOIN_URL`.
5. Deploy the Apps Script, set `SPREADSHEET_ID` and `SHEETS_WEBAPP_SECRET` in Script Properties, run `setupSheets()`, and deploy as a web app accessible to the Edge Function.
6. Deploy the six Edge Functions: `ai-examiner`, `register-workshop`, `registration-status`, `track-event`, `admin-api`, and `resend-webhook`.
7. Verify the Resend sending domain, configure the webhook URL, and verify the webhook secret. Provider acceptance (`sent`) is not proof of inbox delivery (`delivered`).
8. Create an admin account and add its email to `public.admins` using a trusted SQL/editor session. Never expose the service-role key in the browser.
9. Add rate limiting and bot protection to public endpoints before a broad campaign launch; these are not fully implemented in this snapshot.
10. Run a live end-to-end test: tracked campaign URL → repository analysis → timed exam → saved answers → scored report → workshop article → registration → database row → received email → delivery webhook → spreadsheet rows/summary → admin view and retry controls.

## Important caveats

- The project is not production-ready until dependency checks, migrations, endpoint abuse protection, and live integration tests pass.
- Without an AI provider key, the assessment uses deterministic mock questions and heuristic scoring; it is clearly marked as development mode.
- First-touch attribution is stored in browser local storage and session identity in session storage. Cross-device attribution is not guaranteed.
- Some external integrations can be implemented but cannot be verified without the user's Supabase, AI provider, Resend, Google Apps Script, and hosting configuration.
