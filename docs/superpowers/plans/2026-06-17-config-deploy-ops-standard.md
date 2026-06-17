# Config / Deployment / Operations Standard — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the E18 design into a new `nf-conventions` reference (`reference/config-deploy-ops.md`) that codifies the unified config/deployment/operations standard, retire the placeholder `deployment-philosophy.md`, and populate the four existing E18 conformance child issues with concrete per-NF checklists.

**Architecture:** Documentation cycle. Author one reference file from the spec, fold the timeless philosophy out of `deployment-philosophy.md` into it and remove the old file, wire `SKILL.md`'s index, then update GitHub issues. No NF repo changes (conformance is later per-repo work). Validation is `claude plugin validate`.

**Tech Stack:** Claude Code skill markdown, `claude plugin validate`, `gh` CLI.

## Global Constraints

- Repo: `/home/nathanf/next-nf/next-nf`, branch `e18-config-deploy-ops-standard` (already created; stay on it).
- Source of truth: `docs/superpowers/specs/2026-06-17-config-deploy-ops-standard-design.md`.
- New reference style matches existing `reference/*.md`: rule-first, concise, cross-linked, blockquote callouts where useful.
- **Locked decisions (verbatim):** release name `<comp>`; dedicated uninstrumented admin listener serves metrics + health + readiness; structured JSON logging to stdout; StatefulSet + headless Service for clustered NFs; `sys.config`+`vm.args` remain runtime truth, **no env-var override layer**.
- **Canonical ports (verbatim):** Diameter 3868 · SBI 8080 · Provisioning/OAM API 8090 · Web UI 8081 · Admin (metrics+health+ready) 9464.
- This cycle DEFINES the standard; it does NOT modify chf/pcf/smf/udr.
- After skill edits, all three `claude plugin validate` must pass.
- E18 issues already exist: epic `next-nf#21`, standard child `next-nf#22`, conformance children `next-nf/udr#29`, `next-nf/chf#8`, `next-nf/pcf#8`, `next-nf/smf#13`.

---

### Task 1: Author `reference/config-deploy-ops.md` + retire `deployment-philosophy.md`

**Files:**
- Create: `plugins/nf-conventions/skills/nf-architecture/reference/config-deploy-ops.md`
- Remove: `plugins/nf-conventions/skills/nf-architecture/reference/deployment-philosophy.md` (fold its timeless philosophy into the new file first)
- Modify: `plugins/nf-conventions/skills/nf-architecture/SKILL.md` (index row + set-policy bullet)
- Modify (only if they link to `deployment-philosophy.md`): other `reference/*.md` cross-links → repoint to `config-deploy-ops.md`

**Interfaces:**
- Consumes: the spec (all sections) + the Global Constraints above.
- Produces: the canonical standard reference that the conformance issues (Task 2) point to.

- [ ] **Step 1: Read the spec and the current deployment-philosophy.md**

Read `docs/superpowers/specs/2026-06-17-config-deploy-ops-standard-design.md` in full, and
`plugins/nf-conventions/skills/nf-architecture/reference/deployment-philosophy.md` (to preserve its
timeless philosophy prose — position, bare-metal vs container rationale, Erlang/OTP fit).

- [ ] **Step 2: Write `reference/config-deploy-ops.md`**

Translate the spec into a rule-first reference (not a copy of the design doc — state conventions). It MUST contain these sections, with the verbatim locked decisions and port table:

1. **Intro + scope** — one paragraph: all NFs behave the same for config/deployment/operations; udr is the reference; `sys.config`+`vm.args` are the runtime source of truth.
2. **Config** — atom-keyed per-app `sys.config`; override by mounted file / ConfigMap (`container.sys.config` binds `0.0.0.0`); `application:get_env/2,3` with defaults; **no env-var override**. A `> [!NOTE]` for the reserved future path: schema-validated YAML → escript generator → `sys.config`, runnable as a k8s initContainer (specified, not yet built).
3. **Deployment** — relx release named `<comp>`; two-stage Alpine `Containerfile` (`erlang:29-alpine` build → pinned `alpine` runtime, unprivileged, `foreground` entrypoint); the canonical ports table (verbatim from Global Constraints); k8s template set (StatefulSet + headless Service + ConfigMap + probes + optional initContainer).
4. **Operations — k8s lifecycle contract** — liveness `GET /health` (9464); readiness `GET /ready` (9464, true only when cluster-joined + DB-connected + listeners-up); graceful SIGTERM (not-ready → drain → leave cluster → stop within grace period; preStop `bin/<comp> drain`); automatic rejoin + partition-heal on restart (mechanism = E3); rolling updates = pod-by-pod stop/start, no hot upgrade, native records aid value survival.
5. **Logging** — structured JSON to stdout via OTP `logger` formatter; standard level/handlers; OTEL stays primary for traces/metrics.
6. **Tooling (CLI-first)** — `bin/<comp>` subcommands `cluster join|leave|status`, `health`, `drain`; non-interactive, exit codes; optional read-only ops API folded into E12.
7. **Runbooks** — udr's `docs/manual/operations` format + stable IDs; standard set incl. rolling-update + cluster-recovery; authored via `nf-docs`.
8. **Cross-links + tracking** — link E3 (clustering), E12 (ops API), E1 (container base), E4 (metrics). End with: `> **Tracking:** conformance is tracked in [next-nf#21](https://github.com/next-nf/next-nf/issues/21) (epic + per-repo children).`

Fold `deployment-philosophy.md`'s timeless philosophy prose into §1/§3 where it fits (do not lose the bare-metal-vs-container rationale).

- [ ] **Step 3: Remove `deployment-philosophy.md`**

```bash
git rm plugins/nf-conventions/skills/nf-architecture/reference/deployment-philosophy.md
```

- [ ] **Step 4: Update `SKILL.md`**

Replace the `deployment-philosophy.md` index row with a `config-deploy-ops.md` row (same format), and
add/adjust a set-policy bullet noting the unified config/deploy/ops standard. Grep the other reference
files for `deployment-philosophy` links and repoint any to `config-deploy-ops.md`.

```bash
grep -rn "deployment-philosophy" plugins/nf-conventions/skills/nf-architecture/
```
Fix every hit (repoint to `config-deploy-ops.md`).

- [ ] **Step 5: Validate**

```bash
claude plugin validate .claude-plugin/marketplace.json
claude plugin validate plugins/nf-docs
claude plugin validate plugins/nf-conventions
```
Expected: all pass (pre-existing version warnings are fine).

- [ ] **Step 6: Commit**

```bash
git add plugins/nf-conventions/skills/nf-architecture/
git commit -m "nf-conventions: add config-deploy-ops standard, retire deployment-philosophy

Codifies the E18 unified config/deployment/operations standard as a reference
file (udr baseline + k8s lifecycle contract, canonical ports, admin listener,
JSON logging, CLI-first tooling, reserved YAML->escript config path)."
```

---

### Task 2: Populate the E18 conformance child issues + update epic/standard-child

**Files:** none (GitHub issues only).

**Interfaces:**
- Consumes: the merged/branch reference path; the per-NF conformance list from spec §6.

- [ ] **Step 1: Update each conformance child with its checklist**

Append the per-NF checklist (from spec §6) to each issue body via `gh issue edit -R next-nf/<repo> <n> --body-file ...` (preserve the existing body; append a `### Conformance checklist` section). Use these task-lists:

- `next-nf/udr#29` (reference, nearly conformant): `- [ ] /health + /ready on admin listener (9464)` · `- [ ] structured JSON logging` · `- [ ] bin/udr cluster join|leave|status + health + drain` · `- [ ] StatefulSet + headless Service + ConfigMap manifests`.
- `next-nf/chf#8`: `- [ ] add Containerfile (erlang:29 two-stage)` · `- [ ] /health + /ready on admin listener` · `- [ ] move /metrics to 9464 admin listener` · `- [ ] structured JSON logging` · `- [ ] rename release next-chf -> chf` · `- [ ] k8s manifests` · `- [ ] bin/chf cluster/drain CLI`.
- `next-nf/pcf#8`: same as chf with `next-pcf -> pcf`.
- `next-nf/smf#13`: `- [ ] OTP-29 container base (see #4/E1)` · `- [ ] OTEL metrics on admin listener (see #7/E4)` · `- [ ] /health + /ready on 9464 (replace /status/ready:8000)` · `- [ ] structured JSON logging` · `- [ ] rename release smf-c-node -> smf` · `- [ ] converge YAML config to generate-ahead (escript)` · `- [ ] k8s manifests` · `- [ ] bin/smf cluster/drain CLI`.

- [ ] **Step 2: Update epic #21 and standard child #22**

Edit `next-nf/next-nf#21` body: add a line linking the standard reference
(`plugins/nf-conventions/skills/nf-architecture/reference/config-deploy-ops.md`). Edit
`next-nf/next-nf#22`: add a closing comment that the standard is authored in that reference file and
mark it ready to close on merge:
```bash
gh issue comment next-nf/next-nf#22 --body "Standard authored in reference/config-deploy-ops.md (PR pending). Close on merge."
```

- [ ] **Step 3: Verify**

```bash
gh issue view next-nf/chf#8 --json body -q .body | tail -8
```
Expected: the conformance checklist is present.

---

### Task 3: PR

**Files:** none (git/PR).

- [ ] **Step 1: Push**

```bash
git push -u origin e18-config-deploy-ops-standard
```

- [ ] **Step 2: Open PR**

```bash
gh pr create -R next-nf/next-nf --base main --head e18-config-deploy-ops-standard \
  --title "E18: unified config/deployment/operations standard" \
  --body-file <(...)  # summary: new reference file, retired deployment-philosophy, conformance checklists on E18 children; cross-links E3/E12/E1/E4
```

- [ ] **Step 3: Verify**

```bash
gh pr view --json url,state
```
Expected: open PR. Report URL.

---

## Self-Review

**Spec coverage:** §1 config + reserved YAML path → Task 1 Step 2 (§2 + NOTE); §2 deployment/ports/container/k8s → Task 1 Step 2 (§3); §3 lifecycle contract → Task 1 Step 2 (§4); §4 logging → §5; §5 tooling → §6; §6 runbooks + conformance checklist → §7 (reference) + Task 2 (issues); locked decisions → Global Constraints, carried into Task 1; cross-links + deployment-philosophy replacement → Task 1 Steps 2–4. No gaps.

**Placeholder scan:** The `--body-file <(...)` in Task 3 is the standard PR-body pattern (compose at run time), not an unfilled placeholder; the reference-file section list and the per-NF checklists are fully specified inline. No "TBD/handle edge cases".

**Consistency:** Port values, the `<comp>` naming, the admin-listener (9464) role, and the issue numbers are identical across Global Constraints, Task 1, and Task 2.
