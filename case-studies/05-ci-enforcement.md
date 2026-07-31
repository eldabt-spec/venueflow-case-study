# Case Study 5 — Turning CI From Theater Into a Gate

**Context:** A CI pipeline that passed reliably — and caught almost nothing
**Outcome:** Type-check, multiple lint families, and accessibility-contrast checks made genuinely blocking; the deploy platform demoted from load-bearing-for-correctness back to a deploy platform

## The finding

A full read of every CI workflow file turned up a pipeline that looked healthy and wasn't. Two problems compounded:

1. **Gates that couldn't fail.** Several lint jobs ran with `continue-on-error: true`. They executed, they reported, and their result was discarded — so real and growing violations sat behind a green check. A gate that cannot fail is not a gate; it is a status light wired to "on."

2. **Missing gates entirely.** There was no type-check, no lint, and no build step in CI. The only thing standing between a type error or a broken build and production was the deploy platform rejecting it *after merge*. Correctness was being enforced downstream of the merge button, which means it wasn't being enforced at all where it mattered.

The meta-point mirrors the security audit (Case Study 2): a passing signal is a *claim*, and an unverified claim from your own tooling is still unverified. "CI is green" earned no trust until I could show it went red on a real defect.

## Why you can't just flip everything blocking

The honest constraint: the codebase carried a known backlog — roughly 210 pre-existing type errors concentrated in test files, and thousands of accumulated lint violations. Flipping every gate to blocking on day one would have wedged the entire team behind a wall of pre-existing debt and taught everyone to bypass the gates, which is worse than no gates.

So the campaign was **ratchet, not switch**: freeze the debt at its current level, block anything that makes it worse, and burn it down in tracked batches underneath.

## The campaign

Each of these landed as its own reviewed change:

- **Baseline-ratcheted type checking.** A blocking TypeScript gate that diffs against a recorded ~210-error baseline: new errors fail the build, the known backlog doesn't. The baseline is a committed number that only ever ratchets down, so the debt can shrink but never silently grow.

- **Lint families made blocking one at a time.** The error-detail-leak lint (which enforces that anonymous routes don't leak internal error text) went blocking only after its violation count was driven to zero across ~492 routes — so the gate turned on against a clean floor. The dark-mode lint went blocking together with an allowlist-growth guard, so the accessibility backlog can only shrink. A settings-gates lint was taught to recognize helper-based gates (it had been blind to a valid pattern and would have flagged correct code) *before* it was allowed to block.

- **Accessibility contrast as a permanent check.** After a UI batch silently worsened text-on-fill contrast from 3.02:1 to 2.15:1 — below the 3.0 non-text floor — the fix shipped with an automated contrast-verification check wired into CI. The specific regression class can no longer reach production unseen.

- **Pipeline hygiene.** Sharded the unit-test run three ways for speed, added a `node_modules` cache across jobs, and put a concurrency lock and timeout on the migration workflow so two runs can't race.

## Proven, not assumed

The gates were not declared working because they were configured. Each blocking gate was demonstrated to go **red on a real regression** — a deliberately failing change confirming the pipeline rejects what it's supposed to reject — before it was trusted. A gate you haven't watched fail is a gate you're hoping works.

## What this illustrates

Two transferable ideas. First, **the ratchet pattern**: when you inherit debt, you don't have to choose between "ignore it" and "block everyone." Freeze it, gate the delta, and burn it down underneath — the gate protects forward progress from day one while the backlog shrinks on its own schedule. Second, **tooling output is a claim like any other.** The most dangerous state isn't a red pipeline; it's a green one that isn't actually checking anything, because it converts the absence of verification into the appearance of it.
