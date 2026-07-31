# Case Study 4 — Production-Verified Findings: Proving Exploits Before Fixing Them

The strongest guarantee in security work is not "I reviewed the code and it looks fine" — it's "I made the exploit happen, then made it stop happening, then confirmed it can't happen in production." Three findings from the multi-tenant hardening push cleared that bar. Each one was *proven with a live probe before any fix was designed*, and each was re-verified after the fix landed.

---

## Finding 1 — Guest-portal cross-tenant leak (promoted to production)

**Class:** Broken access control / IDOR
**Reach:** Any guest quote or payment link

Guest quote portals authenticate with a token embedded in the link — no login required. The token was stored as columns on the users table, and several portal routes trusted a caller-supplied venue header (`X-Venue-Slug`) to decide which venue's data to return. The result: a valid guest token from venue A, combined with a header naming venue B, returned **venue B's** quote, payment, and detail data. A fourth route accepted expired tokens as still valid.

**Proving it first.** Rather than patch on suspicion, I built a two-venue probe kit: two disposable staging venues with fixed UUIDs and tokens, seed SQL applied one statement at a time, and curl probes with the exact pre-fix and post-fix responses written down in advance. The pre-fix run confirmed four leaked `200` responses — cross-tenant data on three routes, expiry bypass on the fourth.

**The fix, and a falsified assumption.** The original plan was to route all four through an existing auth helper. The first test-driven step *falsified that plan* — a `git grep` of sibling routes showed the real convention was an inline `venue_id` guard, used by five other routes. Following the existing pattern rather than the assumed-better one, the fix became an inline guard returning `403` on venue mismatch, plus an expiry check returning `401`. The probe kit flipped from `200/200/200/200` to `403/403/403/401`. All 66 tests green; two fixture files needed `venue_id` added, which the correct guard surfaced.

**Production promotion with a safety net.** Because this leak was live in production, promotion carried a read-only prod precheck first: zero cross-venue token mismatches and zero affected expired tokens in the live data, confirmed before the fix shipped. Promoted via merge-commit to main.

---

## Finding 2 — Anonymous privilege escalation via `SECURITY DEFINER` functions (fixed in production)

**Class:** Privilege escalation
**Reach:** Anonymous callers

Postgres functions marked `SECURITY DEFINER` run with the definer's privileges, not the caller's. An autonomous audit found 13 such functions with no pinned `search_path` — a pattern that lets a caller who can influence the schema resolution path escalate toward admin-level operations. This was reachable by anonymous callers.

The remediation ran directly against production: a series of read-only verification queries to confirm the exact function set and current state, then a single transactional `ALTER FUNCTION` block pinning the search path on all of them at once — atomic, so a partial failure couldn't leave the system half-fixed. Prod verification via an anonymous curl confirmed the escalation path now returns the expected permission error.

---

## Finding 3 — The RLS policy whose bug was its only safety property

**Class:** Row-level-security misconfiguration
**Reach:** Anonymous reads of every venue's admin structure

This one is the most instructive. The `team_members` and `roles` tables had RLS policies that threw a `42P17` infinite-recursion error. That looked like a pure bug to fix. Investigation revealed something more dangerous: **the recursion error was the only thing preventing anonymous callers from reading every venue's complete admin role structure.** Naively "fixing" the recursion in isolation would have opened the exposure it was accidentally masking.

So the fix could not be split. The recursion correction and the exposure closure had to ship as **one unsplittable migration package** — fix the policy logic and close the anonymous-read hole in the same atomic change. Verification after applying it to staging: an anonymous probe returned all zeros (no rows leaked), the `42P17` error was gone, and a non-regression check confirmed legitimate access was intact — team members, roles, and role-permission rows all still visible to authorized callers.

---

## The through-line

All three share a shape the four-question investigation gate is built to enforce: **root cause Verified (not hypothesized), a working-version proof in the actual environment, full surface area measured, and a production-state precheck before shipping.** Two of the three also carried a lesson beyond the fix itself — the guest-portal fix falsified its own original design in the first test step, and the RLS fix revealed that a bug and a safety property can be the same line of code. Neither would have surfaced from static review alone. You have to run the probe.
