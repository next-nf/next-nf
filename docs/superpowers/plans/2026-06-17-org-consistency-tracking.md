# Org Consistency Tracking — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the program-management substrate for the org-wide consistency effort — a consistent GitHub issue taxonomy + org Projects board across the five repos, a convention-only refactor of the `nf-conventions` skill, and the seeded epic + per-repo child issues.

**Architecture:** Operational cycle, not code. GitHub side: enable issues on `smf`, create a shared label set on all five repos, create an org Projects v2 board, then seed epic issues (in `next-nf`) and per-repo child issues, linking children into epic task-lists and onto the board. Repo side: add `reference/issue-tracking.md`, trim per-repo cleanups out of seven reference files (replacing them with the timeless rule + a link to the tracking epic), and update `SKILL.md` and `CLAUDE.md`. Subagents generate grounded issue bodies; a deterministic pass files them after a user review gate.

**Tech Stack:** `gh` CLI (issues, labels, project v2, api), GitHub Projects v2, Claude Code plugin/skill markdown, `claude plugin validate`.

## Global Constraints

- **Repos (5):** `next-nf/chf`, `next-nf/pcf`, `next-nf/smf`, `next-nf/udr`, `next-nf/next-nf`.
- **Routing:** per-repo work (feature/bug/needs-planning) → that repo; conceptual/overarching → `next-nf`.
- **Label set (verbatim):** type — `type:epic`, `type:feature`, `type:bug`, `type:chore`, `type:spike`; area — `area:db`, `area:otel`, `area:otp29`, `area:records`, `area:naming`, `area:diameter`, `area:data`, `area:docs`, `area:ci`, `area:demo`, `area:integ-tests`, `area:ui`, `area:oss`, `area:release`, `area:deploy`; marker — `convention`.
- **HARD REVIEW GATE:** no `gh issue create` runs until the user approves the generated catalog (Task 4 output). Public issues are effectively irreversible.
- **`smf` LICENSE is reviewed, never auto-normalized** (erGW fork, copyleft — `type:spike` only).
- **Graceful degrade:** any `gh` step that needs scopes/admin the token lacks (enable-issues, project create) is surfaced as a `! gh ...` command for the user to run; the rest of the plan proceeds.
- After any skill edit: `claude plugin validate` must pass for the marketplace and both plugins.
- Branch for this cycle: `org-consistency-tracking` (already created). Frequent commits.

---

### Task 1: Bootstrap — enable smf issues + shared label set

**Files:** none (GitHub state only).

**Interfaces:**
- Produces: the label set exists on all five repos; `smf` has issues enabled. Later tasks attach these labels.

- [ ] **Step 1: Verify gh auth + scopes**

Run: `gh auth status`
Expected: logged in to github.com. Note scopes; `project` / `admin:org` may be absent (Task 3 handles that).

- [ ] **Step 2: Enable issues on smf (graceful degrade)**

Run: `gh api -X PATCH repos/next-nf/smf -F has_issues=true`
Expected: JSON with `"has_issues": true`. If it 403s (not admin), STOP this step and surface to the user:
`! gh api -X PATCH repos/next-nf/smf -F has_issues=true` and continue.

- [ ] **Step 3: Create labels on all five repos**

For each repo in the five, create each label idempotently. Use stable colors (type=`5319e7`, area=`0e8a16`, convention=`fbca04`). Example for one repo:

```bash
REPO=next-nf/chf
for L in "type:epic" "type:feature" "type:bug" "type:chore" "type:spike"; do
  gh label create "$L" -R "$REPO" -c 5319e7 --force; done
for L in area:db area:otel area:otp29 area:records area:naming area:diameter area:data area:docs area:ci area:demo area:integ-tests area:ui area:oss area:release area:deploy; do
  gh label create "$L" -R "$REPO" -c 0e8a16 --force; done
gh label create "convention" -R "$REPO" -c fbca04 --force
```

Repeat with `REPO` = `next-nf/pcf`, `next-nf/smf`, `next-nf/udr`, `next-nf/next-nf`.

- [ ] **Step 4: Verify labels**

Run: `for r in chf pcf smf udr next-nf; do echo "== $r =="; gh label list -R next-nf/$r | grep -c -E 'type:|area:|convention'; done`
Expected: each repo reports `21` (5 type + 15 area + 1 convention).

- [ ] **Step 5: Commit (marker only — no repo files changed)**

Nothing to commit here; record progress in the task tracker and proceed.

---

### Task 2: Create the org Projects v2 board (graceful degrade)

**Files:** none (GitHub state only).

**Interfaces:**
- Produces: project number (`PROJECT_NUMBER`) + fields. Task 5 adds issues to it. If scopes are missing, this task is deferred to the user and Task 5 skips the add-to-board step.

- [ ] **Step 1: Check project scope**

Run: `gh auth status | grep -i 'project'`
Expected: `project` (and ideally `read:org`) present. If absent, surface to user:
`! gh auth refresh -s project,read:org` then re-run. If the user declines, mark the board deferred and continue — issue seeding does not depend on it.

- [ ] **Step 2: Create the project**

Run: `gh project create --owner next-nf --title "next-nf consistency"`
Expected: prints the project URL + number. Capture the number as `PROJECT_NUMBER`.

- [ ] **Step 3: Add single-select fields**

```bash
gh project field-create PROJECT_NUMBER --owner next-nf --name "Workstream" --data-type SINGLE_SELECT \
  --single-select-options "db,otel,otp29,records,naming,diameter,data,docs,ci,demo,integ-tests,ui,oss,release,deploy"
gh project field-create PROJECT_NUMBER --owner next-nf --name "Component" --data-type SINGLE_SELECT \
  --single-select-options "chf,pcf,smf,udr,next-nf"
gh project field-create PROJECT_NUMBER --owner next-nf --name "Effort" --data-type SINGLE_SELECT \
  --single-select-options "Low,Med,High"
```
(The built-in `Status` field already has Todo/In progress/Done; add `Blocked` via the UI or `gh project field` edit if needed.)

- [ ] **Step 4: Verify**

Run: `gh project view PROJECT_NUMBER --owner next-nf`
Expected: shows the project with the three custom fields.

---

### Task 3: Generate the issue catalog (subagents, no filing)

**Files:**
- Create: `docs/superpowers/issue-catalog.md` (reviewable artifact, committed).

**Interfaces:**
- Consumes: the epic catalog in the spec (`docs/superpowers/specs/2026-06-17-org-consistency-tracking-design.md` §4).
- Produces: a committed catalog listing, per epic and per child, `{repo, title, labels[], body}` — bodies grounded in real file paths. Task 4 reads this.

- [ ] **Step 1: Dispatch per-repo grounding subagents**

Dispatch four Explore/general subagents in parallel (one per NF repo: chf, pcf, smf, udr). Each is given the spec's epic catalog rows touching its repo and tasked to return, per child issue: a one-paragraph **scope**, the concrete **file paths / modules** involved (verified against the repo), **acceptance criteria**, and the correct **labels**. They MUST NOT create issues. They return structured text only.

- [ ] **Step 2: Compose the catalog file**

Assemble returns into `docs/superpowers/issue-catalog.md`, grouped by epic. For each epic: title, `next-nf` as repo, labels (`type:epic` + `area:*` + `convention` where applicable), body (purpose + checklist placeholder for children). For each child: target repo, title, labels, grounded body. Include the conceptual `next-nf` epics (E15–E17) and `E-deploy`. Mark `smf` LICENSE child as `type:spike` (review, not normalize).

- [ ] **Step 3: Commit the catalog**

```bash
git add docs/superpowers/issue-catalog.md
git commit -m "docs: issue catalog for org consistency program (pre-filing review)"
```

- [ ] **Step 4: HARD REVIEW GATE**

Present `docs/superpowers/issue-catalog.md` to the user. Do NOT proceed to Task 4 until the user approves the catalog. Apply any edits they request and re-commit.

---

### Task 4: Seed epic issues

**Files:** none (creates GitHub issues). Updates `docs/superpowers/issue-catalog.md` with assigned numbers.

**Interfaces:**
- Consumes: approved `issue-catalog.md`.
- Produces: epic issue numbers (e.g. `E1 → #N`), recorded back into the catalog. Task 5 links children to these.

- [ ] **Step 1: Create each epic in next-nf**

For each epic, create the issue with its labels. Example:

```bash
gh issue create -R next-nf/next-nf --title "[epic] OTP-29 + diameter build toolchain" \
  --label type:epic --label area:otp29 --label convention \
  --body-file <(printf '%s' "$EPIC_BODY")
```
Capture the returned issue number for each epic.

- [ ] **Step 2: Record epic numbers**

Edit `docs/superpowers/issue-catalog.md`, annotating each epic with its `#N` (needed for child task-lists and for the skill links in Task 6).

- [ ] **Step 3: Verify**

Run: `gh issue list -R next-nf/next-nf --label type:epic --limit 50`
Expected: every epic from the catalog is listed.

- [ ] **Step 4: Commit catalog update**

```bash
git add docs/superpowers/issue-catalog.md
git commit -m "docs: record seeded epic issue numbers"
```

---

### Task 5: Seed child issues, link task-lists, populate board

**Files:** none (creates GitHub issues; updates GitHub project).

**Interfaces:**
- Consumes: epic numbers from Task 4; `PROJECT_NUMBER` from Task 2 (if created).
- Produces: all per-repo child issues, each linked to its epic; epic task-lists populated; all issues on the board.

- [ ] **Step 1: Create child issues per repo**

For each child, create in its target repo with labels. Example:

```bash
gh issue create -R next-nf/chf --title "OTP-29 upgrade + switch to diameter_make" \
  --label type:feature --label area:otp29 --label convention \
  --body-file <(printf '%s' "$CHILD_BODY")
```
Capture each child's `repo#number`.

- [ ] **Step 2: Populate epic task-lists**

For each epic, edit its body to append a task-list referencing its children (cross-repo refs work: `- [ ] next-nf/chf#12`). Use `gh issue edit next-nf/next-nf#N --body-file ...`.

- [ ] **Step 3: Add issues to the board (skip if board deferred)**

```bash
for U in $ALL_ISSUE_URLS; do gh project item-add PROJECT_NUMBER --owner next-nf --url "$U"; done
```

- [ ] **Step 4: Verify**

Run: `for r in chf pcf smf udr next-nf; do echo "== $r =="; gh issue list -R next-nf/$r --limit 100 | wc -l; done`
Expected: counts match the catalog (chf/pcf/smf/udr have their children; next-nf has epics + conceptual). Spot-check one epic shows a populated task-list via `gh issue view`.

---

### Task 6: Skill refactor — convention-only + issue-filing guidance

**Files:**
- Create: `plugins/nf-conventions/skills/nf-architecture/reference/issue-tracking.md`
- Modify: `plugins/nf-conventions/skills/nf-architecture/SKILL.md` (index the new reference + set-policy bullet)
- Modify: `reference/erlang-otp.md`, `reference/diameter.md`, `reference/database.md`, `reference/observability.md`, `reference/architecture.md`, `reference/data-handling.md`, `reference/conventions.md`, `reference/deployment-philosophy.md`
- Modify: `CLAUDE.md` (skill philosophy)

**Interfaces:**
- Consumes: epic issue URLs from Task 4 (for the per-file "Tracking:" links).

- [ ] **Step 1: Write `reference/issue-tracking.md`**

Content: when to file an issue (missing feature, bug, work needing its own plan); routing (per-repo vs `next-nf`); the verbatim label set (from Global Constraints); a short body template (Context · Scope · Source convention link · Acceptance); and the rule that convention-derived per-repo work lives in issues, not in the reference files. Link to the board.

- [ ] **Step 2: Index it in SKILL.md**

Add a reference-index row for `issue-tracking.md` and a set-policy bullet ("file per-repo conventions work as issues; this skill states the rule, the epic tracks the work"). Mirror the existing row format.

- [ ] **Step 3: Trim the seven reference files**

For each file+section below, remove the per-repo state/Action table or open-item checklist and replace it with the preserved rule plus a single line: `> **Tracking:** <epic URL> (per-repo children).` Keep all rules, rationale, syntax, and `udr`-reference examples.

- `erlang-otp.md`: §1 OTP-29 table + diameter-toolchain coupling → E1; §3 record-migration counts → E2.
- `diameter.md`: §1b RFC6733-common table, §2 `{request_errors,answer}` table, §3 chf ENUM-redefinition, §6 toolchain table → E7 (toolchain line also → E1).
- `database.md`: §7.1 per-NF status + §7.2 sequencing → E3.
- `observability.md`: §2 per-repo OTEL migration table → E4.
- `architecture.md`: §2 SBI/OAM/Web naming table → E5.
- `data-handling.md`: §1 JSON-key table → E6.
- `conventions.md`: §10 CI note → E9; open-items list → split (naming→E5, clustering→E3, license→E13, supervision-shape→E16).
- `deployment-philosophy.md`: open-items list → E-deploy.

- [ ] **Step 4: Update CLAUDE.md**

Change the `nf-conventions` description: it currently says the skill states "current per-repo state and migration action." Replace with: the skill states the target convention and links to the tracking epic; per-repo state and migration actions live in GitHub issues.

- [ ] **Step 5: Validate**

```bash
claude plugin validate .claude-plugin/marketplace.json
claude plugin validate plugins/nf-docs
claude plugin validate plugins/nf-conventions
```
Expected: all pass.

- [ ] **Step 6: Commit**

```bash
git add plugins/ CLAUDE.md
git commit -m "nf-conventions: convention-only refactor + issue-tracking guidance

Extract per-repo cleanups into the seeded epics (linked), add
reference/issue-tracking.md, update SKILL.md index and CLAUDE.md philosophy."
```

---

### Task 7: Finalize — PR

**Files:** none (git/PR).

- [ ] **Step 1: Push the branch**

Run: `git push -u origin org-consistency-tracking`

- [ ] **Step 2: Open the PR**

```bash
gh pr create -R next-nf/next-nf --base main --head org-consistency-tracking \
  --title "Org consistency program: issue tracking substrate + skill refactor" \
  --body-file <(...)  # summary: labels, board, seeded epics+children, skill convention-only refactor
```

- [ ] **Step 3: Verify**

Run: `gh pr view --json url,state`
Expected: open PR. Report the URL + an issue/board summary to the user.

---

## Self-Review

**Spec coverage:** §1 labels → Task 1; §2 board → Task 2; §3 skill refactor (issue-tracking.md, seven-file trim, SKILL.md, CLAUDE.md) → Task 6; §4 issue catalog (epics + children, E15–E17, E-deploy) → Tasks 3–5; §5 execution order (bootstrap → bodies → review → seed → skill links) → Task ordering; §7 risks (irreversibility gate, gh scopes, philosophy reversal, epic-number bootstrap resolved by seeding before linking) → Global Constraints + graceful-degrade steps. No gaps.

**Placeholder scan:** `$EPIC_BODY`/`$CHILD_BODY`/`$ALL_ISSUE_URLS`/`PROJECT_NUMBER` are runtime values produced by earlier steps (Task 3 catalog, Task 2 output), not unfilled placeholders. No "TBD/handle edge cases" steps.

**Ordering consistency:** epic numbers created in Task 4 are consumed by Task 5 (task-lists) and Task 6 (skill links) — seeding precedes linking, resolving the bootstrap risk. Labels (Task 1) precede issue creation (Tasks 4–5). Board (Task 2) precedes add-to-board (Task 5 Step 3), which is skipped if the board was deferred.
