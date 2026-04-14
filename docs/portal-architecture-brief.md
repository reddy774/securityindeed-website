# SecurityIndeed.org — Portal Architecture Brief (v3)

**Status:** Research complete. Major pivot discovered. Decision needed from user.
**Date:** 2026-04-13
**Author:** Claude (Opus 4.6) working session
**Next step:** User reads the **HEADLINE PIVOTS** in §1 and answers the four questions in §12.

---

## 🚨 HEADLINE PIVOTS (read this first)

After inspecting your existing projects and running the research, the plan from v1/v2 needs to change in four big ways. Read these carefully — everything in the rest of the document flows from them.

### Pivot 1 — **The portal already exists. We shouldn't start from scratch.**
You already have a Next.js + NextAuth + Postgres + Prisma web app at **`C:/Users/Daya/Projects/breach-guardian/`** that is 60–70% of what you described wanting to build. It already has:
- Login / register pages
- Dashboard route
- Training route (with SCORM course viewer)
- API routes for brokers, training, scans, webhooks, audit, users
- **An Optery API key slot already in `.env.example`**
- Argon2id password hashing, account lockout, MFA fields in schema
- Docker-ready, production build artifacts (`.next/standalone`)
- It's **mid-refactor** — has uncommitted changes in `app/(auth)/`, `app/(dashboard)/`, `components/`, `lib/stores/`

**Implication:** The most efficient path is to **finish the existing `breach-guardian/` app** and deploy *that* to Cloud Run on `securityindeed.org`, not build a new one. The static `securityindeed-website` HTML either becomes obsolete or survives as a pure marketing shell at a subdomain. See §4 for options.

### Pivot 2 — **Firebase Auth + Firestore was the wrong recommendation. Ignore it.**
I was wrong in v1/v2. The existing stack is:
- **Auth:** NextAuth.js v4 with Credentials Provider (email + password + Argon2id)
- **DB:** PostgreSQL via Prisma
- **Not Firebase.** No Firebase Auth, no Firestore, no Firebase Admin SDK anywhere.

Re-architecting to Firebase would throw away all the work that already exists in `breach-guardian/`. The correct architecture is:
- **Keep** NextAuth + Postgres + Prisma on the web side
- **Add** a Google OAuth provider to NextAuth (so mobile users can sign into the web with the same Google account they already use in the app)
- **Host** Postgres on **Cloud SQL (Postgres)** on GCP — that IS GCP-native
- Deploy Next.js app as container on **Cloud Run**

### Pivot 3 — **The mobile app uses Google OAuth, NOT Firebase Auth.**
Inspected `C:/Users/Daya/Projects/securityindeed/breach-guardian-mobile/`:
- **Framework:** Expo (React Native 0.76.9), Android-only (no `ios/` folder yet)
- **Auth:** Google OAuth 2.0 (PKCE) via `@react-native-google-signin/google-signin` + `expo-auth-session`
- **Web Client ID:** `121591951720-egglhh0qi98v04rgg9qr0jiukfrqpji4.apps.googleusercontent.com`
- **iOS Client ID:** `121591951720-i0pa7v35dnga89jrmpl36p4fa1smu7sv.apps.googleusercontent.com`
- **Backend:** NONE. App is fully on-device. SQLite + zustand. Uses Gmail API read-only to fetch breach-related emails.
- **No IAP / no billing.**

**Implication for shared auth:** This is actually GOOD news. Since the mobile app already uses **Google OAuth** (with the user's Gmail identity), the cleanest unification path is to **add a Google OAuth provider to the web app's NextAuth**. A user who signed into the mobile app with `user@gmail.com` can then sign into the web portal with the same Google account — NextAuth will link the accounts by email. Roughly ~200 lines of NextAuth config + one Google Cloud Console setup. No migration needed on the mobile side.

### Pivot 4 — **There is NO single Google-native cross-platform billing. That product doesn't exist.**
Confirmed by research. Reality:
- **Apple requires StoreKit (IAP)** for iOS subscriptions (15–30% cut, with 2025 US external-link carve-out)
- **Google requires Play Billing** for Android subscriptions (also mandatory, Android-only)
- **Google Play Billing does NOT work on iOS or web.** Google has never shipped a cross-platform subscription product.
- **Web-only:** Stripe

**Realistic unified pattern: RevenueCat** as the entitlement layer sitting on top of StoreKit + Play Billing + Stripe. Each platform processes the payment through its native system; RevenueCat normalizes it into one `entitlement.active` flag per user, synced to your Postgres via webhooks. Pricing: free up to $2.5k MTR, then 1% of tracked revenue. Alternative: **Adapty** (free under $5k MRR, then 1%).

Use **RevenueCat** on web (via their Web Billing SDK which wraps Stripe) + mobile apps (when they add IAP). Your `/billing` page on the web becomes a Stripe Checkout flow under the hood, with entitlements mirrored to Postgres.

---

## 0. Recovered context from prior sessions
*(unchanged from v2)*

"Google space" = GCP Cloud Run. Product is Breach Guardian: on-device AI scam detection, breach monitoring, data broker removal, Easy Mode for seniors. Palette: fortress/frost. Fonts: DM Serif Display + Outfit. 794-line static `index.html` in `securityindeed-website/` already exists.

---

## 1. Actual project landscape (three repos, not one)

| Repo | Path | What it is | State |
|---|---|---|---|
| **Marketing static** | `C:/Users/Daya/Projects/securityindeed-website/` (WE ARE HERE) | 794-line static HTML landing page, Google Search Console verified | Live, deployed somewhere |
| **Portal (Next.js app)** | `C:/Users/Daya/Projects/breach-guardian/` | Next.js + NextAuth + Postgres + Prisma. Login, register, dashboard, training, brokers, api routes. Optery env var. | **Mid-refactor**, uncommitted changes, last commit "Phase 5 testing + CI/CD" |
| **Mobile app** | `C:/Users/Daya/Projects/securityindeed/breach-guardian-mobile/` | Expo RN, Android-only, Google OAuth, on-device SQLite, Gmail API integration | Active dev, clean tree, commit `a0c9f88` |

**These three are architecturally independent.** They do not share a backend, a user database, or an auth system today. The v3 plan is to make them share one (option C in §4 below).

---

## 2. Architecture — corrected (v3)

| Layer | Choice | Status |
|---|---|---|
| **Frontend + API** | **Next.js 15 App Router** (extend the existing `breach-guardian/` app) | ✅ LOCKED |
| **Auth** | **NextAuth.js v4** (existing) + **add Google OAuth provider** | ✅ LOCKED — unifies with mobile's existing Google OAuth |
| **Database** | **Cloud SQL (PostgreSQL)** on GCP, via Prisma | ✅ LOCKED — most GCP-native relational, keeps existing Prisma schema |
| **Hosting** | **Cloud Run** (containerized Next.js, existing Dockerfile) | ✅ LOCKED |
| **CI/CD** | Cloud Build → Artifact Registry → Cloud Run | ✅ LOCKED |
| **Domain/SSL** | Cloud Run domain mapping + Google-managed cert on `securityindeed.org` | ✅ LOCKED |
| **Secrets** | **Secret Manager** | ✅ LOCKED |
| **PII encryption at rest** | **Cloud KMS** + `pgcrypto` for sensitive columns | ✅ LOCKED |
| **Scrubbing integration** | **Vendor-agnostic `ScrubbingProvider` interface** with Optery as default implementation | ✅ LOCKED |
| **Training format** | **Custom MDX + progress events in Postgres** (replaces SCORM viewer in existing app — or keep SCORM optional) | ✅ LOCKED — "best and economical" |
| **Billing** | **RevenueCat** (Web Billing SDK) + Stripe under the hood for web; StoreKit / Play Billing in mobile when ready; single entitlements table in Postgres | ✅ LOCKED |
| **Logging / audit** | Cloud Logging + Cloud Audit Logs + existing Prisma `AuditLog` model | ✅ LOCKED |

**One-line summary:**
**Finish and deploy the existing `breach-guardian/` Next.js app to Cloud Run + Cloud SQL, add Google OAuth + RevenueCat + vendor-agnostic scrubbing, host on `securityindeed.org`.**

---

## 3. Data scrubbing — vendor landscape & abstraction

### The real vendor landscape (2026 research)

| Vendor | Public API? | Verdict |
|---|---|---|
| **Optery** | ✅ YES — clean, public OpenAPI sandbox, webhooks, 640+ brokers, SOC 2 Type II | **Primary choice** |
| **Onerep** | ✅ YES — REST + webhooks, 310+ brokers | Secondary choice (reputational flag: CEO 2024 scandal) |
| **Privacy Bee** | ✅ YES — enterprise OAuth 2.0, webhooks, B2B slant | Tertiary |
| **DeleteMe** | ❌ No consumer-removal partner API (only breach-data API) | Dealbreaker |
| **Kanary** | ❌ No public API | Dealbreaker |
| **Incogni** | ❌ No public API (Surfshark-bundled consumer only) | Dealbreaker |
| **Privacy Duck** | ❌ No API | Dealbreaker |

**Only 3 real options.** Build the abstraction to the intersection of these three.

### The `ScrubbingProvider` interface (target)

```ts
interface ScrubbingProvider {
  enrollUser(profile: PIIProfile): Promise<{ providerId: string }>
  updateProfile(providerId: string, patch: Partial<PIIProfile>): Promise<void>
  triggerScan(providerId: string): Promise<{ scanId: string }>
  getStatus(providerId: string): Promise<EnrollmentStatus>
  listBrokers(providerId: string): Promise<BrokerRecord[]>
  listRemovals(providerId: string): Promise<RemovalReceipt[]>
  verifyWebhook(headers, rawBody): boolean
  parseWebhook(payload): NormalizedEvent
  deleteUser(providerId: string): Promise<void>
}
```

### `PIIProfile` schema (intersection of all 3)

Required fields across all viable vendors:
- `firstName`, `lastName`, `middleName?`, `aliases[]`
- `dateOfBirth` (ISO-8601)
- `emails[]` (1+)
- `phones[]`
- `addresses[]` (current + historical, with from/to dates)
- `relatives[]` (optional but boosts Optery/Onerep match quality)
- `gender?`
- `consent` block (TOS timestamp, IP, authorized-agent flag — legally required for Optery/Onerep)

Union-only extras, per-provider optional: `employer`, `education`, `socialHandles`, `vehiclePlates`.

**Primary implementation: OpteryProvider.** Fallback implementations for Onerep and PrivacyBee can be added later.

---

## 4. Decision 1 revisited — now that we know three repos exist

Your question was "what do you mean by one app vs split?" Now that we've found the existing `breach-guardian/` portal, the decision is clearer and there are **three real options**:

### Option A — Merge everything into the existing `breach-guardian/` app
- Move the marketing pages (hero, features, pricing, how-it-works, privacy, terms) from static `securityindeed-website/index.html` INTO the Next.js app
- The Next.js app at `breach-guardian/` becomes the entire `securityindeed.org` — marketing + portal
- `securityindeed-website/` static HTML repo is archived
- **One repo, one deploy, one domain.**
- **Pros:** simplest, one SEO footprint, one set of deploys, most consistent UX
- **Cons:** marketing changes ship alongside portal changes; portal bugs can affect landing page uptime

### Option B — Keep marketing static, portal on subdomain
- `securityindeed.org` stays as the existing 794-line static HTML in `securityindeed-website/` (or migrate it to a tiny Next.js SSG build)
- Portal lives at `app.securityindeed.org` from the existing `breach-guardian/` Next.js app
- Two repos, two deploys, two domains
- **Pros:** marketing is bulletproof static, portal can have heavy downtime without affecting the landing page, teams can split later
- **Cons:** two deploys, cookie/CORS cross-subdomain setup, SEO split

### Option C — New consolidated repo
- Start a new repo, move the good parts from `breach-guardian/` (auth, Prisma schema, Optery env, training routes) + marketing HTML into it
- Rebuild with v3 architecture as the organizing principle
- **Pros:** clean slate, consistent architecture from day one
- **Cons:** throws away the "Phase 5 testing + CI/CD" work already in `breach-guardian/`, highest effort, slowest to ship

**My recommendation: Option A.** Finish what's already 60–70% built in `breach-guardian/`, merge the marketing HTML into it, deploy the whole thing as one app to Cloud Run on `securityindeed.org`. Fastest to production, lowest waste.

---

## 5. Billing architecture

### The reality (confirmed by research)
- Apple requires StoreKit / IAP for iOS subs (15–30% cut, 2025 US external-link carve-out)
- Google requires Play Billing for Android subs (Android-only, mandatory, EU DMA carve-outs)
- Google Play Billing is NOT cross-platform. There is no "Google-native unified billing."
- Web-only: Stripe

### The unified pattern — RevenueCat

```
  ┌─ iOS app ─────────→ StoreKit (IAP) ─┐
  │                                      ↓
  │                                 RevenueCat ──webhook──→ Postgres
  │                                      ↑                  (entitlements)
  ├─ Android app ────→ Play Billing ────┤
  │                                      ↑
  └─ securityindeed.org/billing ──→ RevenueCat Web Billing ─→ Stripe
```

One `entitlements` table in Postgres, keyed by user id, fed by RevenueCat webhooks. Your app/portal just checks `entitlement.active = true`. Platform-agnostic.

**Pricing:** Free up to $2.5k monthly tracked revenue, then 1% of tracked revenue. Negligible for MVP.

**Alternative:** Adapty. Free under $5k MRR, then 1%. Very similar SDK surface. Either works.

### For the `/billing` page
- New users who sign up on web → Stripe Checkout via RevenueCat Web Billing
- Users who paid on mobile → read-only status card, "Manage your subscription in the App Store / Play Store" link
- Upgrade, cancel, payment method, invoices — standard Stripe customer portal embedded

---

## 6. Information architecture (target state, Option A)

### Public (no auth) — at `securityindeed.org`
- `/` — landing (migrated from current `index.html`)
- `/features`
- `/pricing`
- `/how-it-works`
- `/privacy` (migrated)
- `/terms` (migrated)
- `/login`, `/register`, `/forgot-password`, `/verify-email`

### Authed (behind NextAuth middleware)
- `/dashboard` — overview: training %, scrub status, alerts, subscription state
- `/training` — module catalog
- `/training/[moduleId]` — MDX content + progress save (or keep SCORM viewer as a parallel option)
- `/data-scrubbing` — submission form + live status
- `/data-scrubbing/history` — past submissions, removal receipts
- `/billing` — plan, invoices, upgrade/cancel, payment method (NEW)
- `/account` — profile, password, email, MFA setup (existing schema supports)
- `/logout`

### Server API routes (already partially exist in `breach-guardian/`)
- `/api/auth/[...nextauth]` — NextAuth (existing)
- `/api/training/progress` — existing
- `/api/brokers` — existing, wire to `ScrubbingProvider`
- `/api/scrub/submit`, `/api/scrub/status`, `/api/scrub/webhook` — NEW (vendor-agnostic)
- `/api/billing/checkout`, `/api/billing/webhook` — NEW (RevenueCat/Stripe)
- `/api/account/delete` — DSAR hard-delete (NEW)
- `/api/audit` — existing

---

## 7. Phased delivery plan (target v4)

1. **Phase 0 — Decisions & setup** (this session + user replies)
2. **Phase 1 — Unify repos** (Option A): migrate marketing HTML into `breach-guardian/` Next.js app, finish the mid-refactor, land all uncommitted changes with tests
3. **Phase 2 — Auth unification:** add Google OAuth provider to NextAuth so mobile + web share identity via Gmail
4. **Phase 3 — Infrastructure:** GCP project setup, Cloud SQL Postgres, Secret Manager, Cloud Build pipeline, Cloud Run service, domain mapping, SSL
5. **Phase 4 — `ScrubbingProvider` abstraction + OpteryProvider implementation** with webhook endpoint + Postgres persistence
6. **Phase 5 — `/billing` page:** RevenueCat integration, Stripe checkout, entitlements table, webhook handling
7. **Phase 6 — Training runner:** custom MDX runner, progress events, dashboard surface
8. **Phase 7 — Hardening:** DSAR delete flow, PII encryption via `pgcrypto` + Cloud KMS DEK, rate limits, CSP/HSTS, audit log review, privacy policy update, DPA with chosen scrubbing vendor
9. **Phase 8 — Launch prep:** trust gaps (named team, SOC2 placeholder, partner logos, live metrics, contact block, research citations), SEO, analytics, Search Console migration
10. **Phase 9 — Mobile app integration:** add IAP via RevenueCat to mobile + wire Google OAuth to mutual backend (separate project, but the plan should unblock it)

Each phase will be broken down into concrete tasks in the v4 plan doc.

---

## 8. Biggest risks (updated v3)

1. ~~Vendor lock-in on scrubbing~~ → Mitigated by `ScrubbingProvider` abstraction
2. ~~Shared auth~~ → Mitigated: mobile uses Google OAuth, add the same provider to NextAuth (~200 LOC)
3. **PII compliance** — DSAR delete flow, Cloud KMS + pgcrypto encryption, retention policy, privacy policy update, DPA with scrubbing vendor
4. **Scope** — still ~3–5× original ask, but much less than v1/v2 assumed because `breach-guardian/` already exists
5. **Cross-platform billing complexity** — managed by RevenueCat
6. **The existing `breach-guardian/` uncommitted changes** — the mid-refactor has to be landed cleanly before we add new features. If it's half-broken, first task is "finish and stabilize"
7. **Three-repo fragmentation** — resolved by choosing Option A / B / C
8. **Mobile has no backend currently** — on-device-only. Adding cloud sync for training progress / scrub status is a net-new backend API surface for the mobile side. Not blocking the web portal, but the user should be aware.

---

## 9. Decisions log

| # | Decision | Status | Locked by |
|---|---|---|---|
| 1 | One app at one domain vs split | ⚠️ Option A recommended, awaiting user confirm | — |
| 2 | Database | ✅ **Postgres (Cloud SQL)** — keeps existing Prisma schema, GCP-native | Research + investigator |
| 3 | Training format | ✅ **Custom MDX + progress in Postgres** (existing SCORM viewer can stay as parallel option) | User: "best and economical" |
| 4 | Mobile uses Firebase Auth? | ✅ **NO** — uses Google OAuth 2.0 (PKCE) | Investigator |
| 5 | Scrubbing vendor relationship | ✅ **Vendor-agnostic abstraction** with Optery as default | User: "vendor agnostic"; research: only 3 viable |
| 6 | Stack | ✅ **Next.js 15 + NextAuth + Postgres + Prisma** (existing), not Firebase | Investigator |
| 7 | Auth unification | ✅ **Add Google OAuth provider to NextAuth** | Investigator + architectural fit |
| 8 | Billing | ✅ **RevenueCat unified entitlement layer** (Stripe under hood on web) | Research — no Google-native cross-platform billing exists |
| 9 | Scrubbing default impl | ✅ **Optery** | Research: only 3 viable (Optery, Onerep, Privacy Bee); Optery cleanest |

---

## 10. What I already did this session

1. ✅ Recovered context from past session logs (v1)
2. ✅ Enhanced scope brief (v2)
3. ✅ Inspected `Projects/breach-guardian-mobile/` (auth, framework, billing, backend) — **done**
4. ✅ Inspected `Projects/breach-guardian/` (is-it-the-same, routes, stack, git state) — **done**
5. ✅ Researched scrubbing vendor API landscape — **done** (Optery / Onerep / Privacy Bee viable; 4 others dealbreakers)
6. ✅ Researched cross-platform billing reality — **done** (no Google-native exists; RevenueCat is the answer)
7. ✅ Rewrote architecture plan with corrected stack (v3)
8. ⏸ Waiting on user decisions before writing v4 formal build plan

---

## 11. What I have NOT done (and will not until user approves)

- No code written
- No files modified in `breach-guardian/` (the other repo)
- No files modified in `breach-guardian-mobile/`
- No GCP resources created
- No RevenueCat or Stripe accounts touched
- No DNS changes

---

## 12. User — please reply with four things

**1. Repo consolidation (the most important decision now):**
- Option A: Merge `securityindeed-website` marketing HTML INTO the existing `breach-guardian/` Next.js app. One repo, one deploy at `securityindeed.org`. (**I recommend this.**)
- Option B: Keep `securityindeed-website` as static marketing at root domain, deploy `breach-guardian/` at `app.securityindeed.org`. Two repos, two deploys.
- Option C: Start a new consolidated repo, port good parts from both.

**2. The existing `breach-guardian/` app has uncommitted mid-refactor changes.** Do you want me to:
- (a) Inspect them in detail and produce a "finish the refactor" plan as Phase 1, OR
- (b) Leave that to another session, OR
- (c) Ask someone else (the original author) what state they were in?

**3. Accounts / relationships you already have:**
- ✅ / ❌ GCP paid project (confirmed earlier — ✅)
- ✅ / ❌ Google Cloud Console OAuth client already exists (you do — the mobile app has it)
- ✅ / ❌ Stripe account
- ✅ / ❌ RevenueCat account
- ✅ / ❌ Apple Developer account (for when mobile adds iOS + IAP)
- ✅ / ❌ Google Play Console account
- ✅ / ❌ Domain `securityindeed.org` registered (you have this — Search Console verified)
- ✅ / ❌ Optery Business account (the `.env.example` has a slot but is it filled?)

**4. Do you want me to proceed to write the v4 formal build plan doc?**
Yes / no. If yes, after your answers to 1–3 I will write `docs/portal-build-plan-v4.md` with concrete phased tasks, file-by-file scope, deployment runbook, and a PR sequencing plan. Still no code, just the written plan.

---

## 13. Files I touched this session (read-only or plan doc only)

- ✅ `docs/portal-architecture-brief.md` — created v1, updated v2, rewritten as v3 (this file)
- Read-only: `C:/Users/Daya/Projects/securityindeed-website/index.html`
- Read-only (via investigator agent): `C:/Users/Daya/Projects/breach-guardian/package.json`, Prisma schema, NextAuth config, app routes, Dockerfile, .env.example, git state
- Read-only (via investigator agent): `C:/Users/Daya/Projects/securityindeed/breach-guardian-mobile/` package.json, auth code, google-services config, git state
- Read-only (via research agent): optery.com, onerep.com, privacybee.dev, revenuecat.com, apple/google policy pages

No source code in any project was modified. No builds run. No deploys triggered.

---

*v3 saved. Living doc. No code written. Awaiting user answers to §12.*
