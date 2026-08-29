# BoardBuddy — Cloudflare deploy guide

Target: `boardbuddy.toppers.workers.dev` (Cloudflare Workers).
Auth is **username + password only** — no email links, no OAuth, no callback route.

---

## 0. Where do I get the values?

Sign in as the owner and open **Owner Panel → Cloud**. Every value below has a
copy button there, plus a "Copy all deploy variables" button that gives you the
whole block ready to paste.

Owner login (permanent):

| | |
| --- | --- |
| Username | `swastikbaniya` |
| Password | `swastik6852` |

Change the password later from the app; the owner + admin roles stay attached to
this account permanently.

---

## 1. Build-time variables (client)

These are inlined into the bundle at build time, so they must exist in the
Cloudflare build environment (Workers → Settings → Variables and Secrets):

| Variable | Where from |
| --- | --- |
| `VITE_SUPABASE_URL` | Owner Panel → Cloud |
| `VITE_SUPABASE_PUBLISHABLE_KEY` | Owner Panel → Cloud (publishable — safe in the browser) |
| `VITE_SUPABASE_PROJECT_ID` | Owner Panel → Cloud |

If they are missing, the app still renders in device-only mode (progress saved
locally) instead of crashing — but sign in/up will fail.

## 2. Runtime secrets (server side, only for payments + AI)

The app uses **one and the same Supabase project everywhere** (Lovable preview,
Lovable published site, and the Cloudflare Worker). The server functions that
create Razorpay orders and call the AI need these three values on the host:

| Variable | Notes |
| --- | --- |
| `SUPABASE_URL` | same value as `VITE_SUPABASE_URL` |
| `SUPABASE_PUBLISHABLE_KEY` | same value as `VITE_SUPABASE_PUBLISHABLE_KEY` |
| `SUPABASE_SERVICE_ROLE_KEY` | secret — never goes in the client bundle (alias `APP_SUPABASE_SERVICE_ROLE_KEY` also works) |

Current project: `https://ctbztladyklnuiifdlcs.supabase.co` (project id `ctbztladyklnuiifdlcs`).
Run `supabase/FULL_SETUP.sql` once in that project's SQL Editor before first use.

### 2a. Email confirmation OFF (required — fixes "Email not confirmed")

Accounts are created with a synthetic address `<username>@boardbuddy.app`, which
can never receive a confirmation mail. Do both of these once:

1. **Supabase Dashboard → Authentication → Sign In / Providers → Email →
   turn "Confirm email" OFF** (project `ctbztladyklnuiifdlcs`), then Save.
2. **SQL Editor → run `supabase/FIX_EMAIL_CONFIRM.sql`.** It adds a trigger that
   auto-confirms every `@boardbuddy.app` signup, a `confirm_signup_email()`
   repair function the app calls automatically, and backfills accounts that are
   already stuck on "Email not confirmed".

After this, sign up returns a session immediately and sign in works right away.


**Why these can't live in the Owner Panel:** the panel stores its values *inside*
the database, so the server needs the Supabase URL + key before it can read
anything. They must be host variables.

Optional host secrets that **override** the Owner Panel values (handy on
Cloudflare, and the safest way to fix a "Razorpay: Authentication failed"):

| Variable | Notes |
| --- | --- |
| `RAZORPAY_KEY_ID` | `rzp_live_…` / `rzp_test_…` — must match the secret below |
| `RAZORPAY_KEY_SECRET` | from the **same** Razorpay Dashboard → Settings → API Keys generation |
| `RAZORPAY_WEBHOOK_SECRET` | same value you set in Razorpay → Settings → Webhooks |

If these are set, the server uses them; otherwise it falls back to the values
saved in Owner Panel → Keys. The AI key still lives in the Owner Panel.


In Cloudflare: Workers → Settings → Variables and Secrets → add the three above
as **secrets** (plus the three `VITE_…` build variables from §1). Owner Panel →
Cloud has copy buttons for the public ones; the service-role key you copy from
your Supabase dashboard (Project Settings → API keys → `service_role`).

Fallback for a host without the service-role key: generate a **server access
token** and set it as `SERVER_ACCESS_TOKEN`; the server then reads keys through
the token-guarded `server_*` database functions. Owner Panel → Keys → *Server
status* always shows which mode the host is running in and whether it can
actually read the saved keys.

For AI on a self-hosted copy, save your own OpenAI / Gemini / OpenRouter key in
**Owner Panel → AI** — the built-in Lovable AI key only exists on Lovable
hosting.


## 3. Build & deploy

```bash
bun install
bun run build          # emits dist/ + dist/server/wrangler.json
npx wrangler deploy --name boardbuddy   # → boardbuddy.<your-subdomain>.workers.dev
```

- The Worker **name** decides the subdomain: name it `boardbuddy` and, with the
  account subdomain `toppers`, the URL becomes
  `https://boardbuddy.toppers.workers.dev`.
- Enable the `workers.dev` route for the Worker (Settings → Domains & Routes →
  Enable `workers.dev`).
- Or connect the GitHub repo in Cloudflare → Workers → Create → Import a
  repository, with build command `bun run build` and the variables from §1.

## 4. Razorpay webhook

After the first deploy, set the webhook in Razorpay → Settings → Webhooks:

```
https://boardbuddy.toppers.workers.dev/api/public/razorpay-webhook
```

Subscribe to `payment.captured`, `order.paid`, `payment.failed`,
`refund.processed`, and use the same secret you saved as
`razorpay_webhook_secret` in Owner Panel → Keys.

Use `rzp_test_*` keys while testing and switch to `rzp_live_*` when you go live —
the Keys tab shows which mode the saved keys are in.

## 5. Auth callbacks / cookies

- No OAuth or magic-link callbacks exist, so no redirect allow-list entries.
- The session lives in `localStorage`, so no cookie/domain config and no SSR
  session handling.
- `/reset-password` redirects to `/auth`, where recovery works with username +
  secret answer.
- Email confirmation is off (auto-confirm), so no "email not verified" state.

## 6. Post-deploy smoke test

1. `/auth` → sign in as `swastikbaniya`.
2. `/owner` → all tabs load (Dashboard, Keys, AI, Cloud …).
3. `/tutor` → ask one question, expect an answer.
4. `/subscribe` → start a checkout with Razorpay test keys and pay with a test
   card; the plan should turn active on `/profile`.

---

## Final Cloudflare checklist (verified 27 Aug 2026)

Set these in Workers → Settings → Variables and Secrets. Use the exact same
Supabase project everywhere (`ctbztladyklnuiifdlcs`):

| Variable | Type | Value |
| --- | --- | --- |
| `VITE_SUPABASE_URL` | build var | https://ctbztladyklnuiifdlcs.supabase.co |
| `VITE_SUPABASE_PUBLISHABLE_KEY` | build var | publishable key |
| `VITE_SUPABASE_PROJECT_ID` | build var | ctbztladyklnuiifdlcs |
| `SUPABASE_URL` | var | same as VITE_SUPABASE_URL |
| `SUPABASE_PUBLISHABLE_KEY` | var | same as VITE_SUPABASE_PUBLISHABLE_KEY |
| `SUPABASE_SERVICE_ROLE_KEY` | secret | service role key (alias `APP_SUPABASE_SERVICE_ROLE_KEY` also works) |
| `RAZORPAY_KEY_ID` | secret | live key id |
| `RAZORPAY_KEY_SECRET` | secret | live key secret |
| `RAZORPAY_WEBHOOK_SECRET` | secret | same value pasted in Razorpay → Webhooks |

With the service-role key present the Owner Panel → Keys page shows
**"Server can read the saved keys." / Access mode: service_role**, and the keys
saved in the panel are used automatically. The `RAZORPAY_*` host secrets simply
override them, so payments keep working even if the database keys are cleared.

Razorpay webhook URL: `https://<your-domain>/api/public/razorpay-webhook`
(events: payment.captured, order.paid, payment.failed, refund.processed).
