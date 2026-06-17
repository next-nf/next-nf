# Design: unified config / deployment / operations standard (E18)

**Date:** 2026-06-17
**Tracks:** `next-nf/next-nf#21` (epic E18) · standard child `next-nf/next-nf#22`
**Deliverable:** a new `nf-conventions` reference (`reference/config-deploy-ops.md`, replacing the old
`deployment-philosophy.md` content) + a per-repo conformance checklist. **This cycle defines the
standard; it does not change the NF repos** (conformance is per-repo follow-on work).

## Context

The four NFs diverge on config, deployment, and operations: only udr and smf are containerized (smf
on a stale erlang:23.3), only smf has a health endpoint, SBI ports disagree (udr 8080 vs chf/pcf
8443), metrics live on different ports/frameworks, release names are inconsistent (`udr` /
`next-chf` / `next-pcf` / `smf-c-node`), and none support automated k8s lifecycle. The goal: **all
components behave the same way** for config, deployment, and operations, with first-class support for
**k8s rolling updates and recovery from unplanned shutdowns** (needed soon).

**Approach:** codify udr's proven model as the baseline, add the few things missing everywhere
(standard health/readiness, canonical ports, consistent naming, container for all), and add a k8s
lifecycle contract. udr's "mount `sys.config`" model maps directly onto a k8s ConfigMap, so **no
env-var override layer is needed**. The future YAML→`sys.config` path becomes a k8s initContainer.

## Locked decisions

- Release name: **`<comp>`** (drop `next-` / `smf-c-node`).
- **Dedicated admin listener** (uninstrumented cowboy) serves metrics + health + readiness, so
  probe/scrape traffic is not self-traced.
- **Structured JSON logging to stdout** (changes udr's untuned default; k8s log aggregation).
- **StatefulSet + headless Service** for the clustered, state-bearing NFs.
- `sys.config` + `vm.args` remain the **runtime source of truth**; no env-var override layer.

## 1. Config

- `config/sys.config` + `config/vm.args` are the runtime truth. Keys are **atoms**, values use Erlang
  types (IP tuples, integer ports, string hostnames), grouped per OTP app (`<comp>_sbi`, `<comp>_api`,
  `<comp>_db`, `<comp>_diameter`, `<comp>_otel`, …) — the udr layout.
- **Override** by mounting a replacement `sys.config` (bare-metal) or a **ConfigMap-mounted**
  `container.sys.config` (k8s) that binds listeners to `0.0.0.0`. No `os:getenv` at normal startup.
- Apps read config via `application:get_env/2,3` with sane hardcoded defaults.
- **Reserved future authoring path (specified now, built later):** schema-validated **YAML → an
  escript generator → `sys.config`**, runnable as a k8s **initContainer** (ConfigMap YAML → escript →
  emptyDir `sys.config` → main container). This is the sanctioned way smf's runtime-YAML converges to
  "generate-ahead," and gives the others schema validation. The escript + schema are a later build.

## 2. Deployment

- **relx release** named `<comp>`; prod profile bundles ERTS; `sys_config`/`vm_args` wired in
  `rebar.config` relx section; `mode prod` for releases, `dev` for shell.
- **Container:** the udr **two-stage Alpine `Containerfile`** for all four — build on `erlang:29-alpine`
  (`rebar3 as container release`), runtime on pinned `alpine:3.x` matching the ERTS ABI, unprivileged
  user, `ENTRYPOINT [".../bin/<comp>", "foreground"]`. (smf's erlang:23.3 image is replaced as part of
  its OTP-29 upgrade, E1.)
- **Canonical ports** (an NF omits a role it doesn't have):

  | Role | Port | Listener |
  | --- | --- | --- |
  | Diameter | 3868 | `<comp>_diameter` |
  | SBI (Nxxx, HTTP/2) | 8080 | `<comp>_sbi` |
  | Provisioning / OAM API | 8090 | `<comp>_api` |
  | Web UI | 8081 | `<comp>_web` |
  | Admin (metrics + health + ready) | 9464 | dedicated uninstrumented cowboy |

- **k8s manifests** (canonical templates in the standard): **StatefulSet** (stable per-pod identity for
  clustering) + **headless Service** (DNS peer discovery) + **ConfigMap** (`sys.config`) + probes +
  optional initContainer (future config generator).

## 3. Operations — k8s lifecycle contract

- **Liveness probe** → `GET /health` on the admin port (9464): process up and responsive. k8s restarts
  hung pods.
- **Readiness probe** → `GET /ready` on 9464: **true only when** cluster-joined **and** DB backend
  connected **and** business listeners accepting. Gates traffic during startup and rolling updates.
- **Graceful shutdown:** on SIGTERM → flip to not-ready → drain in-flight requests/sessions → leave the
  cluster cleanly → stop, all within `terminationGracePeriodSeconds`. A preStop hook (`bin/<comp> drain`
  then wait) is part of the manifest template.
- **Unplanned-shutdown recovery:** on restart a node **automatically rejoins** the cluster and heals
  partitions with no manual step. E18 mandates this *outcome*; the mechanism (syn/mnesia membership,
  cold-start ordering, partition heal) is delivered by **E3 clustering** (chf already documents
  cold-start + partition heal — reuse that).
- **Rolling updates:** pod-by-pod stop/start (no hot code upgrade). The cluster tolerates rolling
  membership; OTP-29 native records let in-flight record values survive a peer's version change
  (see erlang-otp conventions). Readiness gating ensures traffic only hits ready pods.

## 4. Logging

- **Structured JSON to stdout** via an OTP `logger` formatter, standard level (`info`) and handler
  config across all NFs. k8s collects stdout; OTEL spans/metrics remain the primary observability
  surface (logs are for events/errors, not request tracing).

## 5. Tooling — CLI first

- `bin/<comp>` (relx extended start script) subcommands, **non-interactive with proper exit codes** so
  they work from initContainers, probes, and automation:
  - `cluster join <node>` / `cluster leave` / `cluster status`
  - `health` (exit 0/non-0; same signal as `/health`)
  - `drain` (quiesce for graceful shutdown / rolling update)
- An optional **read-only ops API** (cluster + node status, health) may layer on later and is folded
  into **E12** (the web/API surface) — cluster bring-up must never depend on a web stack being up.

## 6. Runbooks + conformance

- **Runbooks** follow udr's `docs/manual/operations` format (Purpose · Pre-conditions · Inputs · Steps ·
  Verify · Rollback · Related) with stable IDs that never renumber. Standard set: deploy, lifecycle,
  **rolling-update**, **cluster-recovery**, backend, observability. (Authoring uses the `nf-docs` skill,
  E8.)
- **Per-NF conformance checklist** (the standard ships this; each item becomes per-repo work):
  - **udr** (reference): add `/health`+`/ready` + admin listener; JSON logging; CLI cluster/drain;
    rename already `udr`. Nearly conformant.
  - **chf/pcf:** add a `Containerfile`; `/health`+`/ready` + admin listener; move `/metrics` to 9464;
    JSON logging; rename `next-chf`/`next-pcf` → `chf`/`pcf`; k8s manifests; CLI cluster/drain.
  - **smf:** the most — OTP-29 container (E1), OTEL metrics (E4), `/health`+`/ready` on the admin port
    (today `/status/ready` on 8000), JSON logging, rename `smf-c-node` → `smf`, converge YAML config to
    generate-ahead, k8s manifests, CLI cluster/drain.

## Cross-links

- **E3 (clustering):** supplies the rejoin/partition-heal mechanism behind §3 recovery.
- **E12 (UI/API):** absorbs the optional read-only ops API behind §5.
- **E1 (OTP-29):** smf's erlang:29 container base.
- **E4 (OTEL):** the metrics on the admin listener.
- This standard replaces the `deployment-philosophy.md` content (now a tracking link to E18).

## Out of scope

Building the escript config generator, the clustering mechanism (E3), the ops API/UI (E12), and the
per-repo conformance changes. Those are later cycles tracked by their epics/children.
