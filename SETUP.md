# OmniStock — Supabase Setup Guide

OmniStock now uses **Supabase** for authentication and live cloud sync. Real
accounts are created via Supabase Auth, and the entire shared retail state
(stores, products, inventory, orders, logs, team directory) syncs to the cloud
in real time across every signed-in device and team member.

### Flow
- Sign up → branded confirmation email is sent to the address.
- Click **“Confirm my email”** → redirected to `https://omnistockai.netlify.app`
  → the app auto-detects the session and logs the user in.

## 2. How it works

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

## 3. Tables overview

| Table | Purpose |
|-------|---------|
| `profiles` | `id (→ auth.users)`, `name`, `role`, `email` |
| `app_state` | shared workspace document: `{ id, data (jsonb), updated_at, updated_by }` |

> Note: Supabase Auth users (real identities) are separate from the in-app
> **Team Directory** managed by the Platform Admin — the directory is synced
> business metadata shared across stores.
