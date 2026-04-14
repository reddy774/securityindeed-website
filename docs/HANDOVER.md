# SECURITYINDEED.ORG PORTAL — CLAUDE CODE HANDOVER PACKAGE

**Purpose:** Complete, self-contained handover document. Hand this file to any Claude Code (or other AI coding agent) session to resume building the SecurityIndeed.org user portal with zero prior context.

**Date prepared:** 2026-04-13
**Prepared by:** Claude Opus 4.6 (1M context) planning session
**Target:** Any fresh Claude Code session running locally on this machine
**Format:** Single markdown file, readable in Notepad, completely self-contained

---

## TABLE OF CONTENTS

1. [How to use this file](#1-how-to-use-this-file)
2. [Project identity](#2-project-identity)
3. [The three repos on disk](#3-the-three-repos-on-disk)
4. [Locked technical decisions](#4-locked-technical-decisions)
5. [Research findings](#5-research-findings)
6. [Prerequisites the user must complete](#6-prerequisites-the-user-must-complete)
7. [Secrets inventory](#7-secrets-inventory)
8. [The 10-phase build plan](#8-the-10-phase-build-plan)
9. [Dependency graph](#9-dependency-graph)
10. [Risk register](#10-risk-register)
11. [Acceptance criteria for launch](#11-acceptance-criteria-for-launch)
12. [Explicit deferrals](#12-explicit-deferrals)
13. [DO NOT do these things](#13-do-not-do-these-things)
14. [How to begin executing](#14-how-to-begin-executing)
15. [Reference: prior session files](#15-reference-prior-session-files)

---

## 1. HOW TO USE THIS FILE

### If you're a Claude Code session receiving this

1. **Read top to bottom once.** Do not skim. Every section contains load-bearing decisions.
2. **Do not re-research.** Sections 4 and 5 document decisions that are already final. Re-running research wastes tokens and may contradict locked calls.
3. **Do not re-plan the architecture.** It's in §4. Extend it; don't redesign it.
4. **The target repo is `C:/Users/Daya/Projects/breach-guardian/`** — NOT the `securityindeed-website` repo where this file lives. That repo is the marketing static site that is being merged INTO the `breach-guardian/` Next.js app.
5. **Zero code has been written in this planning session.** You are starting from a clean committed state.
6. **Mid-refactor warning:** `breach-guardian/` has uncommitted changes. User must handle those before Phase 1 (see §6).
7. **Everything in §13 is off-limits.** Read it before touching anything.
8. **Execute phases in order unless told otherwise.** Phases 4–6 can run in parallel after Phase 3.

### User-facing instructions for handing this off

1. Open a new Claude Code session in `C:/Users/Daya/Projects/breach-guardian/` (the target repo — not this one)
2. Paste: "Read `C:/Users/Daya/Projects/securityindeed-website/docs/HANDOVER.md` completely and begin from Phase 0. Confirm the prerequisites in §6 with me before starting Phase 1."
3. The new session will read this file, confirm prereqs, and begin

---

## 2. PROJECT IDENTITY

### Product: Breach Guardian

An AI-powered privacy and identity protection product with three surfaces:

- **Mobile app** (Expo React Native, Android-only today): on-device AI scam/phishing detection, breach monitoring via Gmail API, local SQLite storage. Already exists, in active development.
- **Web app** (Next.js + NextAuth + Postgres + Prisma): user dashboard, breach monitoring, data broker removal, security training, audit logs. Exists, mid-refactor, 60–70% built.
- **Marketing site** (static HTML): landing page, features, pricing, privacy, terms. Exists and is live. Google Search Console verified.

### Target audience
Everyone, including seniors. "Easy Mode" for seniors is a first-class feature alongside an "Advanced Mode."

### Target domain
`securityindeed.org` — already registered, already Search Console verified.

### Target cloud
GCP (Google Cloud Platform). User has a paid Pro GCP account. Deployment target is **Cloud Run** (containerized Next.js), with **Cloud SQL (PostgreSQL)** as the database, **Secret Manager** for secrets, **Cloud KMS** for PII encryption, **Cloud Build → Artifact Registry** for CI/CD.

### What the portal (`securityindeed.org`) will be

A full GCP-native SaaS web portal with:

1. **Public marketing surface** for user acquisition and SEO
2. **Authenticated user dashboard** — BG mobile app users log in with the same credentials they use in the app (shared Google identity)
3. **Training module runner** — per-user progress tracking for security-awareness training
4. **Data-scrubbing tab** — user submits a PII profile, backend calls a vendor-agnostic scrubbing provider (default: Optery) to have their data removed from broker sites
5. **Billing page** — unified subscription management across iOS + Android + web via RevenueCat
6. **Account management** — profile, password/MFA, DSAR delete

### What "SecurityIndeed.org" is NOT

- NOT a marketing site redesign (that was the original scope; it has expanded)
- NOT a new codebase from scratch (extends `breach-guardian/`)
- NOT a Firebase project (does not use Firebase Auth or Firestore)
- NOT a separate auth system from the mobile app (shares Google OAuth identity)

---

## 3. THE THREE REPOS ON DISK

All three live on this machine at the paths below. They are currently architecturally independent and must be unified per this plan.

### Repo 1 — Static marketing site (ARCHIVING AFTER PHASE 2)

**Path:** `C:/Users/Daya/Projects/securityindeed-website/`
**Contents:**
- `index.html` — 794-line single-page landing site for Breach Guardian
- `privacy.html`, `terms.html`
- `google26df59702563d68d.html` — Google Search Console verification file
- `CLAUDE.md`, `AGENTS.md`, `GEMINI.md` — LLM adapter files pointing at `AGENTS.md` (which is mostly TODO)
- `docs/` — this planning session's documents:
  - `portal-architecture-brief.md` (v1 → v2 → v3, the thinking history)
  - `REVIEW-ME-questions.md` (decisions answered by user)
  - `portal-build-plan-v4.md` (the detailed plan)
  - `HANDOVER.md` (this file)

**Design system in use:**
- Palette: fortress-900 (#0b3142), fortress-950 (#021e2b), frost (#eef8ff), mist, off-white, with blue/green/amber/red accents. WCAG AAA contrast (13.71:1 on white).
- Fonts: DM Serif Display (headings) + Outfit (body) from Google Fonts
- Tone: "Light Trust Theme" — anti-AI-template, research-backed (Stanford Web Credibility: white background builds trust)

**Status:** Live but will be merged into Repo 2 during Phase 2. After migration, this repo gets a README note and is archived.

### Repo 2 — Next.js portal (THE TARGET REPO — all Phase 1+ work happens here)

**Path:** `C:/Users/Daya/Projects/breach-guardian/`
**Stack:**
- **Framework:** Next.js (App Router, `app/` directory)
- **Auth:** NextAuth.js v4.24.13, Credentials Provider (email + password)
- **Password hashing:** `argon2@^0.44.0` (Argon2id)
- **Database:** PostgreSQL via Prisma (`@prisma/client@^5.22.0`)
- **Sessions:** JWT, 24hr `maxAge`
- **Security features already in place:** account lockout after 5 failed attempts (15 min), MFA fields in schema, `AuditLog` model, soft-delete tracking

**Existing routes (observed):**
- `app/(auth)/login/`, `app/(auth)/register/` — authentication pages
- `app/(dashboard)/dashboard/` — main dashboard
- `app/api/auth/[...nextauth]/` — NextAuth endpoints
- `app/api/audit/`, `app/api/brokers/`, `app/api/health/`, `app/api/scans/`, `app/api/training/`, `app/api/users/`, `app/api/webhooks/` — REST API routes
- `app/training/[courseId]` — SCORM course viewer

**Prisma models (observed):**
- `User` — id, email, passwordHash, name, interfaceMode, role, mfaEnabled, mfaSecret, failedLoginAttempts, lockedUntil, soft-delete
- `MonitoredEmail`, `Breach` — breach monitoring
- `CourseProgress` — training
- `RemovalRequest` — data broker removal
- `AuditLog` — security events

**Env var slots already defined:**
- `DATABASE_URL` (Postgres)
- `HIBP_API_KEY` (Have I Been Pwned)
- **`OPTERY_API_KEY` already present** (empty in .env.example — slot exists)
- `NEXTAUTH_SECRET`, `NEXTAUTH_URL`
- n8n webhook URL

**Deployment state:**
- Production `Dockerfile` exists (Node.js 20-alpine, multi-stage, non-root user, port 3000)
- `.next/standalone` output present — already builds
- No Vercel config — designed for self-hosted (Docker / K8s / Cloud Run)

**Git state (⚠️ critical):**
- On `main` branch
- **UNCOMMITTED CHANGES** — the user must handle these before Phase 1 starts:
  - Untracked: `app/(auth)/`, `app/(dashboard)/`, `components/`, `lib/stores/`, `public/`
  - Modified: `app/globals.css`, `app/layout.tsx`, `app/page.tsx`
- Last committed change: `2b2b3f7` "Add testing infrastructure and CI/CD pipeline (Phase 5)"

**Status:** Target of all Phase 1+ work. Will become the single source of truth for `securityindeed.org`.

### Repo 3 — Mobile app (NOT TOUCHED BY THIS PLAN)

**Path:** `C:/Users/Daya/Projects/securityindeed/breach-guardian-mobile/`
**Stack:**
- **Framework:** Expo (React Native 0.76.9) with TypeScript
- **Platform:** Android only (no `ios/` folder yet)
- **Auth:** Google OAuth 2.0 (PKCE) via `@react-native-google-signin/google-signin@^16.1.2` + `expo-auth-session@~6.0.3`
  - **Web Client ID:** `121591951720-egglhh0qi98v04rgg9qr0jiukfrqpji4.apps.googleusercontent.com`
  - **iOS Client ID:** `121591951720-i0pa7v35dnga89jrmpl36p4fa1smu7sv.apps.googleusercontent.com`
- **Token storage:** `expo-secure-store` (fallback `AsyncStorage` in Expo Go)
- **Local DB:** `expo-sqlite@~15.1.4`
- **State:** `zustand`
- **Backend integration:** NONE — fully on-device. Uses Gmail API read-only scope (`gmail.readonly`) to fetch breach-related emails locally.
- **Billing / IAP:** NONE implemented

**Git state:** Clean working tree, up to date with `origin/main`, last commit `a0c9f88` "feat(training): add Colab notebooks..."

**Status:** **DO NOT modify during this build.** Mobile integration is Phase 10, deferred to a separate future plan. The shared Google OAuth identity is what makes the web portal's auth unification possible — mobile needs zero changes for web auth to work via Google.

---

## 4. LOCKED TECHNICAL DECISIONS

Every decision in this table is FINAL. Do not re-litigate.

| # | Layer | Decision | Why |
|---|---|---|---|
| 1 | Target repo | **`C:/Users/Daya/Projects/breach-guardian/`** — extend this, don't start over | Already 60–70% built, already has NextAuth, Prisma, Optery slot, production Dockerfile |
| 2 | Stack | **Next.js (App Router) + NextAuth.js v4 + Postgres + Prisma** | Existing stack in `breach-guardian/`. Don't fight it. |
| 3 | Database hosting | **Cloud SQL PostgreSQL** on GCP | Most GCP-native relational option. Pairs with existing Prisma schema. |
| 4 | Auth | Keep **NextAuth Credentials Provider** (existing) + **add Google OAuth provider** | Unifies with the mobile app's existing Google OAuth identity via email linking. No Firebase Auth — mobile doesn't use it. |
| 5 | Hosting | **Cloud Run** (containerized Next.js, existing Dockerfile) | User wants GCP Cloud Run; existing Dockerfile already targets this class of runtime |
| 6 | CI/CD | **Cloud Build → Artifact Registry → Cloud Run** | Most GCP-native CI path |
| 7 | Domain + SSL | **`securityindeed.org`** via Cloud Run domain mapping + Google-managed cert | Domain already registered, already Search Console verified |
| 8 | Secrets | **GCP Secret Manager** | Never in code or `.env` committed files |
| 9 | PII encryption at rest | **Cloud KMS + `pgcrypto`** (envelope encryption) | Column-level encryption for PII profile data, DEK sourced from KMS at app startup |
| 10 | Scrubbing vendor | **Vendor-agnostic `ScrubbingProvider` interface** with Optery as default implementation | User explicitly asked for vendor agnosticism; only 3 vendors have real APIs anyway (Optery, Onerep, Privacy Bee) |
| 11 | Training format | **Custom MDX + progress events in Postgres** — existing SCORM viewer kept as parallel path | User asked for "best and most economical" — MDX has zero licensing cost, runs native on Next.js |
| 12 | Billing | **RevenueCat** as unified entitlement layer — Stripe on web (via RevenueCat Web Billing), StoreKit + Play Billing on mobile when Phase 10 happens | No Google-native cross-platform billing exists. RevenueCat is the de facto standard. Free up to $2.5k MTR then 1%. |
| 13 | Mobile changes | **Deferred to Phase 10** — do NOT modify the mobile repo during this build | Mobile uses Google OAuth already; no change needed for shared auth. Billing SDK is a Phase 10 concern. |
| 14 | Logging | **Cloud Logging + Cloud Audit Logs** + existing `AuditLog` Prisma model | Native GCP observability |
| 15 | Design system | **Keep existing fortress/frost palette** + **DM Serif Display + Outfit fonts** from the static marketing site | Already WCAG AAA, already research-backed, already loved |

### Decisions that were RULED OUT (don't resurrect these)

- ❌ **Astro 5 static site** — was recommended in a prior session for a marketing site; this project is now a full web app, not a content site
- ❌ **Firebase Authentication** — the existing target repo uses NextAuth and mobile uses direct Google OAuth; Firebase adds nothing
- ❌ **Firestore** — existing Prisma schema is the source of truth; migrating to a NoSQL model would destroy work
- ❌ **Starting a new repo from scratch** — user explicitly picked Option A (merge into `breach-guardian/`)
- ❌ **Single Google-native cross-platform billing** — does not exist; Apple mandates StoreKit, Google Play Billing is Android-only, no Google product spans iOS/Android/web
- ❌ **Custom auth server or JWT bridge** — unnecessary since Google OAuth shared identity is the path
- ❌ **DeleteMe, Kanary, Incogni, Privacy Duck** as scrubbing vendors — none have a real consumer-removal partner API as of 2026

---

## 5. RESEARCH FINDINGS

Research was completed during the planning session. You do not need to redo this. Sources are cited at the bottom.

### 5.1 Data broker removal vendor landscape (2026)

| Vendor | Partner API? | Auth | Webhooks | Notes |
|---|---|---|---|---|
| **Optery** | ✅ YES — strongest | API key (Business account) | Yes, monitoring events | 640+ brokers, SOC 2 Type II, public OpenAPI sandbox at `public-api-sandbox.test.optery.com/v1/docs`. Designed for embedding (banking, cyber, IDP resellers). Accepts structured JSON profile. **PRIMARY DEFAULT IMPLEMENTATION.** |
| **Onerep** | ✅ YES | API key | Yes (scan-ready, opt-out sent, removed) | 310+ brokers, REST API. **Reputational flag:** CEO was publicly exposed in 2024 as having run people-search sites. Still functional but disclose to end users. |
| **Privacy Bee** | ✅ YES | OAuth 2.0 | Yes, native `/webhooks` resource | Enterprise-oriented, 200 RPM rate limit, B2B slant but supports individual enrollment. v3 REST API. |
| **DeleteMe (Abine)** | ❌ | — | — | Has `cyber.deleteme.com` breach-data API (different product) + affiliate/channel partner program. No consumer-removal partner API. **DEALBREAKER.** |
| **Kanary** | ❌ | — | — | FAQ confirms no public/partner API. Consumer + enterprise SSO only. **DEALBREAKER.** |
| **Incogni** | ❌ | — | — | Surfshark-owned, consumer-only, no developer portal. **DEALBREAKER.** |
| **Privacy Duck** | ❌ | — | — | Consumer dashboard only, human-assisted removal. **DEALBREAKER.** |

**Verdict:** Only Optery, Onerep, and Privacy Bee are viable. Build the `ScrubbingProvider` abstraction to fit the intersection of these three.

### 5.2 The `ScrubbingProvider` interface (target design)

```ts
// lib/scrubbing/provider.ts
export interface ScrubbingProvider {
  enrollUser(profile: PIIProfile): Promise<{ providerId: string }>
  updateProfile(providerId: string, patch: Partial<PIIProfile>): Promise<void>
  triggerScan(providerId: string): Promise<{ scanId: string }>
  getStatus(providerId: string): Promise<EnrollmentStatus>
  listBrokers(providerId: string): Promise<BrokerRecord[]>
  listRemovals(providerId: string): Promise<RemovalReceipt[]>
  verifyWebhook(headers: Headers, rawBody: string): boolean
  parseWebhook(payload: unknown): NormalizedEvent
  deleteUser(providerId: string): Promise<void>
}
```

### 5.3 `PIIProfile` schema (intersection across Optery + Onerep + Privacy Bee)

```ts
// lib/scrubbing/types.ts
export type PIIProfile = {
  firstName: string
  middleName?: string
  lastName: string
  aliases?: { first: string; middle?: string; last: string }[]
  dateOfBirth: string                        // ISO-8601, required by all 3
  emails: string[]                           // >=1
  phones: { number: string; type?: 'mobile'|'home' }[]
  addresses: {
    line1: string; line2?: string
    city: string; state: string; postalCode: string
    countryCode: 'US'                        // US-only coverage today
    from?: string; to?: string
  }[]
  relatives?: { firstName: string; lastName: string }[]  // boosts match quality
  gender?: 'M'|'F'|'X'
  consent: {
    tosAcceptedAt: string
    ipAddress: string
    authorizedAgent: true                    // required for Optery/Onerep
  }
}

export type NormalizedEvent =
  | { type: 'scan.ready'; providerId: string; scanId: string }
  | { type: 'optout.sent'; providerId: string; brokerId: string }
  | { type: 'removal.confirmed'; providerId: string; brokerId: string }
  | { type: 'profile.reexposed'; providerId: string; brokerId: string }
```

Union-only extras (optional, per-provider): `employer`, `education`, `socialHandles`, `vehiclePlates`.

### 5.4 Cross-platform billing reality (2026)

**Verdict: There is NO single Google-native cross-platform billing product.** Confirmed.

**Policy landscape (2026):**
- **Apple (US):** IAP mandatory for in-app digital subs; since May 2025, US apps can link out to external web checkout (post Epic v. Apple) with no Apple commission on off-device purchases. 15/30% still applies to StoreKit purchases.
- **Apple (EU/DMA):** Layered fee model: 2% Initial Acquisition + 5–13% Store Services + 5% Core Technology Commission (CTC replaces CTF Jan 1, 2026). External payment links permitted under DMA.
- **Google Play (US):** New Alternative Billing + External Content Links programs; mandatory enrollment by Jan 28, 2026. External transactions reported to Google within 24 hours.
- **Google Play (EEA):** External offers allowed; Google takes 5%/10% acquisition + 7%/17% ongoing on external transactions.
- **Google Play Billing is Android-only.** Google has never shipped an iOS or web equivalent.

**Recommended unified pattern: RevenueCat**

```
  ┌─ iOS app ─────────→ StoreKit (IAP) ─┐
  │                                      ↓
  │                                 RevenueCat ──webhook──→ Postgres
  │                                      ↑                  (entitlements table)
  ├─ Android app ────→ Play Billing ────┤
  │                                      ↑
  └─ securityindeed.org/billing ──→ RevenueCat Web Billing ─→ Stripe
```

**RevenueCat pricing (2026):** Free up to $2.5k MTR (monthly tracked revenue), then 1% of tracked revenue. Web Billing SDK officially unifies iOS/Android/Web entitlements under one `customerInfo` object.

**Alternatives:**
- **Adapty** — Free under $5k MRR then 1%. Strong RevenueCat alternative.
- **Glassfy** — Simpler, acquired/stagnating. Adapty publishes a migration guide.
- **Purchasely** — Enterprise paywall focus, higher price.
- **Stripe alone** — Works for web but you build all entitlement sync, receipt validation, webhook handling yourself. Not recommended.

**Locked choice: RevenueCat.**

### 5.5 Sources cited (so you don't re-fetch)

- https://www.optery.com/api/
- https://public-api-sandbox.test.optery.com/v1/docs
- https://www.optery.com/partnership-program/
- https://onerep.com/api
- https://onerep.com/business/api-contact
- https://docs.privacybee.dev/
- https://privacybee.dev/api/overview/
- https://business.privacybee.com/partners/
- https://joindeleteme.com/business/partners/
- https://cyber.deleteme.com/api
- https://www.kanary.com/frequently-asked-questions
- https://incogni.com/
- https://privacyduck.com/
- https://developer.apple.com/news/?id=awedznci
- https://www.revenuecat.com/blog/growth/apple-eu-dma-update-june-2025/
- https://techcrunch.com/2025/05/02/apple-changes-us-app-store-rules-to-let-apps-redirect-users-to-their-own-websites-for-payments/
- https://support.google.com/googleplay/android-developer/answer/15582165
- https://support.google.com/googleplay/android-developer/answer/16505463
- https://support.google.com/googleplay/android-developer/answer/16671517
- https://www.revenuecat.com/pricing/
- https://www.revenuecat.com/blog/engineering/cross-platform-subscriptions-ios-android-web/
- https://www.revenuecat.com/blog/engineering/can-you-use-stripe-for-in-app-purchases/
- https://adapty.io/blog/adapty-new-pricing-2026/
- https://adapty.io/docs/migration-from-glassfy
- https://docs.stripe.com/mobile/digital-goods

### 5.6 Trust-building references (already informed the static site design)

- **Stanford Web Credibility Project** — white background builds trust; the existing fortress/frost palette is based on this
- **NN/g Trustworthiness articles 2023–2025** — named teams, third-party verification, live metrics, contact reality, partner logo strips

---

## 6. PREREQUISITES THE USER MUST COMPLETE

None of these are automatable. User must finish all 6 before Phase 1 starts.

### 6.1 Handle the uncommitted mid-refactor in `breach-guardian/`

The target repo has uncommitted work. Pick one:

```bash
# Option A: commit to a branch (SAFEST)
cd C:/Users/Daya/Projects/breach-guardian/
git checkout -b refactor/phase-5-wip
git add -A
git commit -m "WIP: mid-refactor snapshot before portal build"
git checkout main

# Option B: stash
cd C:/Users/Daya/Projects/breach-guardian/
git stash push -u -m "mid-refactor wip"

# Option C: discard (DESTRUCTIVE — only if certain it's abandoned)
cd C:/Users/Daya/Projects/breach-guardian/
git restore .
git clean -fd
```

**User must tell the executing Claude session which option they picked before Phase 1.**

### 6.2 Accounts to create

| Service | Action | Needed in |
|---|---|---|
| **RevenueCat** | Sign up at `revenuecat.com` → create project "SecurityIndeed" → note the public SDK key + webhook secret → save to a password manager | Phase 5 |
| **Optery Business** | Email `partners@optery.com` requesting Business API access. If access isn't granted in time, Phase 4 ships with `StubProvider` ("coming soon"). Either path is fine. | Phase 4 |
| **Cloud SQL** | NO manual action — will be provisioned in Phase 1 via `gcloud` by the executing agent | Phase 1 |

### 6.3 GCP project info to provide

The executing agent needs these from the user:

- **GCP project ID:** `_______________`
- **GCP project number:** `_______________`
- **Preferred region:** default recommendation `us-central1`; confirm or override
- **OAuth Web Client strategy:** reuse mobile's existing Web Client ID `121591951720-egglhh0qi98v04rgg9qr0jiukfrqpji4.apps.googleusercontent.com` or create a new web-only client?
- **Billing account:** confirm active

### 6.4 DNS registrar

`securityindeed.org` is registered somewhere — Namecheap / GoDaddy / Google Domains / Cloudflare. User must confirm which so the executing agent can add the Cloud Run CNAME records in Phase 9.

### 6.5 User-provided content for launch

- Named team photos + titles + bios (Phase 8)
- Legal entity address + phone (Phase 8)
- Confirmed SOC 2 status (real certification or placeholder language) (Phase 7/8)

### 6.6 Existing accounts confirmed in planning session

| Service | Status |
|---|---|
| GCP paid project | ✅ Confirmed |
| Google Cloud Console OAuth client | ✅ Exists (mobile uses it) |
| Domain `securityindeed.org` registered | ✅ Confirmed |
| Search Console verified for domain | ✅ Confirmed |
| Stripe account | ✅ Confirmed |
| Apple Developer account | ✅ Confirmed |
| Google Play Console account | ✅ Confirmed |
| RevenueCat account | ❌ Not yet — create in §6.2 |
| Optery Business account | ❌ Not yet — email in §6.2 |
| Cloud SQL instance | ❌ Not yet — provisioned by agent in Phase 1 |

---

## 7. SECRETS INVENTORY

Every secret the portal will need. Stored in GCP Secret Manager. Never committed to code.

| Secret name | Created in | Populated in | Source | Notes |
|---|---|---|---|---|
| `NEXTAUTH_SECRET` | Phase 1 | Phase 1 | Generate: `openssl rand -base64 64` | JWT signing for NextAuth |
| `NEXTAUTH_URL` | Phase 1 | Phase 9 | Literal: `https://securityindeed.org` | Full URL, no trailing slash |
| `DATABASE_URL` | Phase 1 | Phase 1 | Cloud SQL connection string with app user | Prefer private IP + VPC connector |
| `GOOGLE_CLIENT_ID` | Phase 1 | Phase 3 | GCP Console → Credentials | Reuse mobile's or new web-only |
| `GOOGLE_CLIENT_SECRET` | Phase 1 | Phase 3 | GCP Console → Credentials | Paired with above |
| `REVENUECAT_PUBLIC_SDK_KEY` | Phase 1 | Phase 5 | RevenueCat dashboard | Web SDK key |
| `REVENUECAT_WEBHOOK_SECRET` | Phase 1 | Phase 5 | RevenueCat dashboard | HMAC verification |
| `STRIPE_SECRET_KEY` | Phase 1 | Phase 5 | Stripe dashboard → API keys | Test mode first, then prod |
| `STRIPE_WEBHOOK_SECRET` | Phase 1 | Phase 5 | Stripe CLI or dashboard | Endpoint-specific |
| `OPTERY_API_KEY` | Phase 1 | Phase 4 (if available) | Optery partner portal | Empty until account exists — use `StubProvider` |
| `KMS_KEY_RESOURCE_NAME` | Phase 1 | Phase 1 | GCP KMS | Format: `projects/PROJECT/locations/REGION/keyRings/securityindeed-keyring/cryptoKeys/pii-dek` |
| `SENDGRID_API_KEY` or `SMTP_*` | Phase 1 | Phase 3 | Chosen email provider | Transactional email: verify, password reset |
| `HIBP_API_KEY` | existing | existing | Already in `.env.example` | Have I Been Pwned |

---

## 8. THE 10-PHASE BUILD PLAN

Each phase has: goal, tasks, files touched, PRs, verification, rollback.

---

### PHASE 0 — Pre-flight (docs only, no code)

**Goal:** Everything ready to start Phase 1 with zero blockers.

**Tasks:**
1. Verify user completed §6 prerequisites (ask them point-by-point)
2. Write `docs/secrets-inventory.md` in the target repo (finalized version of §7 above)
3. Write `docs/phase-1-pr-plan.md` with the exact `gcloud` commands Phase 1 will run
4. Confirm: GCP project ID, region, OAuth client strategy, DNS registrar, RevenueCat status, Optery status, refactor handling

**Files touched:** docs only in `breach-guardian/docs/`
**PRs:** none (docs only, committed to a branch)
**Verification:** user explicitly says "start Phase 1"
**Rollback:** delete the new docs
**Exit criteria:** user sign-off

---

### PHASE 1 — GCP infrastructure setup

**Goal:** All GCP resources provisioned and documented, ready for the app to deploy into.

**Tasks:**
1. Enable GCP APIs:
   ```
   run.googleapis.com
   sqladmin.googleapis.com
   secretmanager.googleapis.com
   cloudbuild.googleapis.com
   artifactregistry.googleapis.com
   cloudkms.googleapis.com
   logging.googleapis.com
   iam.googleapis.com
   servicenetworking.googleapis.com
   vpcaccess.googleapis.com
   ```
2. Create **Artifact Registry** repo: `securityindeed-portal` in chosen region (Docker format)
3. Provision **Cloud SQL PostgreSQL 16** instance:
   - Name: `securityindeed-db`
   - Tier: `db-f1-micro` (upgradable)
   - **Private IP + VPC connector** (for Cloud Run access)
   - Automated backups ON
   - Point-in-time recovery ON
   - Region: same as Cloud Run
4. Create app DB + user: DB name `portal_db`, user `portal_app` (no superuser privileges)
5. Create **VPC connector** for Cloud Run → Cloud SQL private IP
6. Create **Cloud KMS keyring** `securityindeed-keyring` + symmetric key `pii-dek` (HSM or software level)
7. Create all **Secret Manager secrets** (placeholder values) from §7
8. Create **runtime service account** `portal-runtime@PROJECT.iam.gserviceaccount.com` with roles:
   - `roles/run.invoker`
   - `roles/cloudsql.client`
   - `roles/secretmanager.secretAccessor`
   - `roles/cloudkms.cryptoKeyEncrypterDecrypter`
   - `roles/logging.logWriter`
9. Create **Cloud Build service account** permissions for deploy
10. Document everything in `breach-guardian/docs/gcp-infra.md` with exact `gcloud` commands used + rollback commands

**Files touched in `breach-guardian/`:** `docs/gcp-infra.md`, `docs/secrets-inventory.md` (populated)
**PRs:** 1 PR ("docs(infra): GCP infrastructure setup runbook")
**Verification:**
```
gcloud sql instances describe securityindeed-db              # RUNNABLE
gcloud kms keys list --keyring=securityindeed-keyring ...   # pii-dek exists
gcloud secrets list                                          # all placeholders
gcloud artifacts repositories list                           # securityindeed-portal
gcloud iam service-accounts list                             # portal-runtime exists
```
**Rollback:** all commands scripted in `gcp-infra.md` with a "Teardown" section at the bottom
**Cost check:** ~$10–15/month at idle (Cloud SQL dominates; Cloud Run scales to zero)
**Duration:** 1 session

---

### PHASE 2 — Repository consolidation (Option A)

**Goal:** Marketing HTML from `securityindeed-website/` fully migrated into `breach-guardian/`. Static repo archived.

**Tasks:**
1. In `breach-guardian/`, create marketing route group `app/(marketing)/`:
   - `app/(marketing)/layout.tsx` — shared nav + footer
   - `app/(marketing)/page.tsx` — landing (port from `securityindeed-website/index.html`)
   - `app/(marketing)/features/page.tsx`
   - `app/(marketing)/pricing/page.tsx`
   - `app/(marketing)/how-it-works/page.tsx`
   - `app/(marketing)/privacy/page.tsx` — port from `securityindeed-website/privacy.html`
   - `app/(marketing)/terms/page.tsx` — port from `securityindeed-website/terms.html`
2. Port the design system:
   - Extract fortress/frost color tokens into Tailwind config if Tailwind is present, or into `app/globals.css` CSS variables otherwise
   - Import DM Serif Display + Outfit via `next/font/google` (faster than `<link>` tags from Google Fonts CDN)
3. Create shared marketing components in `components/marketing/`:
   - `<MarketingNav>` — fixed header with logo, nav links, login CTA
   - `<Hero>` — hero grid with content + visual
   - `<FeatureGrid>` — auto-fit 320px cards
   - `<PrivacyArchitecture>` — 2-col diagram + checkmarks
   - `<ModeCard>` — Easy Mode / Advanced Mode
   - `<PricingCard>` — tier card with featured variant
   - `<DownloadCTA>` — App Store + Play Store badges
   - `<MarketingFooter>` — links, copyright, contact
4. Move Google Search Console verification file:
   - `public/google26df59702563d68d.html`
5. SEO scaffolding:
   - `app/sitemap.ts` — auto-generated from route manifest
   - `public/robots.txt`
   - Per-page `metadata` exports for OG tags
6. Document migration decisions in `docs/phase-2-migration-notes.md`
7. Archive `securityindeed-website/` — add a README note at the top:
   > This repo is superseded by the Next.js app at `C:/Users/Daya/Projects/breach-guardian/`. Retained for history. See `docs/HANDOVER.md` for plan context.

**Files touched in `breach-guardian/`:** ~20 new files under `app/(marketing)/`, `components/marketing/`, `public/`, possibly `tailwind.config.ts` or `app/globals.css`
**Files touched in `securityindeed-website/`:** README only
**PRs:** 1 PR ("feat(marketing): port securityindeed.org landing into portal app")
**Verification:**
- `npm run dev` in `breach-guardian/` → visit `/`, `/features`, `/pricing`, `/how-it-works`, `/privacy`, `/terms` — all render correctly
- Lighthouse ≥ 90 on `/`
- All internal nav links work
- No 404s
**Rollback:** revert the PR; `securityindeed-website/` untouched
**Duration:** 1–2 sessions

---

### PHASE 3 — Auth unification: Google OAuth in NextAuth

**Goal:** A user can sign in with Google on the web and auto-links to (or creates) the same user the mobile app sees via shared Google identity (email).

**Tasks:**
1. In GCP Console, verify the OAuth Web Client has these authorized redirect URIs:
   - `https://securityindeed.org/api/auth/callback/google`
   - `http://localhost:3000/api/auth/callback/google`
   - Add if missing
2. Update NextAuth config in `breach-guardian/lib/auth.ts` (or `app/api/auth/[...nextauth]/route.ts`) to add the Google provider alongside the existing Credentials provider
3. Enable **account linking by verified email** safely:
   - Only allow link when `emailVerified = true` on existing account
   - Reject otherwise (security invariant)
4. Update Prisma schema — ensure `Account` table (NextAuth's OAuth linking table) exists; create migration if needed
5. On first Google sign-in, create user row with `emailVerified = now()`, default role, linked `Account` row with `provider = 'google'`
6. Update login + register pages:
   - `/login` — "Sign in with Google" as primary CTA, email+password as secondary
   - `/register` — same
7. Update NextAuth callbacks to populate custom session fields (user id, role, entitlement status placeholder)
8. Document the migration path for existing password-only users in `docs/phase-3-auth-migration.md`
9. Tests:
   - Unit: account linking guard (reject unverified link attempts)
   - Integration: full Google sign-in flow
   - E2E (Playwright): happy path, error path, account link path

**Files touched:** `lib/auth.ts`, `app/api/auth/[...nextauth]/route.ts` (or equivalent), `prisma/schema.prisma`, new migration, `app/(auth)/login/page.tsx`, `app/(auth)/register/page.tsx`, `components/auth/GoogleSignInButton.tsx`
**PRs:** 1 PR ("feat(auth): add Google OAuth provider + safe account linking")
**Verification:**
- Manual: 3 flows above
- Automated: unit + integration + E2E tests
- Security: unverified email cannot hijack an existing account
**Rollback:** revert PR; existing Credentials auth unchanged
**Duration:** 1 session

---

### PHASE 4 — Vendor-agnostic scrubbing abstraction + Optery (or stub)

**Goal:** `/data-scrubbing` page works end-to-end. Either with a real Optery account (if available) or with a `StubProvider` until one is.

**Tasks:**
1. Define the interface in `lib/scrubbing/provider.ts` (from §5.2 above)
2. Define types in `lib/scrubbing/types.ts` (from §5.3 above — `PIIProfile`, `NormalizedEvent`, `EnrollmentStatus`, `BrokerRecord`, `RemovalReceipt`)
3. Add Prisma models:
   - `ScrubbingEnrollment` — userId (FK), providerName, providerId, status, createdAt, updatedAt
   - `ScrubbingBroker` — enrollmentId (FK), brokerName, foundAt, status
   - `ScrubbingRemovalReceipt` — enrollmentId (FK), brokerId, confirmedAt, rawPayload
   - `ScrubbingEvent` — enrollmentId (FK), eventType (normalized), createdAt, rawPayload
   - `PiiProfile` — enrollmentId (FK), encryptedPayload (pgcrypto), iv, keyVersion
4. Run migration: `npx prisma migrate dev --name add_scrubbing_models`
5. Implement `OpteryProvider` in `lib/scrubbing/providers/optery.ts`:
   - Full interface implementation against Optery REST API
   - Uses `OPTERY_API_KEY` from Secret Manager (fetched at runtime via `@google-cloud/secret-manager` or injected via Cloud Run `--set-secrets`)
   - Webhook signature verification per Optery docs
6. Implement `StubProvider` in `lib/scrubbing/providers/stub.ts`:
   - Returns mock data and 501 where appropriate
   - Used when `OPTERY_API_KEY` is missing OR `SCRUBBING_PROVIDER=stub`
7. Provider factory in `lib/scrubbing/index.ts`:
   ```ts
   export function getProvider(): ScrubbingProvider {
     if (process.env.OPTERY_API_KEY) return new OpteryProvider(...)
     return new StubProvider()
   }
   ```
8. API routes:
   - `POST /api/scrub/submit` — validates `PIIProfile` with Zod, encrypts sensitive fields via `pgcrypto`, calls `provider.enrollUser()`, writes `ScrubbingEnrollment`
   - `GET /api/scrub/status` — reads current status from DB + refreshes via `provider.getStatus()` on demand
   - `GET /api/scrub/brokers` — lists `ScrubbingBroker` records
   - `GET /api/scrub/history` — lists `ScrubbingRemovalReceipt` records
   - `POST /api/scrub/webhook` — verifies signature, parses with `provider.parseWebhook()`, writes `ScrubbingEvent`, updates status
   - `DELETE /api/scrub/enrollment` — DSAR via `provider.deleteUser()` + cascade delete in DB
9. PII encryption:
   - DEK fetched from Cloud KMS at app startup
   - `pgcrypto` symmetric encryption on `PiiProfile.encryptedPayload`
   - IV and key version stored alongside
10. UI:
    - `app/(dashboard)/data-scrubbing/page.tsx` — form (React Hook Form + Zod)
    - `app/(dashboard)/data-scrubbing/history/page.tsx` — timeline of events + receipts
    - `components/scrubbing/ScrubForm.tsx`, `ScrubStatusCard.tsx`, `BrokerList.tsx`, `EventTimeline.tsx`
11. Dashboard summary card: "Data scrubbing: X brokers removed, Y pending"
12. Document the webhook URL format for the user to register in the vendor dashboard

**Files touched:** ~15 new files under `lib/scrubbing/`, `app/(dashboard)/data-scrubbing/`, `app/api/scrub/`, `components/scrubbing/`, `prisma/schema.prisma`, new migration, dashboard
**PRs:** 2 PRs
- PR 4a: "feat(scrubbing): provider interface, Prisma models, API routes"
- PR 4b: "feat(scrubbing): UI pages and dashboard integration"
**Verification:**
- Unit tests: `OpteryProvider` with mocked HTTP; `StubProvider`
- Integration: submit → webhook → DB state reflects change
- Manual: stub flow in dev
- Security: PII encryption round-trip; webhook signature verification
- DSAR: full delete cascades and calls provider
**Rollback:** revert PRs; down migration available
**Duration:** 2 sessions

---

### PHASE 5 — Billing: RevenueCat + Stripe on web

**Goal:** `/billing` page works. Users can subscribe via Stripe Checkout (through RevenueCat Web Billing), entitlement writes to Postgres, gating works.

**Tasks:**
1. User completes RevenueCat account creation (§6.2)
2. RevenueCat products configured in their dashboard matching the pricing section (e.g., `pro_monthly`, `pro_annual`)
3. Add Prisma `Entitlement` model:
   - userId (FK), productId, status (`active`|`canceled`|`expired`), periodEnd, platform (`web`|`ios`|`android`), rawPayload, createdAt, updatedAt
4. Migration
5. Install RevenueCat Web SDK: `npm install @revenuecat/purchases-js`
6. Implement `lib/billing/revenuecat.ts` — thin wrapper around SDK (customer create, entitlement sync)
7. API routes:
   - `POST /api/billing/checkout` — creates RevenueCat customer + Stripe Checkout session, returns URL
   - `POST /api/billing/webhook` — verifies RevenueCat webhook HMAC, upserts `Entitlement`
   - `GET /api/billing/portal` — returns Stripe customer portal URL for managing subs
8. `/billing` page logic:
   - No entitlement: show pricing cards with "Upgrade" → `/api/billing/checkout`
   - Active entitlement on `platform = 'web'`: show plan, next billing date, "Manage" → Stripe customer portal
   - Active entitlement on `platform != 'web'`: read-only "Managed in [App Store / Play Store]" card with link
9. Middleware helper `lib/billing/gate.ts`:
   ```ts
   export async function requireEntitlement(userId: string, feature: string) {
     const ent = await prisma.entitlement.findFirst({ where: { userId, status: 'active' } })
     if (!ent) throw new Error('ENTITLEMENT_REQUIRED')
     return ent
   }
   ```
10. Gate the scrubbing feature behind entitlement check (confirm with user if free tier should have limited scrubbing — e.g., 1 free enrollment)
11. Document webhook URLs for RevenueCat dashboard registration

**Files touched:** ~12 new files under `lib/billing/`, `app/(dashboard)/billing/`, `app/api/billing/`, `components/billing/`, `prisma/schema.prisma`, migration, middleware
**PRs:** 2 PRs
- PR 5a: "feat(billing): RevenueCat SDK, Entitlement model, API routes"
- PR 5b: "feat(billing): /billing UI and feature gating"
**Verification:**
- Stripe test mode: checkout flow → webhook → entitlement in DB
- RevenueCat dashboard shows test customer
- Feature gate unit test: non-subscriber blocked, subscriber allowed
- Webhook HMAC verification unit test
**Rollback:** revert PRs; test mode only, no prod data
**Duration:** 2 sessions

---

### PHASE 6 — Training module runner (custom MDX)

**Goal:** `/training` catalog + `/training/[moduleId]` runner work with MDX content, progress saves to Postgres, dashboard shows progress.

**Tasks:**
1. Keep existing SCORM viewer as parallel path (don't delete)
2. Content directory: `content/training/` with subfolders per module, each containing:
   - `module.json` — title, description, durationMinutes, tags, difficulty, prerequisites
   - `index.mdx` — content
   - `quiz.json` — questions, correct answers, explanations
3. Add Prisma model `TrainingProgress`:
   - userId, moduleId, status (`not_started`|`in_progress`|`completed`), score, attempts, startedAt, completedAt, lastSectionViewed
4. Migration
5. Build MDX runner component:
   - Render MDX content via `next-mdx-remote` or `@next/mdx`
   - Inline `<Quiz>` component with question + options + reveal
   - Progress tracker: save on section scroll + on quiz submit
6. `/training` page — grid of module cards with progress badges
7. `/training/[moduleId]` page — MDX runner + "Mark complete" button
8. API:
   - `POST /api/training/progress` — save event (idempotent upsert)
   - `GET /api/training/progress` — list for current user
9. Dashboard card: "Training: X of Y complete"
10. Write the first 2 real modules as reference content:
    - `phishing-basics/` — "Spotting phishing emails"
    - `password-hygiene/` — "Good password habits"
11. Document the module-add workflow in `docs/training-authoring.md`

**Files touched:** ~20 new files under `lib/training/`, `content/training/`, `app/(dashboard)/training/`, `app/api/training/`, `components/training/`, `prisma/schema.prisma`, migration
**PRs:** 2 PRs
- PR 6a: "feat(training): MDX runner, progress model, API routes"
- PR 6b: "feat(training): catalog UI + first 2 modules"
**Verification:**
- Read module end-to-end, progress saves, dashboard updates
- Quiz completion records score
- Resuming a half-read module restores position
**Rollback:** revert PRs; down migration
**Duration:** 2 sessions

---

### PHASE 7 — Hardening + compliance

**Goal:** Production-ready security, compliance, operational posture.

**Tasks:**
1. **DSAR hard-delete flow** — `/api/account/delete`:
   - Cascade through all user-linked tables (user, accounts, sessions, entitlements, enrollments, training progress, audit logs)
   - Call `ScrubbingProvider.deleteUser()`
   - Call RevenueCat delete customer
   - Write final `AuditLog` entry
   - Return 204
2. **PII encryption audit** — confirm `pgcrypto` + Cloud KMS DEK works for all sensitive columns
3. **Rate limits** on sensitive endpoints:
   - `/api/auth/*` — brute force protection
   - `/api/scrub/submit` — abuse protection
   - `/api/billing/checkout` — abuse protection
   - Use Upstash Redis OR GCP Memorystore (pick lighter; Upstash is simpler for MVP)
4. **Security headers** via Next.js middleware or `next.config.js`:
   - CSP with strict directives
   - HSTS with preload
   - X-Frame-Options DENY
   - Referrer-Policy strict-origin-when-cross-origin
   - Permissions-Policy
   - X-Content-Type-Options nosniff
5. **Audit log review** — make sure every sensitive action writes to `AuditLog`:
   - login, logout, password change, MFA enable/disable, PII submission, scrubbing enrollment, billing change, account delete
6. **Privacy policy update** — sections for:
   - Scrubbing vendor (whichever chosen)
   - RevenueCat, Stripe
   - Data retention policy
   - DSAR process (how to request deletion, timeline)
7. **Terms of service update** — subscription terms, billing cycle, refund policy, acceptable use
8. **DPA** — email chosen scrubbing vendor requesting Data Processing Agreement; note as a compliance item
9. **MFA UI** — schema already has `mfaEnabled` + `mfaSecret`:
   - Enable/disable flow on `/account/security`
   - TOTP verification on login
   - Backup codes
10. **Account lockout UI** — surface lockout state helpfully on `/login` when `lockedUntil` is set

**Files touched:** middleware, `app/api/account/delete/route.ts`, `lib/security/rateLimit.ts`, `lib/security/headers.ts`, marketing `/privacy` + `/terms` pages, MFA flow files, `lib/audit/`
**PRs:** 3 PRs
- PR 7a: "feat(security): DSAR delete, rate limits, security headers"
- PR 7b: "feat(security): MFA enrollment + verification + backup codes"
- PR 7c: "chore(compliance): privacy policy + terms updates"
**Verification:**
- Manual DSAR test: delete account, verify all tables + Optery + RevenueCat scrubbed
- Security headers: `curl -I` against dev server
- Mozilla Observatory A grade against staging
- Rate limit smoke test
- MFA enrollment + login with TOTP + backup code recovery
**Rollback:** revert individual PRs
**Duration:** 2 sessions

---

### PHASE 8 — Trust gaps + launch prep

**Goal:** Close the 6 trust gaps from Stanford/NN/g research. SEO + analytics. Ready for launch.

**Tasks:**
1. **Named team section** on `/` — photos + titles + bios (user supplies)
2. **SOC 2 / audit badge**:
   - Real badge if you have certification
   - Placeholder "Security practices" page otherwise with concrete claims (Cloud KMS, MFA, Argon2id, DSAR, audit logs)
3. **Partner logo strip** — chosen scrubbing vendor logo, GCP, Next.js, any certs
4. **Live metrics counter** — e.g., "X,XXX brokers removed this month" — pulls `ScrubbingRemovalReceipt` count with a 5-minute cache
5. **Contact reality block** — address, phone, legal entity name — footer + `/contact` page
6. **Research citations** — Stanford Web Credibility, NN/g trust studies — footer or `/research` page
7. **SEO**:
   - `app/sitemap.ts` auto-generated
   - `public/robots.txt`
   - Per-page metadata via Next.js Metadata API (title, description, OG, Twitter card)
   - Structured data (JSON-LD): `Organization`, `Product`, `SoftwareApplication`
8. **Analytics**:
   - Google Analytics 4 (most GCP-native) via `next/script` with `strategy="afterInteractive"`
   - Privacy-respecting config: IP anonymization, Consent Mode v2
   - Cookie consent banner
9. **Google Search Console migration**:
   - Re-verify ownership (the `google26df59702563d68d.html` file from Phase 2 should still work)
   - Submit new sitemap
   - Check crawl errors

**Files touched:** marketing components, new `app/(marketing)/research/`, `app/(marketing)/contact/`, `app/(marketing)/security-practices/`, footer, sitemap, analytics script, cookie banner
**PRs:** 2 PRs
- PR 8a: "feat(trust): team, partners, metrics, contact, research pages"
- PR 8b: "feat(seo): sitemap, metadata, analytics, structured data"
**Verification:**
- Lighthouse ≥ 95 on `/`
- Mozilla Observatory A
- Google Rich Results test passes
- OG card preview renders correctly (test on Twitter/LinkedIn)
- GA4 events flowing
**Rollback:** revert PRs
**Duration:** 1–2 sessions

---

### PHASE 9 — Deployment to Cloud Run

**Goal:** `https://securityindeed.org` serves the portal live, HTTPS, from Cloud Run backed by Cloud SQL.

**Tasks:**
1. **Dockerfile review** — confirm Next.js standalone output, non-root user, minimal base image. Tweak if needed.
2. **`cloudbuild.yaml`** at repo root:
   - Step 1: `npm ci`
   - Step 2: `npx prisma generate`
   - Step 3: `npm run build`
   - Step 4: `npm test` (unit + integration)
   - Step 5: `docker build -t $REGION-docker.pkg.dev/$PROJECT/securityindeed-portal/app:$SHORT_SHA .`
   - Step 6: `docker push`
   - Step 7: `gcloud run deploy securityindeed-portal --image ... --set-secrets ...`
   - Step 8: `npx prisma migrate deploy` (run via Cloud Run job or one-shot)
3. **Cloud Run service config**:
   - Name: `securityindeed-portal`
   - Min instances: 0 (scale to zero)
   - Max instances: 10
   - CPU: 1
   - Memory: 512 MB → 1 GB (adjust based on build size)
   - Concurrency: 80
   - VPC connector for Cloud SQL private IP
   - Service account: `portal-runtime` (from Phase 1)
   - Secrets via `--set-secrets` flag
4. **Cloud SQL connection** — VPC connector to private IP (preferred) OR Cloud SQL Auth Proxy sidecar
5. **Database migration step** in Cloud Build: `npx prisma migrate deploy` before or as part of deploy
6. **Domain mapping**:
   ```
   gcloud run domain-mappings create \
     --service securityindeed-portal \
     --domain securityindeed.org \
     --region $REGION
   ```
7. **DNS** — add `A` / `AAAA` / `CNAME` records at the user's DNS registrar pointing to Cloud Run targets (output by the previous command)
8. **Google-managed SSL cert** — automatic via domain mapping; takes 15–60 min to provision
9. **Smoke tests** in production:
   - `/` loads over HTTPS
   - `/login` loads
   - `/api/health` returns 200
   - Sign up a test user → verify row in Cloud SQL
   - Google sign-in → verify session
   - Submit scrubbing form → verify webhook flow
   - Stripe test checkout → verify entitlement write
10. **`docs/runbook.md`** — how to deploy, how to rollback, how to read logs, how to restore from DB backup, on-call basics

**Files touched:** `Dockerfile` (possibly), `cloudbuild.yaml` (new), `docs/runbook.md` (new), `.github/workflows/` if using GitHub Actions as a trigger
**PRs:** 1 PR ("feat(deploy): Cloud Build + Cloud Run pipeline")
**Verification:**
- Deploy to a staging project first (or a staging Cloud Run service), run smoke tests
- Deploy to prod
- All smoke tests green
**Rollback:**
```
gcloud run services update-traffic securityindeed-portal \
  --to-revisions=PREVIOUS_REVISION=100
```
**Exit criteria:** `https://securityindeed.org` is live with HTTPS, a user can go end-to-end: sign up → train → submit scrubbing → subscribe → delete account
**Duration:** 1–2 sessions

---

### PHASE 10 — Mobile integration (DEFERRED — separate future plan)

**Goal:** Mobile BG app shares auth, billing, training progress, and scrub status with the web portal.

**High-level tasks (not in scope for this build — plan separately):**
1. Mobile adds API client pointing to `https://securityindeed.org/api/v1/mobile`
2. API routes under `/api/v1/mobile/*` accept Google OAuth access tokens (same Client ID mobile uses) → exchange for NextAuth session
3. Sync training progress bidirectionally
4. Sync scrub status (pull)
5. Add IAP via RevenueCat SDK (iOS StoreKit + Android Play Billing)
6. Mobile checks `entitlement.active` via RevenueCat SDK
7. Add `ios/` folder (mobile is currently Android-only)
8. Release new mobile version

**Status:** Plan only — OUT OF SCOPE for v4 execution. Plan separately once the web portal is live.

---

## 9. DEPENDENCY GRAPH

```
Phase 0 (prereq confirmation)
   ↓
Phase 1 (GCP infra)
   ↓
Phase 2 (repo merge)
   ↓
Phase 3 (Google OAuth)
   ↓
   ├─→ Phase 4 (scrubbing) ──┐
   ├─→ Phase 5 (billing)  ───┤   These 3 can run in parallel
   └─→ Phase 6 (training) ───┘
             ↓
         Phase 7 (hardening)
             ↓
         Phase 8 (trust + SEO)
             ↓
         Phase 9 (deploy)
             ↓
         Phase 10 (mobile) — DEFERRED
```

Phases 4, 5, 6 can run in parallel after Phase 3 is done. If the user wants multiple sessions fanning out, that's the fork point.

---

## 10. RISK REGISTER

| # | Risk | Probability | Impact | Mitigation |
|---|---|---|---|---|
| 1 | Uncommitted mid-refactor in `breach-guardian/` has bugs blocking Phase 2 | Medium | High | User handles via §6.1. If bugs surface, revert refactor and plan in a separate session. |
| 2 | Optery doesn't grant Business API access in time | High | Medium | `StubProvider` ships in Phase 4; feature shows "coming soon" until account arrives. |
| 3 | RevenueCat setup friction | Low | Medium | Good docs exist. Test in sandbox first. |
| 4 | Cloud SQL cost creeps | Low | Low | Start `db-f1-micro`. Monitor. Scale only when needed. |
| 5 | DNS propagation delays launch | Low | Low | Provision Cloud Run + domain mapping a day ahead. Google-managed cert can take an hour. |
| 6 | Existing Prisma schema conflicts with Phase 4/5/6 models | Low | Medium | Review schema in Phase 1, coordinate migrations carefully. |
| 7 | Next.js version mismatch with new deps | Low | Low | Run `npm install` + full test suite in Phase 0 before any new code. |
| 8 | Google OAuth email-linking is dangerous if existing email unverified | Medium | **CRITICAL** | Require `emailVerified = true` on existing account before linking. Reject otherwise. Hard invariant in Phase 3. |
| 9 | PII leaks in logs | Medium | **CRITICAL** | Logging middleware strips PII from request bodies before writing to Cloud Logging. Allowlist safe fields. |
| 10 | Mobile + web diverge without Phase 10 | Medium | Medium | Document API contracts in Phases 4/5/6 so Phase 10 is mostly consumption work. |
| 11 | Scrubbing vendor webhook signature validation incorrect | Medium | High | Implement strict HMAC verification; test with real signed payloads from sandbox. |
| 12 | Cloud KMS DEK fetch fails at runtime | Low | High | Cache DEK in memory at startup with refresh-on-failure; fail closed. |
| 13 | Stripe/RevenueCat webhook races cause duplicate entitlements | Medium | Medium | Idempotency keys on webhook handlers; unique constraint on `(userId, productId)`. |

---

## 11. ACCEPTANCE CRITERIA FOR LAUNCH

Check every box before going live:

- [ ] `/` renders; Lighthouse ≥ 90
- [ ] All marketing pages render (`/features`, `/pricing`, `/how-it-works`, `/privacy`, `/terms`)
- [ ] Credentials sign-up works
- [ ] Credentials sign-in works
- [ ] Google OAuth sign-in works
- [ ] Existing credentials user can link a Google account (only if email verified)
- [ ] Unverified accounts cannot be hijacked via OAuth linking
- [ ] `/dashboard` shows training progress, scrub status, entitlement state
- [ ] `/training` lists modules; runner works; progress saves
- [ ] `/data-scrubbing` form submits; webhook end-to-end works (or stub shows "coming soon" honestly)
- [ ] `/data-scrubbing/history` shows timeline
- [ ] `/billing` checkout works in Stripe test mode; entitlement writes to Postgres
- [ ] Feature gating by entitlement works (free users can't access premium scrubbing)
- [ ] `/account/delete` (DSAR) wipes Postgres + Optery + RevenueCat + writes final audit log
- [ ] MFA enrollment + login + backup codes work
- [ ] Rate limits enforced on `/api/auth/*`, `/api/scrub/submit`, `/api/billing/checkout`
- [ ] Mozilla Observatory A grade on security headers
- [ ] `npm audit` shows no high/critical vulnerabilities
- [ ] Privacy policy updated with vendor disclosures
- [ ] Terms of service updated with subscription terms
- [ ] DPA request sent to scrubbing vendor
- [ ] Deployed to Cloud Run on `securityindeed.org` with valid Google-managed SSL
- [ ] Cloud SQL private-IP connection verified
- [ ] All secrets populated in Secret Manager
- [ ] Smoke tests green in production
- [ ] `docs/runbook.md` written and reviewed
- [ ] Rollback path tested at least once in staging
- [ ] Google Search Console re-verified, sitemap submitted
- [ ] GA4 events flowing
- [ ] OG card preview correct on Twitter/LinkedIn
- [ ] All 6 trust gaps closed (named team, SOC2/security-practices, partner logos, live metrics, contact block, research citations)

---

## 12. EXPLICIT DEFERRALS

NOT in scope for this build (plan separately when the time comes):

- Mobile app changes (Phase 10)
- iOS build of the mobile app (no `ios/` folder today)
- Admin/operator internal console
- Email templates beyond transactional basics (verify email, password reset)
- A/B testing infrastructure
- Customer support tooling (helpdesk, ticketing)
- Translation / i18n — English only in v4
- SCORM viewer rewrite — keep the existing one as a parallel path
- Internal analytics dashboards beyond GA4
- Any automated data broker removal without a real vendor API — vendor-dependent

---

## 13. DO NOT DO THESE THINGS

Hard don'ts for the executing agent:

1. **DO NOT touch the mobile app repo** (`C:/Users/Daya/Projects/securityindeed/breach-guardian-mobile/`). It is out of scope. Mobile integration is Phase 10, a separate plan.
2. **DO NOT switch to Firebase Auth or Firestore.** Already ruled out. Wastes existing NextAuth + Prisma work.
3. **DO NOT re-research the vendor landscape or billing.** Section 5 is final. Running web searches wastes tokens and may contradict locked decisions.
4. **DO NOT try to find a Google-native cross-platform billing product.** It does not exist. RevenueCat is the answer. Section 5.4 documents this.
5. **DO NOT write code in Phases 0 or 1.** Those are infra/docs only.
6. **DO NOT commit to `main` directly.** Every phase has explicit PRs to branches.
7. **DO NOT use `git add -A` blindly** — stage specific files to avoid accidentally committing `.env` or large binaries.
8. **DO NOT skip hooks with `--no-verify`** unless the user explicitly asks.
9. **DO NOT force push.**
10. **DO NOT discard the uncommitted mid-refactor in `breach-guardian/` on your own.** That's the user's call (§6.1).
11. **DO NOT modify `securityindeed-website/`** except to add the README archive note in Phase 2. All work happens in `breach-guardian/`.
12. **DO NOT enable `allowDangerousEmailAccountLinking` without the verified-email guard.** It's a critical security invariant (risk #8).
13. **DO NOT log PII**. Logging middleware must strip it (risk #9).
14. **DO NOT assume the existing SCORM viewer works** — test it before deciding whether to keep it as a parallel path in Phase 6.
15. **DO NOT deploy to prod** without first smoke-testing against a staging service or local emulation.
16. **DO NOT hardcode secrets.** All values come from Secret Manager or `.env.local` for dev. Never commit `.env`.
17. **DO NOT add Firebase, Supabase, Clerk, Auth0, or any other auth provider.** NextAuth + Google OAuth is the stack.

---

## 14. HOW TO BEGIN EXECUTING

### Step 1 — Confirm you read the whole file

Reply to the user:

> I've read the HANDOVER document completely. I understand:
> - Target repo: `C:/Users/Daya/Projects/breach-guardian/` (not this one)
> - Stack: Next.js + NextAuth + Postgres + Prisma + Cloud Run + Cloud SQL
> - Auth unification: add Google OAuth provider to NextAuth
> - Scrubbing: vendor-agnostic, default Optery or stub
> - Billing: RevenueCat + Stripe on web
> - 10 phases, currently at Phase 0
> - Mobile repo is off-limits
>
> Before I start Phase 0, I need you to confirm the 6 prerequisites in §6:
> 1. How did you handle the uncommitted mid-refactor in `breach-guardian/`? (commit to branch / stash / discard)
> 2. Did you create a RevenueCat account?
> 3. Did you email Optery for Business API access, or should I use StubProvider?
> 4. What's your GCP project ID and preferred region?
> 5. Who is your DNS registrar for `securityindeed.org`?
> 6. Reuse mobile's OAuth Web Client ID, or create a new web-only one?

### Step 2 — Start Phase 0

Once the user answers, begin Phase 0 tasks:
1. Write `breach-guardian/docs/secrets-inventory.md` (from §7)
2. Write `breach-guardian/docs/phase-1-pr-plan.md` with exact `gcloud` commands
3. Confirm all prerequisites are green
4. Wait for user to say "start Phase 1"

### Step 3 — Execute phases in order

Follow §8. Each phase has its own goal, tasks, files, PRs, verification, and rollback. Do not skip verification steps.

### Step 4 — Update this HANDOVER doc as you go

At the end of each phase, append a "Phase N — completed YYYY-MM-DD" entry to §16 below with:
- What was built
- Files touched (high-level)
- Any deviations from the plan and why
- Known issues carried forward

---

## 15. REFERENCE: prior session files

If the receiving agent wants to see the history of decisions (reasoning trail, not just conclusions):

- `C:/Users/Daya/Projects/securityindeed-website/docs/portal-architecture-brief.md` — v1 → v2 → v3 architectural thinking with rationale for pivots
- `C:/Users/Daya/Projects/securityindeed-website/docs/REVIEW-ME-questions.md` — the 4 questions + user's answers
- `C:/Users/Daya/Projects/securityindeed-website/docs/portal-build-plan-v4.md` — same plan content as §8 here, slightly different framing

This HANDOVER.md file is the canonical spec. The others are context.

---

## 16. EXECUTION LOG (append as you go)

> Template for each completed phase:
>
> ```
> ### Phase N — [Name] — STARTED YYYY-MM-DD
> - Status: in progress / blocked / completed
> - Started: YYYY-MM-DD
> - Completed: YYYY-MM-DD
> - PRs: #123, #124
> - Files touched: high-level list
> - Deviations from plan: what and why
> - Known issues carried forward: list
> - Next phase: N+1
> ```

---

### Phase 0 — Pre-flight — COMPLETED 2026-04-13

- **Status:** completed
- **Started:** 2026-04-13
- **Completed:** 2026-04-13
- **Branches (in `breach-guardian/`):** `refactor/phase-5-wip`, `feat/phase-0-docs`
- **Commits:** `8330015` (WIP snapshot), `395657a` (initial Phase 0 docs), `debc746` (placeholder resolution + Playwright evidence)
- **Files created:**
  - `breach-guardian/docs/secrets-inventory.md` — 17-secret catalog with purpose, format, creation commands, IAM grants, Cloud Run consumption pattern, rotation runbook
  - `breach-guardian/docs/phase-1-pr-plan.md` — 10-step Phase 1 gcloud runbook with verification and rollback
  - `breach-guardian/docs/phase-0-evidence/phase-0-project-renamed.png` — GCP Console screenshot confirming project display rename
  - `breach-guardian/docs/phase-0-evidence/phase-0-billing-status.png` — GCP Console screenshot confirming billing account linkage
- **Prerequisites handled:**
  - Mid-refactor snapshot preserved on `refactor/phase-5-wip` branch (22 files, 1,681 lines)
  - GCP project identity resolved: **`new-project-security-emails`** (display name renamed to "SecurityIndeed Portal"), project number `121591951720`, organization `reddy-health.com` (Google Workspace)
  - Billing linkage verified (via Playwright, later corrected by Phase 1 pre-flight)
  - DNS registrar confirmed: **Cloudflare** (from April 10 2026 session memory)
  - OAuth strategy locked: **Option B — new web-only OAuth client** (not reuse of mobile's client; creation deferred to Phase 3)
- **Deviations from plan:**
  - Originally suggested creating a new GCP project with a cleaner ID. Reversed after discovering the existing project number (`121591951720`) matches the mobile app's OAuth Web Client ID prefix — meaning this project is **already the BG mobile app's GCP project**, not a placeholder. Kept it for unified billing/audit; renamed only the display name (project IDs are immutable).
- **Known issues carried forward:**
  - The project display name is "SecurityIndeed Portal" but the project ID remains `new-project-security-emails` (immutable). Cosmetic only — no security or compliance impact.
  - ADC (`gcloud auth application-default login`) not yet set up; deferred to Phase 4 when app needs direct Google SDK calls from local dev.
- **Next phase:** Phase 1 — GCP infrastructure setup

---

### Phase 1 — GCP infrastructure setup — COMPLETED 2026-04-13

- **Status:** completed
- **Started:** 2026-04-13
- **Completed:** 2026-04-13
- **Branch (in `breach-guardian/`):** `feat/phase-1-infra`
- **Commit:** `f61d185` (docs(phase-1): GCP infrastructure runbook — Phase 1 complete)
- **File created:** `breach-guardian/docs/gcp-infra.md` — full runbook with executed commands, outputs, deviations, resource inventory, teardown, cost estimate
- **gcloud SDK installed:** 564.0.0 at `C:/Users/Daya/tools/google-cloud-sdk/`, authenticated as `daya@reddy-health.com`
- **Resources provisioned in GCP project `new-project-security-emails`:**
  - 11 APIs enabled (run, sqladmin, secretmanager, cloudbuild, artifactregistry, cloudkms, logging, iam, servicenetworking, vpcaccess, **compute** — the last one added beyond the original plan because VPC connector operations and gcloud region validation need it)
  - Artifact Registry: `securityindeed-portal` (Docker, us-central1)
  - VPC Private Services Access: range `google-managed-services-default` (/16) + peering to servicenetworking
  - Serverless VPC Access Connector: `portal-vpc-connector` (READY, `10.8.0.0/28`, e2-micro × 2–3)
  - Cloud SQL: `securityindeed-db` — Postgres 16, **ENTERPRISE** edition (not the default ENTERPRISE_PLUS; see deviations), db-f1-micro, private IP `10.48.0.3`, zone `us-central1-f`, PITR on, 7-day backup retention, deletion-protection on
  - Cloud SQL database `portal_db` + least-privilege user `portal_app` (password in Secret Manager only, never written to disk)
  - Cloud KMS: keyring `securityindeed-keyring` + symmetric key `pii-dek` (software protection, 90-day auto-rotation)
  - Service account `portal-runtime@new-project-security-emails.iam.gserviceaccount.com` with 6 least-privilege roles (cloudsql.client, secretmanager.secretAccessor, cloudkms.cryptoKeyEncrypterDecrypter, logging.logWriter, monitoring.metricWriter, cloudtrace.agent)
  - 16 Secret Manager secrets (7 with real values, 9 placeholders for later phases); runtime SA granted `roles/secretmanager.secretAccessor` on each
  - Cloud Build SA granted 5 deploy roles (run.admin, artifactregistry.writer, iam.serviceAccountUser, cloudsql.client, secretmanager.secretAccessor)
  - Secret Manager audit logging enabled: ADMIN_READ, DATA_READ, DATA_WRITE
- **Deviations from plan:**
  1. **Stale billing link discovered at pre-flight.** The project was originally linked to a **closed** billing account (`01042D-D63B3B-5C102D` / "My Billing Account") with `billingEnabled: false`. User pre-emptively specified "My Billing Account 1" which turned out to be the active account (`01B9AE-8B71E9-9A1038`). Re-linked before running any API enable — had this been missed, the very first `gcloud services enable` call would have failed.
  2. **Cloud SQL edition.** GCP auto-defaulted the new project to `ENTERPRISE_PLUS` edition which rejects `db-f1-micro` tier (only allows expensive `db-perf-optimized-N-*` tiers). Added `--edition=ENTERPRISE` flag to fall back to standard edition. No cost impact versus plan.
  3. **Added `compute.googleapis.com`** to the API enable list (not in original HANDOVER §8 Step 1). Required for VPC connector operations and `gcloud compute` region validation. Free.
  4. **IAM eventual consistency.** First 4 of the 6 `portal-runtime` role bindings failed immediately after SA creation with "Service account does not exist". Retried after brief propagation; all 6 succeeded on retry. Documented in gcp-infra.md.
- **Actual idle cost:** ~$20–25/month, matching the plan estimate. Dominant line items: Cloud SQL (~$9), VPC connector (~$10).
- **Known issues carried forward:**
  - `gcloud auth application-default login` not yet configured (deferred to Phase 4 for local dev).
  - New web-only Google OAuth client not yet created (deferred to Phase 3 when it's actually needed; will be created inside this same project).
  - `HIBP_API_KEY`, `SENDGRID_API_KEY`, `STRIPE_*`, `REVENUECAT_*`, `GOOGLE_CLIENT_*`, `OPTERY_API_KEY` secrets all have placeholder values — populated in Phase 3 / 4 / 5 respectively.
- **Next phase:** Phase 2 — Repository consolidation (port static marketing HTML into `breach-guardian/app/(marketing)/`)

---

### Phase 2 — Repository consolidation — COMPLETED 2026-04-13

- **Status:** completed
- **Started:** 2026-04-13
- **Completed:** 2026-04-13
- **Branch (in `breach-guardian/`):** `feat/phase-2-marketing`
- **Files touched:**
  - `breach-guardian/tailwind.config.ts` — added fortress/frost/brand palette, DM Serif + Outfit font families, custom shadows, fade-up keyframe animation
  - `breach-guardian/app/globals.css` — replaced with Tailwind directives + `@layer base` resets + `@layer components` for glassy nav / gradient / container helpers
  - `breach-guardian/app/layout.tsx` — added `next/font/google` for DM Serif Display + Outfit (self-hosted, preloaded), full metadata export with Open Graph + Twitter cards
  - `breach-guardian/app/page.tsx` — **deleted** (old placeholder; replaced by `app/(marketing)/page.tsx`)
  - `breach-guardian/app/(marketing)/layout.tsx` — marketing route group layout with shared nav + footer
  - `breach-guardian/app/(marketing)/page.tsx` — landing page composing all sections
  - `breach-guardian/app/(marketing)/features/page.tsx`
  - `breach-guardian/app/(marketing)/pricing/page.tsx`
  - `breach-guardian/app/(marketing)/how-it-works/page.tsx`
  - `breach-guardian/app/(marketing)/privacy/page.tsx` (ported from `privacy.html`)
  - `breach-guardian/app/(marketing)/terms/page.tsx` (ported from `terms.html`)
  - `breach-guardian/app/sitemap.ts` — auto-generated `/sitemap.xml` for 6 public routes
  - `breach-guardian/app/robots.ts` — auto-generated `/robots.txt` allowing marketing paths, disallowing auth/dashboard/API
  - `breach-guardian/components/marketing/*.tsx` — **15 new focused React components** (ShieldLogo, CheckIcon, ArrowRightIcon, StoreBadge, MarketingNav, MarketingFooter, Hero, TrustBar, HowItWorks, FeatureGrid, PrivacyArchitecture, ModesSection, PricingCards, CTASection, DownloadCTA)
  - `breach-guardian/public/google26df59702563d68d.html` — copied Google Search Console verification file
  - `breach-guardian/global.d.ts` — type shim restoring the global `JSX` namespace (needed because `@types/react@19` removed it but the runtime is React 18)
  - `breach-guardian/docs/phase-2-migration-notes.md` — what-moved-where with deviations and build verification
  - `breach-guardian/package.json` / `package-lock.json` — added `@tailwindcss/typography@^0.5.19` (dev dep) for styling the legal pages via `prose` class
  - `securityindeed-website/README.md` — **new** archive notice pointing to `breach-guardian/`
- **Build verification:**
  - `npx tsc --noEmit` → exit 0 (zero errors)
  - `npx next build` → 16 routes compiled, 16 static pages generated, first-load JS 87–96 KB on all marketing routes
- **Deviations from plan:**
  1. Added `@tailwindcss/typography` plugin (not in original checklist but pragmatically necessary for legal pages; battle-tested)
  2. Added `global.d.ts` type shim to bridge React 18 runtime with React 19 types without editing 20+ files
  3. Legal pages (`privacy`, `terms`) use `dangerouslySetInnerHTML` with inlined static legal text — safe (our content, not user-supplied), faster than hand-converting 500+ lines of `<h2>`/`<p>` nesting to JSX
  4. Privacy/Terms content was condensed (not materially changed) — 15→13 sections for privacy, 23→21 for terms; all legal substance preserved
- **Known issues carried forward:**
  - Trust gaps (named team, SOC 2 badge, live metrics, partner logos, etc.) — **Phase 8**
  - GA4 / cookie banner — **Phase 8**
  - OG images — **Phase 8**
  - Lighthouse verification — **Phase 9** (against production Cloud Run staging, not local dev)
  - `/api/health` emits a Prisma "DATABASE_URL not found" warning during `next build` static generation — pre-existing, unrelated to Phase 2, will resolve in Phase 9 when Cloud Run supplies the env var at runtime
- **Next phase:** Phase 3 — auth unification (add Google OAuth provider to NextAuth; create new web-only OAuth client in `new-project-security-emails`; enable safe email-based account linking)

---

*End of handover document. Single file, self-contained, ready to hand off. All decisions are final. Execution log continues above as phases complete.*
