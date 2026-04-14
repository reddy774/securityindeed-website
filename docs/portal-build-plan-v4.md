# SecurityIndeed.org — Formal Build Plan (v4)

**Status:** Build plan locked. Awaiting user go-ahead to start executing Phase 0.
**Date:** 2026-04-13
**Supersedes:** `docs/portal-architecture-brief.md` v1–v3 (context), `docs/REVIEW-ME-questions.md` (answers)
**Target repo:** `C:/Users/Daya/Projects/breach-guardian/` (NOT this repo)
**Target domain:** `securityindeed.org`
**Target cloud:** GCP (Cloud Run + Cloud SQL + Secret Manager + Cloud KMS + Cloud Build + Artifact Registry)
**Code status:** ZERO code in this plan. Pure written spec.

---

## 0. Executive summary

Finish the existing `breach-guardian/` Next.js app, merge in the marketing HTML from `securityindeed-website/`, add Google OAuth to NextAuth, build a vendor-agnostic data-scrubbing abstraction, add RevenueCat-unified billing, ship it to Cloud Run on `securityindeed.org`.

**10 phases.** Phases 0–2 are prerequisites + consolidation. Phases 3–6 are feature work. Phases 7–9 are hardening + launch. Phase 10 is mobile integration (deferred).

---

## 1. Locked decisions (from v3 brief + user answers)

| # | Decision | Lock |
|---|---|---|
| 1 | Repo consolidation | **Option A** — one repo, one deploy. Marketing HTML migrates INTO `breach-guardian/`. `securityindeed-website/` gets archived after migration. |
| 2 | Database | **Cloud SQL PostgreSQL** (GCP-native, keeps existing Prisma schema) |
| 3 | Auth | **NextAuth.js v4** (existing) + **add Google OAuth provider** |
| 4 | Training format | **Custom MDX + progress events in Postgres** (parallel path to existing SCORM viewer) |
| 5 | Scrubbing vendor | **Vendor-agnostic `ScrubbingProvider` interface**, default impl = Optery |
| 6 | Billing | **RevenueCat** as unified entitlement layer (Stripe on web, StoreKit + Play Billing on mobile later) |
| 7 | Hosting | **Cloud Run** (containerized Next.js, existing Dockerfile) |
| 8 | CI/CD | **Cloud Build** → Artifact Registry → Cloud Run |
| 9 | Mid-refactor in `breach-guardian/` | **Deferred** — user handles. Plan builds on top of the committed state. |
| 10 | Mobile changes | **Deferred to Phase 10.** Mobile stays as-is (Expo + Google OAuth + on-device) until portal ships. |

---

## 2. Prerequisites — things only the user can do (Phase -1)

Before I start Phase 0, the user must complete these manual steps. None involve code.

### 2.1 Repo state
1. In `C:/Users/Daya/Projects/breach-guardian/`: decide what to do with the uncommitted mid-refactor (`app/(auth)/`, `app/(dashboard)/`, `components/`, `lib/stores/` untracked + `app/globals.css`, `app/layout.tsx`, `app/page.tsx` modified).
   - **Option A:** commit them to a branch (`git checkout -b refactor/phase-5 && git add -A && git commit -m "WIP: in-progress refactor"`)
   - **Option B:** stash them (`git stash push -u -m "mid-refactor wip"`)
   - **Option C:** discard if abandoned (`git restore . && git clean -fd`) — DESTRUCTIVE, only if you're sure
   - **Tell me which** before Phase 0 starts.

### 2.2 Accounts to create (user does these in browser)
| Account | Action | Why |
|---|---|---|
| **RevenueCat** | Sign up at revenuecat.com. Create project "SecurityIndeed". Note the public SDK key + webhook secret. | Billing Phase 5 |
| **Optery** (or alternative) | Email Optery at partners@optery.com requesting Business API access. **OR** decide to stub the scrubbing feature (show "coming soon") until an account is ready. | Scrubbing Phase 4 |
| **Cloud SQL** | Will be provisioned by me in Phase 1 via `gcloud` commands. You just need to confirm the GCP project ID. | Infra Phase 1 |

### 2.3 GCP project info I need from user
Please provide (paste into chat or edit this file):
- **GCP project ID:** `_______________`
- **GCP project number:** `_______________`
- **Preferred region:** `us-central1` recommended (cheapest, lowest latency for most US users) — confirm or override
- **Existing OAuth Web Client ID:** `121591951720-egglhh0qi98v04rgg9qr0jiukfrqpji4.apps.googleusercontent.com` (from mobile app) — reuse or create new web-only client?
- **Billing account status:** Pro confirmed earlier — just re-confirm active

### 2.4 DNS / domain
- `securityindeed.org` is registered and Search Console verified. Confirm which registrar hosts the DNS (GoDaddy? Namecheap? Google Domains?) so we know where to add the Cloud Run CNAME later.

### 2.5 Secrets inventory (will be created in Phase 1, just noting what's needed)
- `NEXTAUTH_SECRET` (generate new)
- `NEXTAUTH_URL` = `https://securityindeed.org`
- `DATABASE_URL` = Cloud SQL connection string
- `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` (from GCP console)
- `REVENUECAT_PUBLIC_SDK_KEY`, `REVENUECAT_WEBHOOK_SECRET`
- `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` (from Stripe dashboard)
- `OPTERY_API_KEY` (when available; stub otherwise)
- `KMS_KEY_RESOURCE_NAME` (created in Phase 1)
- `GMAIL_APP_PASSWORD` or SendGrid/Postmark key for transactional email (verify, password reset)

All stored in **GCP Secret Manager**, never in code.

---

## 3. Phase breakdown

Each phase: goal, tasks, files touched, PR sequence, verification, rollback.

---

### Phase 0 — Pre-flight (session prep, no code) 🎯

**Goal:** Everything ready to start Phase 1 with zero blockers.

**Tasks:**
1. User completes §2.1 (refactor state decision) and tells me
2. User creates RevenueCat account (§2.2)
3. User emails Optery (or decides to stub) (§2.2)
4. User provides GCP project ID + region (§2.3)
5. User confirms DNS registrar (§2.4)
6. I write a `docs/secrets-inventory.md` checklist
7. I generate a PR plan file-by-file for Phase 1

**Files I touch:** `docs/secrets-inventory.md`, `docs/phase-1-pr-plan.md`
**Files in `breach-guardian/`:** none
**PRs:** none (docs only)
**Verification:** User confirms all prereqs ✅
**Rollback:** delete the new docs

**Exit criteria:** User explicitly says "start Phase 1."

---

### Phase 1 — GCP infrastructure setup 🏗️

**Goal:** All GCP resources provisioned and documented, ready for the app to deploy into.

**Tasks:**
1. Enable GCP APIs: `run.googleapis.com`, `sqladmin.googleapis.com`, `secretmanager.googleapis.com`, `cloudbuild.googleapis.com`, `artifactregistry.googleapis.com`, `cloudkms.googleapis.com`, `logging.googleapis.com`, `iam.googleapis.com`
2. Create Artifact Registry repo for Docker images: `securityindeed-portal` in chosen region
3. Provision **Cloud SQL PostgreSQL 16** instance: `securityindeed-db`, tier `db-f1-micro` for MVP (upgradable), **private IP + VPC connector** for Cloud Run access, automated backups, point-in-time recovery ON
4. Create app DB + user: `portal_db`, `portal_app` role (no superuser)
5. Create **Cloud KMS keyring** `securityindeed-keyring` + symmetric key `pii-dek` (for PII envelope encryption)
6. Create **Secret Manager secrets** (values added later): all items in §2.5
7. Create **service account** `portal-runtime@PROJECT.iam.gserviceaccount.com` with roles: `roles/run.invoker`, `roles/cloudsql.client`, `roles/secretmanager.secretAccessor`, `roles/cloudkms.cryptoKeyEncrypterDecrypter`, `roles/logging.logWriter`
8. Create **Cloud Build service account** permissions for deploy
9. Document all of the above in `docs/gcp-infra.md` with exact `gcloud` commands used

**Tools I'll use:** `gcloud` CLI (requires user to be logged in or to grant me permission to run commands)
**Files touched in `breach-guardian/`:** none yet
**Files added in `securityindeed-website/docs/`:** `gcp-infra.md`, `secrets-inventory.md` (populated)
**PRs:** none (infra only, no code)
**Verification:**
- `gcloud sql instances describe securityindeed-db` returns RUNNABLE
- `gcloud kms keys list` shows `pii-dek`
- `gcloud secrets list` shows all placeholder secrets
- `gcloud artifacts repositories list` shows `securityindeed-portal`
**Rollback:** `gcloud sql instances delete`, `gcloud kms keys destroy`, etc. All scripted in `docs/gcp-infra.md`.

**Exit criteria:** All GCP resources created, documented, reachable. Cost check: should be ~$10–15/month at idle (Cloud SQL dominates; Cloud Run is free tier).

**Duration:** 1 session.

---

### Phase 2 — Repository consolidation (Option A) 📦

**Goal:** Marketing HTML from `securityindeed-website/` is fully migrated into `breach-guardian/`. Static repo can be archived.

**Tasks:**
1. In `breach-guardian/`, create new marketing routes under `app/(marketing)/`:
   - `app/(marketing)/page.tsx` — landing (ported from `securityindeed-website/index.html`)
   - `app/(marketing)/features/page.tsx`
   - `app/(marketing)/pricing/page.tsx`
   - `app/(marketing)/how-it-works/page.tsx`
   - `app/(marketing)/privacy/page.tsx` (ported from `privacy.html`)
   - `app/(marketing)/terms/page.tsx` (ported from `terms.html`)
2. Port CSS:
   - Extract fortress/frost color tokens from `index.html` into Tailwind config (assuming `breach-guardian/` uses Tailwind — to confirm in Phase 1) or into `app/globals.css` CSS variables
   - Import DM Serif Display + Outfit via `next/font`
3. Create shared `components/marketing/` with `<Nav>`, `<Hero>`, `<FeatureGrid>`, `<ModeCard>`, `<PricingCard>`, `<Footer>` — mirror the structure of the existing static site
4. Move Google Search Console verification file: `public/google26df59702563d68d.html`
5. Add `public/robots.txt` and `app/sitemap.ts` for SEO
6. Document the migration in `docs/phase-2-migration-notes.md`
7. Archive `securityindeed-website/` → add a README note "Superseded by `breach-guardian/` — see `docs/` for plan history. Kept for history only."

**Files touched in `breach-guardian/`:** ~20 new files under `app/(marketing)/`, `components/marketing/`, `public/`. Possible Tailwind config update.
**Files touched in `securityindeed-website/`:** README note only.
**PRs:** 1 PR ("feat(marketing): port securityindeed.org landing pages into portal app")
**Verification:**
- `npm run dev` in `breach-guardian/` → visit `/`, `/features`, `/pricing`, `/privacy`, `/terms` → all render with new fortress/frost palette
- Lighthouse score ≥ 90 on `/`
- All internal links work
**Rollback:** revert the PR; `securityindeed-website/` repo is unchanged so zero data loss

**Exit criteria:** Visiting `localhost:3000/` in the `breach-guardian/` dev server shows the marketing landing, fully styled, with nav linking to `/login`, `/features`, etc.

**Duration:** 1–2 sessions.

---

### Phase 3 — Auth unification: Google OAuth in NextAuth 🔐

**Goal:** A user can sign in with Google on the web and it auto-links to (or creates) the same user that the mobile app sees via the shared Google identity (email).

**Tasks:**
1. In GCP Console, verify the existing OAuth Web Client has `https://securityindeed.org/api/auth/callback/google` + `http://localhost:3000/api/auth/callback/google` in the authorized redirect URIs. If not, add them.
2. In `breach-guardian/`, update `lib/auth.ts` (or wherever NextAuth config lives — confirm in Phase 1) to add Google provider alongside Credentials provider
3. Enable **account linking by verified email** in NextAuth (`allowDangerousEmailAccountLinking` with safeguards, or use adapter-native linking)
4. Update Prisma schema to ensure `Account` table exists (it's the NextAuth OAuth account linking table); migration if needed
5. On first Google sign-in, create user row with `emailVerified = now()`, default role, linked `Account` row with provider=`google`
6. Update `/register` page to show "Sign in with Google" as primary CTA, "Email + password" as secondary
7. Update `/login` page same
8. Test flows:
   - New Google user signs in → user row created
   - Existing credentials user signs in with Google (same email) → accounts linked
   - Logout and re-login preserves session
9. Document migration path for existing password-only users in `docs/phase-3-auth-migration.md`

**Files touched:** `lib/auth.ts` (or `app/api/auth/[...nextauth]/route.ts`), `prisma/schema.prisma`, new migration, `app/(auth)/login/page.tsx`, `app/(auth)/register/page.tsx`, `components/auth/GoogleSignInButton.tsx`
**PRs:** 1 PR ("feat(auth): add Google OAuth provider + account linking")
**Verification:**
- Manual test all 3 flows above on localhost
- Unit test: account linking by email
- E2E Playwright test: Google sign-in happy path
**Rollback:** revert PR; existing credentials auth unchanged

**Exit criteria:** Signing into the mobile app as `user@gmail.com` and then signing into the web app with Google as the same `user@gmail.com` shows the same user record in the dashboard.

**Duration:** 1 session.

---

### Phase 4 — Vendor-agnostic scrubbing abstraction + Optery (or stub) 🧹

**Goal:** `/data-scrubbing` page works end-to-end. Either with a real Optery account (if available) or with a stub provider that shows "awaiting vendor activation."

**Tasks:**
1. Define `ScrubbingProvider` interface in `lib/scrubbing/provider.ts` (from v3 brief §3)
2. Define `PIIProfile` schema in `lib/scrubbing/types.ts`
3. Add Prisma models: `ScrubbingEnrollment`, `ScrubbingBroker`, `ScrubbingRemovalReceipt`, `ScrubbingEvent`, `PiiProfile` (encrypted JSON column via `pgcrypto`)
4. Migration: `npx prisma migrate dev --name add_scrubbing_models`
5. Implement **`OpteryProvider`** in `lib/scrubbing/providers/optery.ts`:
   - `enrollUser`, `getStatus`, `listBrokers`, `listRemovals`, `triggerScan`, `verifyWebhook`, `parseWebhook`, `deleteUser`
   - Uses `OPTERY_API_KEY` from Secret Manager
6. Implement **`StubProvider`** in `lib/scrubbing/providers/stub.ts` — returns mock data, used when `OPTERY_API_KEY` is missing
7. Factory: `lib/scrubbing/index.ts` picks provider based on env
8. API routes:
   - `POST /api/scrub/submit` — accepts `PIIProfile`, validates with Zod, calls `provider.enrollUser()`
   - `GET /api/scrub/status` — returns current status
   - `GET /api/scrub/brokers` — list brokers where user was found
   - `POST /api/scrub/webhook` — verifies signature, parses normalized event, writes to `ScrubbingEvent` + updates status
   - `DELETE /api/scrub/enrollment` — DSAR delete via provider
9. PII encryption: sensitive `PiiProfile` fields encrypted with `pgcrypto` symmetric encryption using a DEK fetched from Cloud KMS at app startup
10. `/data-scrubbing` page with form (React Hook Form + Zod)
11. `/data-scrubbing/history` page with timeline of events + removal receipts
12. Dashboard card: "Data scrubbing: X brokers removed, Y pending"
13. Document the Optery webhook URL for the user to register in the Optery dashboard

**Files touched:** ~15 new files under `lib/scrubbing/`, `app/(dashboard)/data-scrubbing/`, `app/api/scrub/`, `prisma/schema.prisma`, new migration, dashboard components
**PRs:** 2 PRs
  - PR 4a: "feat(scrubbing): provider interface, Prisma models, API routes"
  - PR 4b: "feat(scrubbing): UI pages and dashboard integration"
**Verification:**
- Unit tests for `OpteryProvider` with mocked HTTP
- Unit tests for `StubProvider`
- Integration test: submit → webhook → status reflects change
- Manual test with stub in dev
- Security review: PII encryption round-trip, webhook signature verification
**Rollback:** revert PRs; `scrubbing_*` tables remain but are empty (or run down migration)

**Exit criteria:** Logged-in user can visit `/data-scrubbing`, fill form, submit, see status change after a simulated webhook call.

**Duration:** 2 sessions.

---

### Phase 5 — Billing: RevenueCat + Stripe on web 💳

**Goal:** `/billing` page works end-to-end. User can subscribe via Stripe Checkout (handled by RevenueCat Web Billing), entitlement is written to Postgres, and access is gated.

**Tasks:**
1. User creates RevenueCat project, shares keys (done in §2.2)
2. Add Prisma model: `Entitlement` (userId, productId, status, periodEnd, platform, rawPayload)
3. Migration
4. Install RevenueCat Web SDK in `breach-guardian/`
5. Configure products in RevenueCat dashboard: `pro_monthly`, `pro_annual`, maybe `family_plan` — matches your existing pricing section
6. Implement `lib/billing/revenuecat.ts` — thin wrapper around SDK
7. API routes:
   - `POST /api/billing/checkout` — creates RevenueCat customer, returns Stripe Checkout URL
   - `POST /api/billing/webhook` — verifies RevenueCat webhook signature, upserts `Entitlement`
   - `GET /api/billing/portal` — returns Stripe customer portal URL for managing existing subs
8. `/billing` page:
   - If no entitlement: show pricing cards with "Upgrade" → Checkout
   - If active entitlement on web: show plan, next billing date, "Manage" → Stripe customer portal
   - If active entitlement on mobile (platform != `web`): show read-only card "Managed in [App Store / Play Store]" + link
9. Middleware helper: `requireEntitlement(userId, feature)` → gates premium features
10. Gate the scrubbing feature behind an entitlement check (per user plan — confirm if free tier should have limited scrubbing)
11. Document webhook endpoints to register in RevenueCat dashboard

**Files touched:** ~12 new files under `lib/billing/`, `app/(dashboard)/billing/`, `app/api/billing/`, `prisma/schema.prisma`, new migration, middleware
**PRs:** 2 PRs
  - PR 5a: "feat(billing): RevenueCat SDK, Entitlement model, API routes"
  - PR 5b: "feat(billing): /billing UI and feature gating"
**Verification:**
- Stripe test mode: checkout flow → webhook → entitlement in DB
- RevenueCat dashboard shows test customer
- Feature gate test: non-subscriber cannot access gated feature; subscriber can
- Webhook signature verification unit test
**Rollback:** revert PRs; no production data touched since Stripe test mode only in dev

**Exit criteria:** Subscribing in Stripe test mode results in entitlement written to DB and user can access gated features. Unsubscribing removes access.

**Duration:** 2 sessions.

---

### Phase 6 — Training module runner (custom MDX) 📚

**Goal:** `/training` catalog + `/training/[moduleId]` runner work with custom MDX content, progress saves to Postgres, dashboard shows progress.

**Tasks:**
1. Decide fate of existing SCORM viewer — **recommendation: keep as parallel path**, add MDX runner alongside. Catalog shows both kinds.
2. Content directory: `content/training/` with subfolders per module (`phishing-basics/`, `password-hygiene/`, `romance-scams/`, `senior-safety/`)
3. Each module has `module.json` (metadata: title, description, durationMinutes, tags) + `index.mdx` (content) + `quiz.json` (questions)
4. Add Prisma model: `TrainingProgress` (userId, moduleId, status, score, attempts, startedAt, completedAt)
5. Migration
6. Build MDX runner component with:
   - Markdown content rendering (via `next-mdx-remote` or `@next/mdx`)
   - Inline quiz component
   - Progress tracker (saves on section view + quiz submit)
7. `/training` page: grid of module cards with progress badges
8. `/training/[moduleId]` page: MDX runner + "Mark complete" button
9. API: `POST /api/training/progress` (save event), `GET /api/training/progress` (list for user)
10. Dashboard card: "Training: 3 of 8 complete"
11. Write the first 2 modules as real content:
    - `phishing-basics/` — "Spotting phishing emails" (existing content if any, or write fresh)
    - `password-hygiene/` — "Good password habits"
12. Document the module-add workflow in `docs/training-authoring.md`

**Files touched:** ~20 new files under `lib/training/`, `content/training/`, `app/(dashboard)/training/`, `app/api/training/`, `components/training/`, `prisma/schema.prisma`, new migration
**PRs:** 2 PRs
  - PR 6a: "feat(training): MDX runner, progress model, API routes"
  - PR 6b: "feat(training): catalog UI + first 2 modules"
**Verification:**
- Read module end-to-end, progress saves, dashboard updates
- Quiz completion updates score
- Resuming a half-read module restores position
**Rollback:** revert PRs; training tables remain (or down migration)

**Exit criteria:** User completes a module, sees checkmark on dashboard, can re-take quizzes.

**Duration:** 2 sessions.

---

### Phase 7 — Hardening + compliance 🛡️

**Goal:** Production-ready security, compliance, and operational posture.

**Tasks:**
1. **DSAR hard-delete flow**: `/api/account/delete` cascades through all user-linked tables + calls `ScrubbingProvider.deleteUser()` + RevenueCat delete customer + writes to `AuditLog`
2. **PII encryption at rest**: confirm `pgcrypto` + Cloud KMS DEK is working for `PiiProfile` and any sensitive `Entitlement` fields
3. **Rate limits**: add rate limiting middleware on `/api/auth/*`, `/api/scrub/submit`, `/api/billing/checkout` (use Upstash Redis or GCP Memorystore — pick lightest)
4. **Security headers**: CSP, HSTS, X-Frame-Options, Referrer-Policy, Permissions-Policy via Next.js middleware or config
5. **Audit log review**: make sure every sensitive action writes to `AuditLog`: login, logout, password change, PII submission, scrub delete, billing change, account delete
6. **Privacy policy update**: add sections for Optery/scrubbing vendor, RevenueCat, Stripe, data retention, DSAR process
7. **Terms of service update**: add subscription terms, billing cycle, refund policy
8. **DPA**: email Optery requesting a Data Processing Agreement; note the requirement in a compliance ticket
9. **MFA UI**: the schema already has `mfaEnabled` + `mfaSecret` — build the enable/disable flow and TOTP verification on login
10. **Account lockout UI**: existing schema has `failedLoginAttempts` + `lockedUntil` — make sure the login page surfaces the lockout state helpfully

**Files touched:** middleware, `/api/account/delete/route.ts`, `lib/security/rateLimit.ts`, `lib/security/headers.ts`, `app/(marketing)/privacy/page.tsx`, `app/(marketing)/terms/page.tsx`, MFA flow files, `lib/audit/`
**PRs:** 3 PRs
  - PR 7a: "feat(security): DSAR delete, rate limits, security headers"
  - PR 7b: "feat(security): MFA enrollment + verification flow"
  - PR 7c: "chore(compliance): privacy policy + terms updates"
**Verification:**
- Manual DSAR test: delete account, verify all tables + Optery + RevenueCat
- Security headers check via `curl -I` against dev server
- Rate limit smoke test
- MFA enrollment + login with TOTP
**Rollback:** revert individual PRs

**Exit criteria:** Security scan (e.g., `npm audit`, OWASP ZAP baseline, Mozilla Observatory A grade), manual penetration checklist passed.

**Duration:** 2 sessions.

---

### Phase 8 — Trust gaps + launch prep ✨

**Goal:** Close the 6 trust gaps identified in earlier research + SEO + analytics. Ready for launch.

**Tasks:**
1. **Named team section** on `/` — photos + titles (you'll supply)
2. **SOC 2 / audit badge** — placeholder section if not yet certified; real badge if you have one. Alternative: "Security practices" page with concrete claims
3. **Partner logo strip** — Optery logo (or whichever vendor), GCP, Next.js, any certs
4. **Live metrics counter** — e.g., "X,XXX brokers removed this month" — pulls from `ScrubbingRemovalReceipt` count with a cache
5. **Contact reality block** — address, phone, legal entity name — goes in footer + `/contact` page
6. **Research citations** — Stanford Web Credibility, NN/g trust studies — in footer or a `/research` page
7. **SEO**:
   - `app/sitemap.ts` auto-generated
   - `public/robots.txt`
   - Per-page metadata via Next.js Metadata API
   - OG tags for social sharing
   - Structured data (JSON-LD) for Organization, Product, SoftwareApplication
8. **Analytics**:
   - Google Analytics 4 (most GCP-native) via `next/script`
   - Privacy-respecting config (anonymize IP, consent mode v2)
9. **Google Search Console migration**:
   - Re-verify ownership of new deploy (the `google26df59702563d68d.html` file ported in Phase 2 should still work)
   - Submit new sitemap

**Files touched:** marketing components, new `app/(marketing)/research/`, `app/(marketing)/contact/`, footer, sitemap, analytics script
**PRs:** 2 PRs
  - PR 8a: "feat(trust): team, partners, metrics, contact, research"
  - PR 8b: "feat(seo): sitemap, OG tags, analytics, structured data"
**Verification:**
- Lighthouse ≥ 95 on `/`
- Mozilla Observatory A
- Google Rich Results test passes
- OG card preview looks right
**Rollback:** revert PRs

**Exit criteria:** `/` scores ≥95 Lighthouse, trust gaps closed, analytics reporting.

**Duration:** 1–2 sessions.

---

### Phase 9 — Deployment to Cloud Run 🚀

**Goal:** `https://securityindeed.org` serves the portal live, over HTTPS, from Cloud Run, backed by Cloud SQL.

**Tasks:**
1. **Dockerfile review**: confirm existing `breach-guardian/Dockerfile` uses Next.js standalone output and runs as non-root
2. **`cloudbuild.yaml`**:
   - Steps: install deps, Prisma generate, build, run tests, build image, push to Artifact Registry, deploy to Cloud Run
   - Trigger on push to `main`
3. **Cloud Run service**:
   - Name: `securityindeed-portal`
   - Min instances: 0 (scale to zero to save money)
   - Max instances: 10
   - CPU: 1, memory: 512 MB → 1 GB depending on Next.js bundle
   - Concurrency: 80
   - Connector: VPC connector for Cloud SQL private IP
   - Service account: `portal-runtime` (from Phase 1)
   - Env vars: all from Secret Manager via `--set-secrets`
4. **Cloud SQL connection**: use Cloud SQL Auth Proxy OR direct private IP via VPC connector
5. **Database migration** step in Cloud Build: `npx prisma migrate deploy`
6. **Domain mapping**: `gcloud run domain-mappings create --service securityindeed-portal --domain securityindeed.org`
7. **DNS**: add `A` / `AAAA` / `CNAME` records at the registrar pointing to Cloud Run — exact records printed by the previous command
8. **Google-managed SSL cert**: automatic via domain mapping
9. **Smoke test** on production:
   - `/` loads
   - `/login` loads
   - `/api/health` returns 200
   - Sign up a test user → verify in Cloud SQL
   - Sign in with Google → verify session
   - Submit scrubbing form → verify webhook flow
   - Billing checkout → verify entitlement write
10. **Write `docs/runbook.md`**: how to deploy, how to rollback, how to read logs, how to restore from DB backup, on-call basics

**Files touched:** `Dockerfile` (minor), `cloudbuild.yaml` (new), `docs/runbook.md` (new)
**PRs:** 1 PR ("feat(deploy): Cloud Build + Cloud Run pipeline")
**Verification:**
- Deploy to staging project first, smoke test
- Deploy to prod
- All smoke tests pass in prod
**Rollback:** `gcloud run services update-traffic securityindeed-portal --to-revisions PREVIOUS_REVISION=100`

**Exit criteria:** `https://securityindeed.org` is live, HTTPS, serving the portal. Test user can go end-to-end: sign up → train → submit scrubbing → subscribe.

**Duration:** 1–2 sessions.

---

### Phase 10 — Mobile integration (deferred, separate project) 📱

**Goal:** The BG mobile app shares auth, billing, training progress, and scrub status with the web portal.

**Tasks (high-level — expanded in a future plan doc):**
1. Mobile `breach-guardian-mobile/` adds an API client pointing to `https://securityindeed.org/api/v1/mobile`
2. API routes under `/api/v1/mobile/*` accept Google OAuth access tokens (same Client ID the mobile already uses) — exchange for a NextAuth session
3. Sync training progress: mobile can pull + push `TrainingProgress` records
4. Sync scrub status: mobile can pull `/api/scrub/status`
5. **Add IAP** via RevenueCat SDK (iOS StoreKit + Android Play Billing)
6. Mobile checks `entitlement.active` via RevenueCat SDK → matches web's gating
7. Add iOS folder (mobile is currently Android-only)
8. Release new mobile version

**Status:** Plan only — not in scope for v4 execution. Will be planned separately once the web portal is live.

---

## 4. Secrets inventory (Phase 1 creates these, other phases populate)

| Name | Created in | Populated in | Notes |
|---|---|---|---|
| `NEXTAUTH_SECRET` | Phase 1 | Phase 1 | Random 64 bytes |
| `NEXTAUTH_URL` | Phase 1 | Phase 9 | `https://securityindeed.org` |
| `DATABASE_URL` | Phase 1 | Phase 1 | Cloud SQL private IP + app user |
| `GOOGLE_CLIENT_ID` | Phase 1 | Phase 3 | From GCP console |
| `GOOGLE_CLIENT_SECRET` | Phase 1 | Phase 3 | From GCP console |
| `REVENUECAT_PUBLIC_SDK_KEY` | Phase 1 | Phase 5 | From RevenueCat dashboard |
| `REVENUECAT_WEBHOOK_SECRET` | Phase 1 | Phase 5 | From RevenueCat dashboard |
| `STRIPE_SECRET_KEY` | Phase 1 | Phase 5 | From Stripe (test mode first, then prod) |
| `STRIPE_WEBHOOK_SECRET` | Phase 1 | Phase 5 | From Stripe CLI or dashboard |
| `OPTERY_API_KEY` | Phase 1 | Phase 4 (when account ready) | Stub until available |
| `KMS_KEY_RESOURCE_NAME` | Phase 1 | Phase 1 | `projects/PROJECT/locations/REGION/keyRings/securityindeed-keyring/cryptoKeys/pii-dek` |
| `SMTP_*` or `SENDGRID_API_KEY` | Phase 1 | Phase 3 | For transactional email |
| `HIBP_API_KEY` | Phase 1 | existing | Already in current `.env.example` |

---

## 5. Dependency graph

```
Phase 0 (prereq)
   ↓
Phase 1 (GCP infra) ─────────┐
   ↓                          │
Phase 2 (repo merge) ────────┤
   ↓                          │
Phase 3 (Google OAuth) ──────┤  These phases unblock Phases 4–6 independently
   ↓                          │
   ├─→ Phase 4 (scrubbing) ──┤
   ├─→ Phase 5 (billing) ────┤
   └─→ Phase 6 (training) ───┘
             ↓
         Phase 7 (hardening)
             ↓
         Phase 8 (trust + SEO)
             ↓
         Phase 9 (deploy)
             ↓
         Phase 10 (mobile) — deferred
```

Phases 4, 5, 6 can run in parallel after Phase 3 is done. If multiple sessions are available, fan out.

---

## 6. Risk register (v4)

| # | Risk | Probability | Impact | Mitigation |
|---|---|---|---|---|
| 1 | Uncommitted mid-refactor in `breach-guardian/` has bugs that block Phase 2 | Medium | High | User handles in §2.1. If bugs surface, revert the refactor and plan it in a separate session. |
| 2 | Optery doesn't grant Business API access in time | High | Medium | Stub provider in Phase 4. Feature ships with "coming soon" until account arrives. |
| 3 | RevenueCat setup friction | Low | Medium | Good docs exist, test in sandbox. |
| 4 | Cloud SQL cost creeps up | Low | Low | Start with `db-f1-micro`, monitor. Scale tier only when needed. |
| 5 | Domain DNS propagation delays launch | Low | Low | Provision Cloud Run service + domain mapping a day ahead of launch. |
| 6 | Existing schema in `breach-guardian/` conflicts with new Phase 4/5/6 models | Low | Medium | Review schema in Phase 1, coordinate migrations. |
| 7 | Next.js version mismatch with new deps | Low | Low | Run `npm install` + full test suite in Phase 0 before any new code. |
| 8 | Google OAuth account-linking by email is dangerous if email is unverified | Medium | High | Require `emailVerified = true` on the existing account before linking; reject otherwise. |
| 9 | PII leak in logs | Medium | Critical | Add logging middleware that strips PII fields from request bodies before writing to Cloud Logging. |
| 10 | Mobile + web diverge without Phase 10 | Medium | Medium | Document API contracts in Phase 4/5/6 so Phase 10 is mostly consumption. |

---

## 7. Acceptance criteria for launch

The portal is considered ready to ship when ALL of:

- [ ] `/` renders, Lighthouse ≥ 90
- [ ] Sign up + Google sign in + credentials sign in all work
- [ ] Existing credentials user can link a Google account
- [ ] `/dashboard` shows training progress, scrub status, entitlement state
- [ ] `/training` lists modules, runner works, progress saves
- [ ] `/data-scrubbing` form submits, webhook loop works end-to-end (or stub shows "coming soon")
- [ ] `/billing` checkout works in Stripe test mode, entitlement writes to Postgres
- [ ] Feature gating by entitlement works
- [ ] `/account/delete` wipes everything across Postgres, Optery, RevenueCat, audit log
- [ ] MFA enrollment + login works
- [ ] Rate limits enforced on sensitive endpoints
- [ ] Security headers: Mozilla Observatory A
- [ ] Privacy policy + terms updated
- [ ] DSAR flow tested end-to-end
- [ ] Deployed to Cloud Run on `securityindeed.org` with Google-managed SSL
- [ ] Smoke tests green in prod
- [ ] Runbook (`docs/runbook.md`) exists
- [ ] Rollback path tested

---

## 8. What's NOT in this plan (explicit deferrals)

- Mobile app changes — **Phase 10, separate plan doc**
- iOS build of the mobile app — **not started, separate track**
- Admin/operator console for internal use — not in scope
- Email templates beyond transactional basics — minimal in v4, expand later
- A/B testing infrastructure — not in scope
- Customer support tooling (helpdesk, ticketing) — not in scope
- Translation / i18n — English only in v4
- SCORM viewer rewrite — keep existing as parallel path
- Internal analytics dashboards beyond GA4 — not in scope
- Any automated data broker removal without a vendor — **not possible**, vendor-dependent

---

## 9. What the user needs to do between now and Phase 1 start

**Check-list:**
1. ☐ Handle the uncommitted mid-refactor in `breach-guardian/` (§2.1) — commit, stash, or discard. Tell me which.
2. ☐ Create RevenueCat account, share keys when ready (§2.2)
3. ☐ Email Optery Business sales for API access OR decide to launch with stub (§2.2)
4. ☐ Confirm GCP project ID + region (§2.3)
5. ☐ Confirm DNS registrar for `securityindeed.org` (§2.4)
6. ☐ Confirm whether to reuse the mobile app's OAuth Web Client or create a new web-only one (§2.3)

Once all 6 are done, say **"start Phase 0"** (or "start Phase 1" if you want to skip the docs-only Phase 0) and I'll begin.

---

## 10. Files touched in this session so far

Only in `C:/Users/Daya/Projects/securityindeed-website/docs/`:
- `portal-architecture-brief.md` (v1 → v2 → v3)
- `REVIEW-ME-questions.md`
- `portal-build-plan-v4.md` (this file)

**Zero files modified in `C:/Users/Daya/Projects/breach-guardian/`** (the target repo). I have only *inspected* it via the investigator agent — no writes, no migrations, no git actions, no `npm install`.

**Zero GCP resources created.** All of Phase 1 is still ahead.

---

## 11. Sign-off

To start executing:

**Reply:** "start Phase 0" (or "start Phase 1" to skip the docs-only step)

To change the plan first:

**Reply with changes needed.** I'll update this file in place and tag as v4.1, v4.2, etc.

---

*v4 saved. Living doc. No code written. Awaiting sign-off.*
