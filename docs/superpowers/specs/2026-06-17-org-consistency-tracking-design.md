# Design: org-wide consistency program — issue tracking + skill refactor

**Date:** 2026-06-17
**Repo:** `next-nf/next-nf` (profile + plugin marketplace; the org coordination repo)
**Status:** design for review

## Context

The four next-nf network functions — `chf`, `pcf`, `smf`, `udr` — are now all on GitHub but at
very different levels of maturity. `udr` is the de-facto reference (docs, CI, demos, the modern
data layer); the others trail it to varying degrees, and `smf` (the erGW fork) is the structural
outlier. The `nf-conventions` skill already describes the *target* architecture, but it currently
also carries a large amount of **per-repo cleanup/migration guidance** baked into its reference
files — which is the wrong place for tracked, mutable work.

This program brings every component "at least as good as UDR" and makes the gap **trackable**. The
deliverable of *this* cycle is the **program-management substrate**, not the engineering itself:

1. A consistent GitHub issue taxonomy + routing across all five repos.
2. An org-level GitHub Projects v2 board over those issues.
3. A refactor of the `nf-conventions` skill: keep the timeless conventions, **extract** all per-repo
   cleanups into issues, and **add** guidance on filing issues.
4. The seeded set of **epic + per-repo child issues** for every workstream.

The actual NF engineering (docs, DB unification, OTP-29, OTEL, UI, …) is explicitly **out of scope
here** — each becomes its own later brainstorm → spec → plan → implement cycle, tracked by the
issues seeded now.

## Current state (inventory, 2026-06-17)

| | README | LICENSE | docs/ | CI | UI app | issues | umbrella |
|---|---|---|---|---|---|---|---|
| **udr** | ✅ 183L | ✅ 661L | `design/ manual/` | ✅ 4 wf (incl demos) | ❌ no `udr_web` | enabled (1 open: #25) | richest (11 apps) |
| **chf** | ❌ none | ✅ 661L | `clustering configuration` | ✅ `ci.yml` | ✅ `chf_web` | enabled (empty) | full (8 apps) |
| **pcf** | ❌ none | ✅ 661L | `METRICS.md` | ❌ none | ✅ `pcf_web` (+`pcf_http`) | enabled (empty) | full (9 apps) |
| **smf** | ✅ 557L | ⚠️ 340L (different) | ❌ none | ❌ none | **disabled** | erGW fork, 5 apps |

## Decisions (locked)

- **Routing:** per-repo work (feature/bug/needs-planning) → that repo; conceptual/overarching → `next-nf`.
- **Skill trim:** skill states **only** the timeless convention; **all** per-repo state + Action columns
  + open-item checklists move to issues; skill links to the tracking epic. `CLAUDE.md`'s description
  of the skill philosophy is updated to match.
- **Granularity:** each workstream = one `next-nf` **epic** + one **child issue per affected repo**,
  all filed in this cycle.
- **Coordination:** linked tracking issues **plus** an org-level **GitHub Projects v2** board.

## 1. Label taxonomy (applied to all five repos)

- **type:** `type:epic`, `type:feature`, `type:bug`, `type:chore`, `type:spike` (needs-planning).
- **area:** `area:db`, `area:otel`, `area:otp29`, `area:records`, `area:naming`, `area:diameter`,
  `area:data` (JSON/encoding), `area:docs`, `area:ci`, `area:demo`, `area:integ-tests`, `area:ui`,
  `area:oss`, `area:release`, `area:deploy`.
- **marker:** `convention` (issue extracted from / governed by the `nf-conventions` skill).

Labels are created idempotently per repo (`gh label create … || gh label edit …`) with stable
colors so the board reads consistently.

## 2. GitHub Projects v2 board

- One org-level project `next-nf consistency` over all five repos' issues.
- Fields: **Status** (Todo / In progress / Blocked / Done), **Workstream** (single-select mirroring
  `area:`), **Component** (chf/pcf/smf/udr/next-nf), **Effort** (Low/Med/High).
- Epics and children are added to the board at creation time.

> **Prerequisite / risk:** Projects v2 + org-level boards require a `gh` token with `project` (and
> `read:org`) scope and org-admin rights. If the available token lacks these, the board step is
> deferred to a manual `gh auth refresh -s project,read:org` (run by the user via `!`) — the issue
> seeding does **not** depend on the board and proceeds regardless.

## 3. Skill refactor (`plugins/nf-conventions/skills/nf-architecture/`)

**Add — `reference/issue-tracking.md`** (new, indexed from `SKILL.md`): when to file an issue
(missing feature, bug, work needing its own plan), routing (repo vs. `next-nf`), the label set above,
and a short issue body template (context · scope · source convention link · acceptance). States that
convention-derived work is tracked in issues, not in the reference files.

**Extract** per-repo "current state + Action" tables and open-item checklists from these files,
replacing each with the timeless rule + a link to its tracking epic:

| File | What is extracted → epic |
|---|---|
| `erlang-otp.md` | §1 OTP-29 upgrade table + diameter-toolchain coupling → **E1**; §3 record migration surface → **E2** |
| `diameter.md` | §1b RFC6733-common verify, §2 `{request_errors,answer}` table, §3 chf ENUM redefinition, §6 toolchain → **E7** (+E1) |
| `database.md` | §7.1 per-NF status + §7.2 sequencing → **E3** |
| `observability.md` | §2 per-repo OTEL migration table → **E4** |
| `architecture.md` | §2 SBI/OAM/Web naming table → **E5** |
| `data-handling.md` | §1 JSON-key table → **E6** |
| `conventions.md` | §10 CI note → **E9**; open-items list → routed (naming→E5, clustering→E3, license→E13, supervision-shape→E16) |
| `deployment-philosophy.md` | open-items list → **E14/E-deploy** |

What **stays**: the rules, rationale, syntax, the "why", and `udr`-as-reference examples (these are
timeless). Only the mutable per-repo state and TODOs leave.

**`CLAUDE.md`:** update the `nf-conventions` description — it currently says the skill states "current
per-repo state and migration action"; change to "states the target convention and links to the
tracking epic; per-repo state and migration actions live in GitHub issues."

## 4. Issue catalog (the review centerpiece)

Each **Exx** is a `type:epic` issue in `next-nf` with a task-list linking its children. Children are
`type:feature`/`spike` issues in the named repo. `udr` is usually the reference (no child, or a
"done/reference" note). Source = the skill section the body is grounded in.

### Conventions-application epics (extracted from skill)

- **E1 — OTP-29 + diameter build toolchain** `area:otp29 convention`
  - `chf`: 28→29; `rebar3_diameter_compiler` → `diameter_make`. · `pcf`: 28→29; toolchain switch.
  - `smf`: 26.2→29 (largest jump); toolchain switch. · `udr`: reference (done).
- **E2 — Native-records migration** `area:records convention`
  - `smf`: ~67 hand-written classic records (`smf_core`, move `.hrl` defs into owning modules).
  - `pcf`: 2 Cowboy `#state{}` → native. · `chf`: 2 module-local `#state{}` → native (the 4
    `chf_db.hrl` Mnesia rows **stay classic**). · `udr`: reference (done).
- **E3 — Data-layer unification** `area:db convention` — sequence **udr → pcf → chf → smf**
  - `udr`: widen 6→11-callback contract, ETS→`udr_db_mnesia`, `schema_version` + accessors (becomes
    reference). · `pcf`: records→documents, generic contract, add `pcf_db_mongo`+`pcf_cluster`.
  - `chf`: records→documents **+ aggregate redesign + CDR outbox**, add `chf_db_mongo`+`chf_cluster`
    (safety-critical). · `smf`: registries→collections, route `smf_aaa`/`smf_core` raw-ETS through
    `smf_db`, benchmark-gated.
- **E4 — OTEL rollout** `area:otel convention`
  - `smf`: port custom Prometheus metrics (GTP-C/U, PFCP, Diameter collectors) → OTEL instruments;
    wire collectors into app startup. · `pcf`: swap exporter → OTEL (+Prometheus exporter); add
    Gx CCR/CCA instruments. · `chf`: same; add Gy/Ro, Rf, balance-op instruments. · `udr`: reference.
- **E5 — App/module naming** `area:naming convention`
  - `chf`: `chf_api`→`chf_sbi` **and** `chf_provision`→`chf_api` (modules, `.app.src`, boot order;
    largest rename). · `smf`: split top-level `smf` into `smf_sbi`/`smf_api`/`smf_web`; `_handler`→`_h`.
  - `udr`: `udr_provision`→`udr_api`. · `pcf`: `pcf_provision`→`pcf_api` (+ decide `pcf_http` lib name).
- **E6 — JSON key handling** `area:data convention`
  - `pcf`: binary keys in `pcf_sbi_json`/`pcf_provision_json`/`pcf_http_json`. · `chf`:
    `chf_provision_subscriber_h` `binary_to_existing_atom` → binary. · `smf`: replace `jsx` with OTP
    `json`; keep payloads binary-keyed.
- **E7 — Diameter discipline** `area:diameter convention`
  - `chf`: remove hand-redefined ENUMs/result codes (`chf_diameter_avp.erl`, `_rf.erl`, `_gy.erl`);
    add `{request_errors, answer}` to Gy/Ro + Rf; verify RFC6733 as common App-Id 0. · `pcf`: add
    `{request_errors, answer}` to Gx; verify RFC6733 common. · `smf`: add `{request_errors, answer}`
    to Gx/Ro/Rf/NASREQ; verify RFC6733 common.

### Consistency / program epics

- **E8 — Documentation parity** `area:docs` (UDR is the bar; apply the `nf-docs` skill)
  - `chf`: write README; bring `docs/` to UDR `manual/`+`design/` level. · `pcf`: write README;
    expand `docs/` (only `METRICS.md` today). · `smf`: create `docs/`; align README. · `udr`:
    reference — link/close existing docs issue **#25**.
- **E9 — CI consistency** `area:ci`
  - `pcf`: add CI. · `smf`: add CI. · `chf`: align to OTP-29 / shared template. · `next-nf` child:
    factor a shared CI template (conventions.md §10 TODO).
- **E10 — Global integration demo** `area:demo` (next-nf-led; depends on E15)
  - `next-nf`: compose-based all-NF demo + inter-NF wiring. · per-repo: demo profiles/hooks as needed
    (`udr` demo workflows are the reference).
- **E11 — Global integration tests** `area:integ-tests` (next-nf-led; depends on E15)
  - `next-nf`: cross-NF integration harness. · per-repo: expose test fixtures/seams as needed.
- **E12 — Minimum UI per component** `area:ui` (health state · config display · provisioning API)
  - `udr`: new `udr_web` (none today). · `smf`: web surface (none). · `chf`: flesh out `chf_web`.
    · `pcf`: flesh out `pcf_web`.
- **E13 — OSS hygiene & tagging** `area:oss`
  - all repos: `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`; first semver tag.
  - `smf`: **LICENSE review** (340-line, erGW fork — copyleft/legal review, **not** auto-normalize) — `type:spike`.
- **E14 — Release / versioning / compatibility scheme** `area:release` (next-nf conceptual + apply)
  - `next-nf`: define scheme (semver per repo + inter-NF compatibility matrix). · per-repo: apply once defined.
- **E-deploy — Deployment guidance** `area:deploy` (next-nf conceptual; from deployment-philosophy open items)
  - `next-nf`: per-component deployment guidance (bare-metal vs container); document/link the bare-metal
    procedure; capture DPDK/SR-IOV/CPU-pinning requirements when concrete (`type:spike`).

### Conceptual epics (next-nf only)

- **E15 — Inter-NF interface contracts** `area:integ-tests` — which reference points are wired; prereq for E10/E11.
- **E16 — Consistency scorecard / definition-of-done** — measurable "as good as UDR"; also the canonical
  `<comp>_core` supervision-tree shape (conventions.md open item).
- **E17 — next-nf forks upkeep** — lifecycle of org-owned forks (OTEL libs, `semantic-conventions`).

### Program bootstrap (executed this cycle — not a tracked epic)

- Enable issues on `smf`. · Create the label set on all five repos. · Create the Projects v2 board.

## 5. Execution approach

1. **Bootstrap:** enable smf issues; create labels (all repos); create the board (or defer per the
   scope-risk note).
2. **Skill refactor:** add `issue-tracking.md`; trim the seven reference files to convention-only with
   epic links; update `SKILL.md` index and `CLAUDE.md`. (Epic issue numbers are filled in after step 3,
   or links are added in a follow-up edit once numbers exist.)
3. **Issue bodies:** subagents do a per-repo grounding pass (real file paths from the extraction) and
   return structured `{repo, title, labels, body, epic}` objects — no filing yet.
4. **Seed:** deterministic pass creates epics first (to get numbers), then children (linking to their
   epic), then populates epic task-lists and adds everything to the board.
5. **Validate:** `claude plugin validate` on the marketplace + both plugins after the skill edits.

The complete `{epic → children}` list (titles + bodies) is produced for **user review before any
`gh issue create` runs** — public issues are effectively irreversible.

## 6. Out of scope

All actual NF engineering — each epic's implementation is a separate later cycle. This cycle creates
the tracking substrate and the (reviewed) issues only.

## 7. Risks

- **Irreversibility:** ~50 public issues. Mitigated by the pre-filing review gate and creating in a
  single reviewed batch.
- **`gh` scopes:** board + cross-org label creation may need `project,read:org` scopes / admin; board
  step degrades gracefully to manual.
- **Skill philosophy reversal:** trimming per-repo state contradicts current `CLAUDE.md`; that file is
  updated in the same change so the skill and its own guidance stay consistent.
- **Epic-number bootstrapping:** skill links need epic issue numbers that don't exist until filing;
  resolved by linking after seeding (step 4) or a follow-up edit.
