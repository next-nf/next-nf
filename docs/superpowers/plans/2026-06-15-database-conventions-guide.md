# Database Conventions Guide Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Author `reference/database.md` in the `nf-conventions` skill — the org-wide data/persistence convention guide — and wire it into the skill's index and sibling reference files.

**Architecture:** This is a **documentation** deliverable, not code. The authoritative content source is the committed design spec at `docs/superpowers/specs/2026-06-15-unified-db-architecture-design.md` (same branch). Each task **translates** one spec section into the established `nf-conventions` reference-file voice — *Rule → current per-repo state → migration action*, with `> [!NOTE]/[!IMPORTANT]/[!WARNING]` callouts and cross-links — matching the style of the existing `erlang-otp.md`, `diameter.md`, and `observability.md`. The per-repo "current state" facts come from the four-repo data-layer catalog already established in this session and encoded in the spec.

**Tech Stack:** Markdown (GitHub-flavored). Verification is `claude plugin validate plugins/nf-conventions` plus grep-based cross-link and grounding checks. No build/test toolchain.

**Out of scope (own future cycle):** the UDR→PCF→CHF→SMF *code* migrations. This plan produces only the convention guide + its integration.

**Conventions for every task below:**
- Work in the worktree root `/home/nathanf/next-nf/next-nf/.claude/worktrees/db-conventions` on branch `db-conventions`. Use plain paths; cwd resolves to this worktree.
- Target file: `plugins/nf-conventions/skills/nf-architecture/reference/database.md` (referred to as `database.md`).
- "Adapt spec §X" means: rewrite that section's content in reference-file voice (rule-first, imperative `shall/should`, tables for per-repo state, callouts for hazards) — **do not** paste spec prose verbatim; match the sibling files' register. The spec is the source of truth for the facts; do not invent new ones.
- Commit after each task with the shown message.

---

### Task 1: Skeleton + §1 "The policy" (principles)

**Files:**
- Create: `plugins/nf-conventions/skills/nf-architecture/reference/database.md`

- [ ] **Step 1: Create the file with the grounded-as-of header and the policy section.**

Header note in the style of the other reference files (compare `observability.md:3`):
```markdown
# Database & persistence: the unified data layer

> Grounded as of 2026-06-15. Set policy: every NF persists through one `<comp>_db` behaviour — a **document store** (binary-keyed maps in named collections) with **single-document CAS**, over **Mnesia (ram/disc) + MongoDB** backends. `udr` is the reference (closest to target); `chf`/`pcf`/`smf` migrate toward it.
```
Then a `## 1. The policy` section listing principles **P1–P9** from spec §3 as a numbered/`shall`-worded list (one line each: one contract; documents not records; one aggregate = one document; coordination via `syn`, correctness via CAS; two backends/three tiers; backend-native indexes; one API for everything + dirty-read fast path; backend-agnostic conformance; versioned documents). Cross-link `[`erlang-otp.md`](erlang-otp.md)` §2.4 on the native-record/Mnesia rule (P2) and `[`data-handling.md`](data-handling.md)` (binary keys).

- [ ] **Step 2: Verify the plugin still validates.**

Run: `claude plugin validate plugins/nf-conventions`
Expected: `✔ Validation passed` (the "no version specified" warning is expected and fine).

- [ ] **Step 3: Commit.**
```bash
git add plugins/nf-conventions/skills/nf-architecture/reference/database.md
git commit -m "Add database.md skeleton and policy section

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: §2 "The `<comp>_db` contract"

**Files:**
- Modify: `database.md` (append `## 2. The <comp>_db contract`)

- [ ] **Step 1: Write the contract section.** Adapt spec §4. Must contain, concretely:
  - The **types** block (`collection/key/doc/version/selector/index/coll_opts`) verbatim from spec §4.1 (these are the contract — reproduce exactly).
  - A **backend-callbacks table** (the 11 callbacks: `child_spec`, `ensure_collection`, `get`, `put`, `cas_put`, `delete`, `take`, `find`, `find_by`, `fold`, `count`) with the one-line semantics from spec §4.2.
  - The **facade helpers** `update(Coll, Key, Fun)` (the functional CAS — bounded retry, `{aborted,R}` never retried) and `create(Coll, Key, Doc)`, plus the `persistent_term` backend dispatch note.
  - A `> [!IMPORTANT]` capturing the four semantics from spec §4.4: single-doc atomicity only; `version` invisible to domain (vs `schema_version` doc field); reads not serializable except via `update/3`; indexes backend-maintained. Note the declarative `#{set,inc}` form is deliberately not in the contract (deferred optimization).

- [ ] **Step 2: Verify.** Run: `claude plugin validate plugins/nf-conventions` — Expected: `✔ Validation passed`.

- [ ] **Step 3: Commit.**
```bash
git add plugins/nf-conventions/skills/nf-architecture/reference/database.md
git commit -m "database.md: add the <comp>_db contract section

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 3: §3 "Backends & deployment tiers"

**Files:**
- Modify: `database.md` (append `## 3. Backends & deployment tiers`)

- [ ] **Step 1: Write the backends section.** Adapt spec §5. Must contain:
  - **`<comp>_db_mnesia`**: the generic envelope with **promoted index columns** (`attributes = [key | DeclaredIndexFields] ++ [version, doc]`, Mnesia index per promoted column, `find_by`→`index_read`); op→dirty-vs-transaction mapping; the `storage => ram_copies | disc_copies` durability/clustering knob; bootstrap (`create_schema`→`ensure_collection`→`wait_for_tables`). Cross-link `[`erlang-otp.md`](erlang-otp.md)` §2.4 (why no domain records as rows).
  - **`<comp>_db_mongo`**: `_id`=key, top-level `version`, `createIndex`; op→Mongo-call mapping incl. `cas_put`→`updateOne({_id,version},$set,$inc)` with `n=0` disambiguation; stateless nodes + replica-set replication; `majority` write concern for charging; `{<comp>_db_mongo, load}` load-not-start.
  - The **four-tier deployment table** (spec §5.3): Dev/unit/CI-fast · CI-cluster · Light-prod · Serious-prod.
  - A `> [!NOTE]` on the typing trade-off (generic map envelope forfeits Dialyzer on stored shapes; mitigated by accessor modules, §4).

- [ ] **Step 2: Verify.** Run: `claude plugin validate plugins/nf-conventions` — Expected: `✔ Validation passed`.

- [ ] **Step 3: Commit.**
```bash
git add plugins/nf-conventions/skills/nf-architecture/reference/database.md
git commit -m "database.md: add backends and deployment tiers

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 4: §4 "Data model & aggregate discipline"

**Files:**
- Modify: `database.md` (append `## 4. Data model & aggregate discipline`)

- [ ] **Step 1: Write the data-model section.** Adapt spec §6. Must contain:
  - Document conventions: collection = lowercase singular atom; key = natural 3GPP id as binary; reserved `<<"schema_version">>`/`<<"created_at">>`/`<<"updated_at">>`. Cross-link `[`data-handling.md`](data-handling.md)`.
  - The **aggregate-per-document** rule (P3).
  - The **CHF money worked example** — reproduce the balance-aggregate map literal from spec §6.3 (reservations folded into the balance doc; `available` derived; `reserve`/`commit`/`refund` as one `update/3` with invariant `Fun`; session doc descriptive).
  - The **accessor-module-per-aggregate** pattern (owns shape, `from_doc/1`/`to_doc/1`, invariant `Fun`s, `schema_version` + upgrade-on-read).
  - **Schema evolution** (P9): additive free; reshaping via upgrade-on-read; bulk runner deferred.

- [ ] **Step 2: Verify.** Run: `claude plugin validate plugins/nf-conventions` — Expected: `✔ Validation passed`.

- [ ] **Step 3: Commit.**
```bash
git add plugins/nf-conventions/skills/nf-architecture/reference/database.md
git commit -m "database.md: add data model and aggregate discipline

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 5: §5 "Concurrency & coordination"

**Files:**
- Modify: `database.md` (append `## 5. Concurrency & coordination`)

- [ ] **Step 1: Write the concurrency section.** Adapt spec §7.1. Must contain: the two-layer model (`syn` per-key lock = coordination via `<comp>_cluster:with_entity/3`; `version` CAS = correctness); the rules (wrap the *procedure* not a lone write; lock the aggregate key; saga locks the primary entity; tight scope, no global locks; `update/3` retries `version_conflict` to N=100, `{abort,R}` never retried); uniform across tiers; the **SMF node-local carve-out** (hot per-node indices skip Layer 1; cluster-wide context-ownership is an SMF concern, not a DB convention).

- [ ] **Step 2: Verify.** Run: `claude plugin validate plugins/nf-conventions` — Expected: `✔ Validation passed`.

- [ ] **Step 3: Commit.**
```bash
git add plugins/nf-conventions/skills/nf-architecture/reference/database.md
git commit -m "database.md: add concurrency and coordination

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 6: §6 "Consistency & error handling"

**Files:**
- Modify: `database.md` (append `## 6. Consistency & error handling`)

- [ ] **Step 1: Write the consistency section.** Adapt spec §7.2–7.5. Must contain:
  - The **error taxonomy table** (`ok` · `not_found` · `version_conflict` · `{aborted,Reason}` · `max_retries` · `exists` · `Reason`) + the backend-normalization rule (errors bubble up, never swallowed; map to Diameter result code / SBI 5xx — cross-link `[`diameter.md`](diameter.md)`, `[`data-handling.md`](data-handling.md)`).
  - **Idempotency**: delete idempotent, put upsert, create→exists; money ops carry an idempotency token checked inside the `Fun`.
  - The **CDR outbox** as the one sanctioned saga + the general cross-aggregate template (stamp intent atomically → drain with idempotent write → clear), in a `> [!NOTE]`.
  - Read consistency / durability concern per call site; the **readiness gate** (`shall not` accept signaling until `<comp>_db` ready).

- [ ] **Step 2: Verify.** Run: `claude plugin validate plugins/nf-conventions` — Expected: `✔ Validation passed`.

- [ ] **Step 3: Commit.**
```bash
git add plugins/nf-conventions/skills/nf-architecture/reference/database.md
git commit -m "database.md: add consistency and error handling

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 7: §7 "Current state & migration" (per-NF + sequencing)

**Files:**
- Modify: `database.md` (append `## 7. Current state & migration`)

- [ ] **Step 1: Write the per-NF migration section.** Adapt spec §8.1–8.2. Must contain:
  - A **per-NF table** (`udr`/`pcf`/`chf`/`smf`) with columns *Today* (grounded from the catalog) and *Target shift* and *Effort*, matching spec §8.1.
  - The **UDR→PCF→CHF→SMF sequencing** with the one-line rationale, and the note that each NF is its own spec→plan→implementation cycle.
  - Cross-link `[`conventions.md`](conventions.md)` §6 (data layer) and `[`architecture.md`](architecture.md)` §3 (pluggable-DB pattern).

- [ ] **Step 2: Verify.** Run: `claude plugin validate plugins/nf-conventions` — Expected: `✔ Validation passed`.

- [ ] **Step 3: Commit.**
```bash
git add plugins/nf-conventions/skills/nf-architecture/reference/database.md
git commit -m "database.md: add per-NF current state and migration sequencing

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 8: §8 "Testing" + §9 "Checklist"

**Files:**
- Modify: `database.md` (append `## 8. Testing` and `## 9. Checklist`)

- [ ] **Step 1: Write the testing + checklist sections.** Adapt spec §8.3. Testing must contain: the **backend-agnostic conformance suite** as the executable spec (must pass on mnesia-ram, mnesia-disc, mongo) with the scenario list; per-NF domain tests on mnesia-ram + Mongo-testcontainers parity subset; cluster/failover (mnesia-ram multi-node CT + Mongo replica-set testcontainers); charging concurrency/property tests + outbox exactly-once under crash injection; the SMF perf gate; shared CT helper (kill copy-pasted `setup_mnesia/0`).
  Then a `## 9. Checklist` (markdown `- [ ]` items) for "instrumenting a component's data layer", mirroring the checklist style of `diameter.md` §7 / `observability.md` §6: persists via `<comp>_db` only; documents-not-records; aggregate-per-document; declared indexes; `syn`+CAS; error taxonomy normalized; conformance suite passes on all backends; readiness gate; `schema_version` on every doc.

- [ ] **Step 2: Verify.** Run: `claude plugin validate plugins/nf-conventions` — Expected: `✔ Validation passed`.

- [ ] **Step 3: Commit.**
```bash
git add plugins/nf-conventions/skills/nf-architecture/reference/database.md
git commit -m "database.md: add testing strategy and checklist

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 9: Integrate into SKILL.md index and sibling cross-links

**Files:**
- Modify: `plugins/nf-conventions/skills/nf-architecture/SKILL.md` (reference table ~lines 34-43; set-policies list ~lines 13-19)
- Modify: `plugins/nf-conventions/skills/nf-architecture/reference/conventions.md` (§6 Data layer, ~lines 56-58)
- Modify: `plugins/nf-conventions/skills/nf-architecture/reference/erlang-otp.md` (§2.4 — add a forward cross-link to `database.md` for the Mnesia/native-record rule)

- [ ] **Step 1: Add a reference-index row in `SKILL.md`.** In the "Reference (load on demand)" table, add a row:
```markdown
| The **unified data layer** (`<comp>_db` document/CAS contract, Mnesia+Mongo backends, aggregate discipline, migration) | [`reference/database.md`](reference/database.md) |
```
And add a set-policy bullet in the policies list:
```markdown
- **Unified data layer:** one `<comp>_db` document/CAS behaviour over Mnesia (ram/disc) + MongoDB; aggregate-per-document; `syn`+CAS. (`udr` reference.)
```

- [ ] **Step 2: Point `conventions.md` §6 at the new guide.** Replace the §6 body so it states the data layer follows `database.md` (keep the existing pluggable-`<comp>_db` mention; add: "See [`database.md`](database.md) for the unified document/CAS contract, backends, and migration.").

- [ ] **Step 3: Add the forward cross-link in `erlang-otp.md` §2.4.** On the Mnesia-incompatibility bullet, append: "The unified data layer avoids this entirely by storing documents in a generic envelope, not domain records — see [`database.md`](database.md)."

- [ ] **Step 4: Verify validation + cross-link integrity.**

Run: `claude plugin validate plugins/nf-conventions`
Expected: `✔ Validation passed`.

Run (every linked file must exist — expect no "MISSING" lines):
```bash
cd plugins/nf-conventions/skills/nf-architecture/reference && \
for f in $(grep -oE '\]\([a-z-]+\.md\)' database.md | sed -E 's/\]\(|\)//g' | sort -u); do [ -f "$f" ] && echo "ok $f" || echo "MISSING $f"; done
```
Expected: all `ok`, no `MISSING`.

- [ ] **Step 5: Commit.**
```bash
git add plugins/nf-conventions/skills/nf-architecture/SKILL.md plugins/nf-conventions/skills/nf-architecture/reference/conventions.md plugins/nf-conventions/skills/nf-architecture/reference/erlang-otp.md
git commit -m "nf-conventions: index database.md and cross-link from conventions/erlang-otp

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 10: Final self-consistency pass + grounding check

**Files:**
- Modify (if issues found): `database.md`

- [ ] **Step 1: Grounding check against the spec.** Re-read `database.md` start-to-finish next to `docs/superpowers/specs/2026-06-15-unified-db-architecture-design.md`. Confirm: 11 backend callbacks named identically throughout; the four deployment tiers match; the CHF balance-aggregate fields match; principle numbering P1–P9 consistent. Fix any drift inline.

- [ ] **Step 2: Placeholder scan.** Run:
```bash
grep -nE 'TODO|TBD|FIXME|\bXXX\b|placeholder' plugins/nf-conventions/skills/nf-architecture/reference/database.md || echo "clean"
```
Expected: `clean` (any genuine "deferred"/"out of scope" wording is fine; bare TODO/TBD is not).

- [ ] **Step 3: Final validate.** Run: `claude plugin validate plugins/nf-conventions` — Expected: `✔ Validation passed`.

- [ ] **Step 4: Commit any fixes (skip if none).**
```bash
git add plugins/nf-conventions/skills/nf-architecture/reference/database.md
git commit -m "database.md: consistency and grounding fixes

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Self-review (filled in by plan author)

**Spec coverage:** spec §3→Task 1; §4→Task 2; §5→Task 3; §6→Task 4; §7.1→Task 5; §7.2–7.5→Task 6; §8.1–8.2→Task 7; §8.3→Task 8; §9 (deferred items) surface as "deferred" notes within Tasks 2/4/8; §10 (next step) realized as Task 9 integration + this plan's out-of-scope note on UDR. No spec section unmapped.

**Placeholders:** none — content source is the committed spec; Task 10 scans for stray TODO/TBD.

**Type/name consistency:** the 11 backend callbacks and 2 facade helpers are named once (Task 2) and referenced by the same names in Tasks 3/5/6/8; Task 10 step 1 explicitly re-checks naming drift across the file.
