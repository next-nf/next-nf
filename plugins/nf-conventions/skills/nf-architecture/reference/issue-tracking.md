# Issue tracking: where per-repo work lives

## Purpose

Per-repo conventions work and other tracked work live in **GitHub issues**, not in these reference files. The reference files state the rule and link to the tracking epic. This keeps the skill content timeless (conventions only) while giving implementers a single place to find current status and coordinate across repos.

## When to file an issue

File an issue when there is:

- A **missing feature** — something the convention calls for that a repo does not yet implement.
- A **bug or violation** — existing code that breaks a convention.
- A **work item with its own plan** — multi-step or cross-repo work that needs coordination.

Do not create issues for one-line fixes you make inline; do create them for anything that will span multiple PRs or require a spec.

## Routing

- **Per-repo work** (implementing a convention in one component) → file in that repo: `next-nf/<comp>` (e.g. `next-nf/chf`, `next-nf/udr`).
- **Conceptual or overarching work** (a new convention, an org-wide decision, or a multi-repo epic) → file in `next-nf/next-nf`.

Org epics live in `next-nf/next-nf`; per-repo child issues link back to the epic.

## Labels

Apply labels consistently so the org program board can filter and track:

**Type** (pick one):
- `type:epic` — org-level epic that groups per-repo work
- `type:feature` — new capability
- `type:bug` — violation or regression
- `type:chore` — refactor, rename, cleanup
- `type:spike` — research or investigation

**Area** (pick one or more):
- `area:db` — data layer / `<comp>_db`
- `area:otel` — observability / OpenTelemetry
- `area:otp29` — OTP-29 upgrade / Diameter toolchain
- `area:records` — native records migration
- `area:naming` — app / module naming conventions
- `area:diameter` — Diameter stack discipline
- `area:data` — JSON keys / data-handling conventions
- `area:docs` — operator documentation
- `area:ci` — CI configuration and consistency
- `area:demo` — demo / integration scenarios
- `area:integ-tests` — integration test suites
- `area:ui` — minimum web UI
- `area:oss` — OSS hygiene, licensing, release
- `area:release` — release process
- `area:deploy` — deployment configuration
- `area:ops` — operations / runbooks

**Marker**:
- `convention` — work governed by the `nf-conventions` skill (this plugin); apply to any issue that implements or tracks a convention in this skill.

## Issue body template

Use this structure for new issues:

```
## Context
<Why this work matters; what triggered the issue.>

## Scope
<Which repo(s) are affected; what is in and out of scope.>

## Source convention
<Link to the relevant section in the nf-conventions skill — e.g. [erlang-otp.md §1](…).>

## Acceptance
<What done looks like — checklist or criteria.>
```

## Org program board

The org program board exists at the `next-nf` org level and aggregates epics and child issues across all component repos. Epics in `next-nf/next-nf` are the top-level entries; per-repo children link to them via the issue body or GitHub's tracked-by relationship.
