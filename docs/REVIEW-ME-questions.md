# REVIEW ME — Quick Decision Sheet

**Date:** 2026-04-13
**Purpose:** Short, scrollable summary of the four headline findings + four questions that are blocking progress on the SecurityIndeed portal build.
**Full detail:** see `docs/portal-architecture-brief.md` (v3)

---

## The four headlines (what changed in the plan)

### 1. The portal you want already exists — sort of.
Path: `C:/Users/Daya/Projects/breach-guardian/`

It's a **Next.js + NextAuth + PostgreSQL + Prisma** web app that's 60–70% of what you described:
- Login / register / dashboard routes already built
- Training route with SCORM course viewer
- API routes for brokers, training, scans, webhooks, audit, users
- **Optery API key slot already in `.env.example`**
- Argon2id password hashing, account lockout, MFA fields in schema
- Production-ready Dockerfile
- **MID-REFACTOR** — has uncommitted changes in `app/(auth)/`, `app/(dashboard)/`, `components/`, `lib/stores/`
- Last commit: "Phase 5 testing + CI/CD"

**Recommendation:** finish this app, don't start from scratch.

---

### 2. Firebase Auth + Firestore was the wrong call. Scrap it.
The existing stack is:
- **Auth:** NextAuth.js v4 Credentials Provider (email + password + Argon2id)
- **DB:** PostgreSQL via Prisma
- NOT Firebase anywhere

Correct plan: **keep NextAuth + Postgres**, host Postgres on **Cloud SQL** (that IS GCP-native), deploy the existing Docker image to **Cloud Run**. Still fully GCP.

---

### 3. Mobile + web auth unification is easy.
Your mobile app (`C:/Users/Daya/Projects/securityindeed/breach-guardian-mobile/`):
- **Framework:** Expo React Native 0.76.9, Android-only (no iOS folder yet)
- **Auth:** Google OAuth 2.0 (PKCE) via `@react-native-google-signin/google-signin`
- **Web Client ID:** `121591951720-egglhh0qi98v04rgg9qr0jiukfrqpji4.apps.googleusercontent.com`
- **iOS Client ID:** `121591951720-i0pa7v35dnga89jrmpl36p4fa1smu7sv.apps.googleusercontent.com`
- Backend: none — fully on-device
- No billing

**Unification path:** Add a Google OAuth provider to the web app's NextAuth (~200 LOC + GCP console config). Mobile users who signed in with `user@gmail.com` then sign into the web with the same account. NextAuth links them by email. **No mobile-side migration needed.**

---

### 4. "Single Google-native cross-platform billing" does NOT exist.
Confirmed by research:
- Apple requires **StoreKit / IAP** for iOS subs (mandatory, 15–30% cut)
- **Google Play Billing is Android-only** — Google has never shipped a cross-platform equivalent
- Web-only: **Stripe**

**Realistic unified pattern: RevenueCat** wraps all three (StoreKit + Play Billing + Stripe) under one entitlement system.
- Pricing: free up to $2.5k MTR, then 1% of tracked revenue
- Alternative: Adapty (free under $5k MRR, then 1%)
- Your `/billing` page uses RevenueCat Web Billing (Stripe under the hood)
- Single `entitlements` table in Postgres = source of truth

---

## Scrubbing vendor landscape — only 3 are actually usable

| Vendor | Partner API? | Notes |
|---|---|---|
| **Optery** | ✅ YES | Best — public OpenAPI, webhooks, 640+ brokers, SOC 2 Type II — **primary** |
| **Onerep** | ✅ YES | REST + webhooks — reputational flag (CEO 2024 scandal) |
| **Privacy Bee** | ✅ YES | OAuth 2.0, enterprise slant |
| DeleteMe | ❌ | No consumer-removal API |
| Kanary | ❌ | No public API |
| Incogni | ❌ | No public API |
| Privacy Duck | ❌ | No API |

**Vendor-agnostic abstraction still correct.** Default implementation = Optery.

---

## The four questions I need you to answer

### Question 1 — Repo consolidation (most important)
You now have THREE repos on disk:
- `C:/Users/Daya/Projects/securityindeed-website/` → static HTML marketing landing page (where we are now)
- `C:/Users/Daya/Projects/breach-guardian/` → Next.js portal (mid-refactor)
- `C:/Users/Daya/Projects/securityindeed/breach-guardian-mobile/` → Expo mobile app

Pick one:
- **Option A (recommended):** Merge the `securityindeed-website` marketing HTML INTO the existing `breach-guardian/` Next.js app. Archive the static repo. One codebase, one deploy at `securityindeed.org`.
- **Option B:** Keep `securityindeed-website` as static marketing at `securityindeed.org`. Deploy `breach-guardian/` as portal at `app.securityindeed.org`. Two repos, two deploys, two SSL certs.
- **Option C:** Start a new consolidated repo, port good parts from both. Slowest path.

**Your answer:** _______________

---

### Question 2 — The mid-refactor in `breach-guardian/`
The existing Next.js portal has uncommitted changes. Pick one:
- **(a)** I should inspect those changes in detail and produce a "finish the mid-refactor" plan as Phase 1 of the build
- **(b)** Leave the mid-refactor for another session — just assume the existing committed state and plan forward from there
- **(c)** You'll check with whoever was working on it and tell me the state

**Your answer:** _______________

---

### Question 3 — Accounts you already have (check each)

| Service | Have it? |
|---|---|
| GCP paid project | ✅ (confirmed) |
| Google Cloud Console OAuth client | ✅ (mobile already uses one) |
| Domain `securityindeed.org` | ✅ (Search Console verified) |
| Stripe account | ☐ Yes ☐ No |
| RevenueCat account | ☐ Yes ☐ No |
| Apple Developer account (for iOS + IAP later) | ☐ Yes ☐ No |
| Google Play Console account | ☐ Yes ☐ No |
| Optery Business account (API key) | ☐ Yes ☐ No ☐ Unsure |
| Cloud SQL instance already running | ☐ Yes ☐ No |

**Your answers:** _______________

---

### Question 4 — Proceed to v4 formal build plan?
Do you want me to now write `docs/portal-build-plan-v4.md` with:
- Concrete phased tasks (Phase 0 → Phase 9)
- File-by-file scope for each phase
- PR sequencing plan
- Deployment runbook (GCP project setup → Cloud SQL → Secret Manager → Cloud Build → Cloud Run → domain mapping)
- Rollback and verification steps
- Still zero code — pure written plan

**Your answer:** ☐ Yes, write v4   ☐ No, something else first: _______________

---

## TL;DR — fill these in to unblock me

```
Q1: A / B / C
Q2: a / b / c
Q3: Stripe=?, RevenueCat=?, Apple Dev=?, Play Console=?, Optery=?, Cloud SQL=?
Q4: Yes / No (and what instead)
```

Reply with those four lines in chat (or edit this file and I'll read your answers) and I'll proceed.

---

*File saved for review. No code will be written until you answer.*
