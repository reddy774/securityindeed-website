# securityindeed-website

> ⚠️ **ARCHIVED — SUPERSEDED BY `breach-guardian/` as of 2026-04-13**
>
> This repository contained the static HTML marketing site for
> **Breach Guardian / securityindeed.org**. As of **Phase 2** of the
> portal build, all marketing content has been ported into the Next.js
> application at `C:/Users/Daya/Projects/breach-guardian/` under
> `app/(marketing)/`.
>
> **All future marketing work happens in `breach-guardian/`.** Do not
> edit files in this repository — changes here will not be deployed.

---

## What lives here now (historical record only)

- `index.html`, `privacy.html`, `terms.html` — the original static pages
  that were ported to Next.js App Router routes in Phase 2
- `google26df59702563d68d.html` — Google Search Console verification
  file (also copied to `breach-guardian/public/` so it stays live on
  `securityindeed.org`)
- `docs/` — the Phase 0 / Phase 1 planning package:
  - [`HANDOVER.md`](./docs/HANDOVER.md) — canonical 1,100-line handover
    document with 10-phase build plan, locked technical decisions,
    research findings, risk register, and execution log
  - [`portal-build-plan-v4.md`](./docs/portal-build-plan-v4.md) —
    detailed phase-by-phase build plan (same content as HANDOVER §8,
    slightly different framing)
  - [`portal-architecture-brief.md`](./docs/portal-architecture-brief.md)
    — v1 → v2 → v3 thinking trail with rationale for pivots
  - [`REVIEW-ME-questions.md`](./docs/REVIEW-ME-questions.md) — user's
    answered decision questions
- `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `.cursorrules`,
  `.github/copilot-instructions.md` — LLM adapter files from the
  canonical-adapter pattern bootstrap

## Where work happens now

**Production codebase:** `C:/Users/Daya/Projects/breach-guardian/`

**Marketing routes** in that repo (ported from this one):
- `app/(marketing)/page.tsx` → `securityindeed.org/`
- `app/(marketing)/features/page.tsx` → `/features`
- `app/(marketing)/how-it-works/page.tsx` → `/how-it-works`
- `app/(marketing)/pricing/page.tsx` → `/pricing`
- `app/(marketing)/privacy/page.tsx` → `/privacy`
- `app/(marketing)/terms/page.tsx` → `/terms`

**Deployment target:** GCP Cloud Run in project
`new-project-security-emails` (display name: "SecurityIndeed Portal"),
region `us-central1`. See
[`breach-guardian/docs/gcp-infra.md`](../breach-guardian/docs/gcp-infra.md)
for the full infrastructure runbook.

## Why this repo is retained

This repository is kept online (rather than deleted) for three reasons:

1. **Historical record** — the planning documents in `docs/` capture
   the reasoning behind architectural decisions that the breach-guardian
   repo references via relative paths
2. **Search Console recovery** — the `google26df59702563d68d.html`
   verification file here serves as a backup if we ever need to
   re-verify domain ownership from this repo
3. **Git history** — the old static HTML revisions remain browsable
   for reference

If you need to update anything marketing-related, open
`breach-guardian/` instead. Changes made in this repo will not be
deployed and will diverge from production.
