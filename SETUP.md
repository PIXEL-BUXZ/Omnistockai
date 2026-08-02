# OmniStock — Supabase Setup Guide

OmniStock now uses **Supabase** for authentication and live cloud sync. Real
accounts are created via Supabase Auth, and the entire shared retail state
(stores, products, inventory, orders, logs, team directory) syncs to the cloud
in real time across every signed-in device and team member.

## 1. Create the database schema

1. Open your Supabase dashboard → **SQL Editor** → **New query**.
2. Paste the entire contents of [`supabase/schema.sql`](./supabase/schema.sql).
3. Click **Run**.

This creates the `profiles` and `app_state` tables, enables **Row Level
Security** with permissive authenticated policies, adds a trigger to auto-create
a profile on sign-up, and enables **Realtime** on `app_state`.

## 2. Enable email verification (for omnistockai.netlify.app)

> The app already sends `emailRedirectTo: https://omnistockai.netlify.app` on
> sign-up, but **Supabase will only honour that URL if it is in your Redirect
> URLs allowlist.** If the button still opens `localhost`, you missed step 2.

1. **Authentication → URL Configuration → Site URL** → set to
   `https://omnistockai.netlify.app`
2. **Authentication → URL Configuration → Redirect URLs** → click *Add URL* and
   add **both**:
   - `https://omnistockai.netlify.app`
   - `https://omnistockai.netlify.app/**`
   (This is the step that fixes the `localhost` redirect.)
3. **Authentication → Providers → Email → “Confirm email” → ON**
4. **Authentication → Email Templates → “Confirm signup”** → open
   [`supabase/email-confirm.html`](./supabase/email-confirm.html), **select all
   and copy**, then paste it into the template editor replacing everything, and
   click **Save**.

### Why the first email looked blank
The earlier template used CSS gradient-text (`color:transparent`), which most
mail clients render as **invisible**. The new template uses solid colors only,
so it renders everywhere. If you still see a blank/plain email, you haven’t
saved the new template in step 4 (or your mail client cached the old one).

### Flow
- Sign up → branded confirmation email is sent to the address.
- Click **“Confirm my email”** → redirected to `https://omnistockai.netlify.app`
  → the app auto-detects the session and logs the user in.

## 3. Credentials

Project credentials live in [`src/lib/supabase.ts`](./src/lib/supabase.ts):

- **URL:** `https://kuzqjnqubscwiskguewd.supabase.co`
- **Key:** the publishable key (with the classic anon JWT available as a
  fallback constant if the publishable format isn't accepted by your
  supabase-js version).

These are client-safe, publishable values — data is protected by RLS.

## 4. How it works

- **Sign up / Sign in** on the auth screen creates a real Supabase account.
  The role you pick during sign-up is stored in your profile and decides which
  dashboard you see.
- **Cloud sync:** the first time a signed-in user loads the app, the local seed
  is pushed to the shared `app_state` row. From then on every change (a sale, a
  stock edit, a new store/product, a purchase order) is debounced-pushed to
  Supabase and broadcast to all other clients via Realtime (last-write-wins).
- **Resilience:** `localStorage` remains the immediate source of truth. If
  Supabase is unreachable, the app keeps working locally and the header shows
  **"Local Mode"** instead of **"Cloud Synced"**.
- **Demo fallback:** the auth screen's "explore in local demo mode" buttons let
  you preview any role instantly without a Supabase account (offline, no sync).

## 5. Tables overview

| Table | Purpose |
|-------|---------|
| `profiles` | `id (→ auth.users)`, `name`, `role`, `email` |
| `app_state` | shared workspace document: `{ id, data (jsonb), updated_at, updated_by }` |

> Note: Supabase Auth users (real identities) are separate from the in-app
> **Team Directory** managed by the Platform Admin — the directory is synced
> business metadata shared across stores.
