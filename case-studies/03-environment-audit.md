# Case Study 3 - Environment Variable Audit & Documentation Rebuild

**Outcome:** 18 production variables cross-referenced against the live deployment; 2 confirmed configuration gaps surfaced before launch; `.env.example` and README rebuilt from scratch to match reality

## The problem

The project's `.env.example` was incomplete and partially stale: it referenced variables the code no longer read, omitted variables the code required, and gave no guidance on which values were sensitive, how to generate them, or which features broke without them. The README's security notes described behavior the code didn't have.

For a solo-maintained production SaaS about to onboard developers and a second tenant, that's not a cosmetic problem. Stale config documentation is how the JWT-fallback vulnerability (Case Study 1) stays invisible: nobody can audit config against a spec that doesn't exist.

## The audit

Three-way cross-reference, treating each source as a claim to verify rather than a fact:

1. **What the code reads** - grep for every `process.env.*` reference in the codebase
2. **What the docs claim** - the existing `.env.example` and README
3. **What production actually has** - the live Vercel environment variable dashboard

Discrepancies in any direction are findings: code reading an undocumented variable, docs listing a dead variable, production missing a required one. The audit surfaced **2 confirmed gaps in the live production configuration** - caught before launch rather than as runtime failures.

It also found dead code: verification scripts checking for environment variables that no longer existed anywhere in the codebase, silently passing (or silently skipping) their checks. Those were replaced with real checks against live endpoints, including one that treats an expected `401` as the *pass* condition - proving the API and database are reachable without requiring credentials in the test environment.

## The rebuild

The new `.env.example` documents all 18 variables, organized into 6 setup stages (database → authentication → app config → business features → integrations → operational), with per-variable:

- Which files consume it
- How to generate it where relevant (`openssl rand -hex 32` for secrets)
- Explicit severity warnings ("treat this like a root database password" on the service-role key)
- What breaks without it

The README gained an architecture overview, a comparison table of the two auth systems (who logs in where, which cookie, which secret), a step-by-step local setup path, and a security section that describes the code's actual current behavior.

## What this illustrates

Documentation is a QA artifact. The rule applied throughout: **every named thing in the docs - variable, route, command, file path - is drafted from reading the actual code, not from an assumed model of it.** Where verification contradicted the assumed model, the docs changed to match reality, not the other way around.
