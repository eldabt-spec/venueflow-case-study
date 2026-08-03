# VenueFlow - Technical Due Diligence & AI QA Case Study

**Role:** Technical Advisor & AI QA Lead (March 2026 – present)
**Stack:** Next.js (App Router) · TypeScript · Supabase (Postgres) · Vercel
**Codebase:** Private multi-tenant SaaS venue-management platform (700+ PRs merged to the integration branch). This repo documents my work on it without exposing client code.

---

## What VenueFlow is

A multi-tenant SaaS platform for event-venue businesses: public pricing calculator and inquiry forms, a CRM for venue teams (quotes, bookings, packages, finance), a platform-admin portal, automated email follow-ups, and scheduled cron jobs - all serving multiple venue tenants from one deployment.

I was brought in to assess production readiness and lead QA as the platform prepared to onboard its second tenant. The engagement expanded into security remediation, multi-tenant isolation auditing, migration planning, CI enforcement, and building the AI-assisted QA process the project now runs on. Over the course of it the platform moved from a static readiness assessment through a sustained hardening push to a clean production promotion.

## Headline results

| Area | Outcome |
|---|---|
| **Authentication** | Found and remediated hardcoded JWT fallback secrets in 4 production files - a vulnerability allowing anyone with source access to forge valid sessions for both the CRM and platform-admin portals |
| **Guest-portal cross-tenant leak (shipped to prod)** | Proved a live production vulnerability - a guest token from one venue plus a caller-controlled tenant header returned another venue's quote and payment data on 4 routes - via a two-venue probe kit; TDD fix flipped the probes 200→403/401, 66/66 tests green, promoted to production with a read-only prod safety precheck |
| **Database-layer privilege escalation (fixed in prod)** | Autonomous audit of `SECURITY DEFINER` functions found anonymous callers could escalate to admin via 13 functions missing search-path pins; remediated directly in production via a transactional `ALTER FUNCTION` block after read-only verification |
| **RLS remediation** | Discovered a policy-recursion error was accidentally the only thing preventing anonymous reads of every venue's admin role structure; shipped the recursion fix and exposure closure as one unsplittable migration package, verified by an all-zeros anonymous probe with zero member-visibility regression |
| **Multi-tenant isolation** | Systematic per-route `venue_id` audit + runtime probing against a disposable second tenant; found 6 anonymous cross-tenant data leaks a prior static assessment had missed; closed access-control gaps across 30+ API routes |
| **Schema audit** | Audited 83 multi-tenant tables; found ~7 global UNIQUE constraints (emails, quote numbers) that would break the second tenant's inserts with `23505` errors on day one |
| **Error/XSS hardening at scale** | Landed an error-detail leak sweep with a static lint guard verified at 0 violations across 492 API routes, plus AI-generated-HTML sanitization and email-template escaping verified against hostile payloads on staging |
| **Environment config** | Audited 18 production environment variables against the live Vercel deployment; surfaced 2 confirmed gaps before launch |
| **CI audit** | Full read of every CI workflow found lint gates silently passing on real violations (`continue-on-error`) and no compile/build check in CI at all - deploy platform was load-bearing for correctness; produced a prioritized remediation queue, including a new automated contrast-verification check that caught and fixed a WCAG regression (3.02:1 → 2.15:1) a UI batch had introduced |
| **Incident work** | Root-caused a 6-week cross-tenant email leak (shared Gmail impersonation env var vs. per-venue config) and specified the per-venue fix |
| **Velocity with gates** | ~210 PRs merged to the integration branch in a two-week window - security, isolation, upload hardening, test stabilization, CI enforcement, runbooks - every one through adversarial review, with a 5,700+ test suite kept green throughout |
| **Release management** | Drove the platform through a stakeholder-imposed staging freeze to a clean freeze-lift and full staging→main promotion, sequencing dozens of accumulated fixes behind a promotion-day runbook and per-change provenance checks |
| **Process** | Built the AI QA workflow the project runs on: 37-lens adversarial review, calibrated confidence labels, four-question investigation gates, premise-gated autonomous batch runs |

## Case studies

1. **[JWT fallback secrets - authentication remediation](case-studies/01-jwt-fallback-secrets.md)**
   How a "sensible default" became a session-forgery vulnerability, and the startup-guard pattern that fixed it.

2. **[Multi-tenant isolation audit](case-studies/02-multi-tenant-isolation-audit.md)**
   Why the existing static security assessment couldn't catch cross-tenant leaks, the census + runtime-probe method that did, and the leak pattern taxonomy that came out of it.

3. **[Environment variable audit](case-studies/03-environment-audit.md)**
   Cross-referencing documented config against the live deployment, and rebuilding `.env.example` and the README so the docs match reality.

4. **[Production-verified findings: proving exploits before fixing them](case-studies/04-production-verified-findings.md)**
   Three findings that reached "proven with a live probe" before any fix was designed - a guest-portal cross-tenant leak, a database-function privilege escalation, and an RLS policy whose *bug* was its only safety property - and how each was verified, fixed, and re-verified in production.

5. **[Turning CI from theater into a gate](case-studies/05-ci-enforcement.md)**
   A CI pipeline that reported green while real violations accumulated behind `continue-on-error`, and the incremental campaign that made type-checking, lint families, and accessibility contrast genuinely blocking without halting a fast-moving team.

6. **[The Outage That Passed Every Test](case-studies/silent-outage-incident.md)** — root-causing a silent production failure invisible to every CI gate

## Process documentation

- **[AI QA workflow](process/ai-qa-workflow.md)** - how I run QA with Claude Code as the execution layer and myself as the verification layer: evidence labels (Verified / Inferred / Hypothesized), the four-question investigation gate, adversarial review, and autonomous-run architecture with STOP-AND-REPORT gates.

## What this demonstrates

- Security auditing of a production SaaS system (authn, authz, tenant isolation, config)
- Root-cause discipline: no fix designed until the cause is *Verified*, the surface area is measured, and production state is confirmed
- AI-assisted engineering with real quality gates - treating LLM output as untrusted until grounded in code
- Clear technical writing: every finding here was originally delivered to stakeholders as an actionable document

---

*All examples are described at the pattern level. No proprietary client code appears in this repo. Identifying details of tenant businesses have been generalized.*
