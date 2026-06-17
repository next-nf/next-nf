# Config / deployment / operations standard

> Grounded as of 2026-06-17. Set policy: **all four NFs (smf, udr, pcf, chf) share one config/deployment/operations standard.** `udr` is the reference implementation. Conformance is tracked in [next-nf#21](https://github.com/next-nf/next-nf/issues/21) (epic E18 + per-repo children).

## 1. Scope and philosophy

All next-nf network functions behave identically for configuration, deployment, and operations. The standard codifies `udr`'s proven model, adds the pieces missing everywhere (health/readiness, canonical ports, consistent naming, containers for all NFs), and defines a first-class k8s lifecycle contract.

`sys.config` + `vm.args` are the **runtime source of truth**. No env-var override layer exists or is planned. This keeps the config model simple and maps cleanly onto a Kubernetes ConfigMap.

> [!NOTE]
> **Deployment target.** The k8s lifecycle contract is the primary operations target for the control-plane and charging NFs (smf, udr, pcf, chf). For the latency-critical user-plane core, **bare metal remains the preference**: deterministic latency and hardware-line-rate throughput are incompatible with the scheduling and networking abstractions of an orchestrator. Erlang/OTP's built-in supervision, fault isolation, and live-upgrade story provides the resilience an operator would otherwise reach to k8s for — without the orchestration tax. The k8s manifests and lifecycle contract in this file target the signalling/control/charging plane, not a DPDK/SR-IOV user plane. The bare-metal case rests on three concrete arguments: **hardware affinity / the NUMA gap** (CPU pinning and NUMA-local memory the orchestrator can't guarantee), the **networking "death spiral"** (overlay/CNI hops adding latency and loss under load), and **troubleshooting opacity** (layers of abstraction hiding the failure). This is not a blanket ban — it scopes orchestration to where it pays off.

## 2. Config

### Rules

- `config/sys.config` and `config/vm.args` are the runtime config files. **Atom keys, Erlang-typed values** (IP 4-tuples, integer ports, string hostnames). Grouped per OTP application: `<comp>_sbi`, `<comp>_api`, `<comp>_db`, `<comp>_diameter`, `<comp>_otel`, etc. — the `udr` layout.
- Apps read config with `application:get_env(App, Key, Default)` (or `/2` where the key is mandatory). Hardcode sane defaults in the fallback; never rely on `os:getenv/1` at normal startup.
- **Override** by mounting a replacement file:
  - Bare-metal: replace `config/sys.config` in the release.
  - Kubernetes: mount a **ConfigMap** as `container.sys.config`. This file binds all listeners to `0.0.0.0` (the only difference from the bare-metal default, which binds to a specific interface IP).
- No env-var override layer. The `sys.config` mount IS the override mechanism.

> [!NOTE]
> **Reserved future path (specified now, not yet built):** schema-validated YAML → an escript generator → `sys.config`. The escript is runnable as a k8s **initContainer**: ConfigMap YAML → escript → emptyDir `sys.config` → main container. This is the sanctioned convergence path for smf's runtime-YAML model and gives all NFs schema validation. The escript and schema are a later build; do not implement it ad-hoc.

## 3. Deployment

### Release naming

The relx release is named **`<comp>`** — exactly the component short-name, no prefix or suffix:

| Component | Release name |
| --- | --- |
| smf | `smf` (was `smf-c-node`) |
| udr | `udr` |
| pcf | `pcf` (was `next-pcf`) |
| chf | `chf` (was `next-chf`) |

`rebar.config` relx section wires `sys_config`/`vm_args`; use `mode prod` for releases, `mode dev` for shell.

### Container image (two-stage Alpine)

All four NFs use the same two-stage `Containerfile` pattern (the `udr` model):

1. **Build stage:** `FROM erlang:29-alpine` → `rebar3 as container release`
2. **Runtime stage:** `FROM alpine:<pinned>` (pinned to match the ERTS ABI) → copy release → create unprivileged user → `ENTRYPOINT ["bin/<comp>", "foreground"]`

> [!IMPORTANT]
> smf's current `erlang:23.3` base image is replaced as part of its OTP-29 upgrade (E1). Do not update smf's Containerfile to this pattern independently; it is bundled with E1.

### Canonical ports

Every NF exposes only the roles it has. The ports are fixed — do not deviate:

| Role | Port |
| --- | --- |
| Diameter | 3868 |
| SBI (Nxxx, HTTP/2) | 8080 |
| Provisioning / OAM API | 8090 |
| Web UI | 8081 |
| Admin (metrics + health + ready) | 9464 |

The **admin listener** (port 9464) is a dedicated, **uninstrumented** cowboy instance. It serves `/metrics`, `/health`, and `/ready`. Traffic on this port is never self-traced — probe and scrape traffic does not generate OTEL spans.

> [!NOTE]
> This table is the single source of truth for ports ([`conventions.md`](conventions.md) §8 points here). `pcf`/`chf` today run SBI on 8443 and `/metrics` on the web port 8081; their conformance work (E18 children) moves them onto this scheme.

### Kubernetes manifest set

Canonical k8s template set (per NF):

- **StatefulSet** — stable per-pod identity required for Erlang clustering (predictable node names for peer discovery).
- **Headless Service** — DNS-based peer discovery within the StatefulSet.
- **ConfigMap** — mounts `container.sys.config` into the pod.
- **Probes** — liveness `GET /health` (9464), readiness `GET /ready` (9464). See §4 for the exact semantics.
- **preStop hook** — `bin/<comp> drain` (see §4 graceful shutdown).
- **initContainer** (optional) — reserved for the future YAML→`sys.config` generator (§2).

## 4. Operations — k8s lifecycle contract

### Liveness

`GET /health` on port 9464. Returns 200 when the BEAM process is up and the HTTP handler responds. k8s restarts a pod whose liveness probe fails (hung runtime, deadlock, etc.).

### Readiness

`GET /ready` on port 9464. Returns 200 **only when all three conditions hold**:

1. The node has joined its Erlang cluster (cluster-joined).
2. The DB backend is connected and responsive.
3. All business listeners are accepting connections (Diameter, SBI, API, Web — whichever the NF uses).

Readiness gates traffic during startup and rolling updates. A pod that loses DB connectivity or cluster membership flips to not-ready and is removed from Service endpoints until it recovers.

### Graceful shutdown (SIGTERM sequence)

On SIGTERM the node executes in order, within `terminationGracePeriodSeconds`:

1. Flip to not-ready (stop accepting new sessions/requests).
2. Drain in-flight requests and active sessions.
3. Leave the Erlang cluster cleanly.
4. Stop the BEAM.

The k8s manifest template includes a **preStop hook** (`bin/<comp> drain`) that triggers the drain step before SIGTERM, giving the pod time to quiesce before k8s force-kills it.

### Automatic rejoin after unplanned shutdown

On restart, a node **automatically rejoins** the cluster and heals any partitions with no manual operator step. This standard mandates the *outcome*; the mechanism (syn membership, mnesia partition heal, cold-start ordering) is delivered by **E3 (clustering)** — see [next-nf#6](https://github.com/next-nf/next-nf/issues/6); reuse `chf`'s existing cold-start + partition-heal documentation as the reference.

### Rolling updates

Rolling updates proceed pod-by-pod (stop → start) using the StatefulSet update strategy. There is **no hot code upgrade**. The readiness gate ensures traffic only hits ready pods throughout the rollout.

OTP-29 native records let in-flight record values survive a peer's version change during a rolling update — an older node can decode a record tagged by a newer peer (see [`erlang-otp.md`](erlang-otp.md)). This is the key property that makes pod-by-pod rolling updates safe without a coordinated cutover.

## 5. Logging

- **Structured JSON to stdout** via an OTP `logger` formatter, standard level (`info`) and handler config across all NFs.
- k8s collects stdout and forwards to the log aggregator; no log file management in the container.
- **OTEL spans and metrics remain the primary observability surface** (see [`observability.md`](observability.md)). Logs are for discrete events and errors, not request tracing.
- Use `logger:info/2`, `logger:warning/2`, `logger:error/2` with a metadata map. Do not use `error_logger` or `io:format` for operational logging.

## 6. Tooling — CLI first

`bin/<comp>` (the relx extended start script) provides subcommands. All subcommands are **non-interactive with proper exit codes** so they work from initContainers, preStop hooks, probes, and automation scripts:

| Subcommand | Purpose |
| --- | --- |
| `cluster join <node>` | Join the named node into the Erlang cluster |
| `cluster leave` | Leave the cluster cleanly (for planned maintenance) |
| `cluster status` | Print cluster membership; exit 0 if joined, non-zero if not |
| `health` | Exit 0 if healthy (same signal as `GET /health`) |
| `drain` | Quiesce the node for graceful shutdown or rolling update |

> [!IMPORTANT]
> Cluster bring-up **must never depend on a web stack being up**. The CLI subcommands operate over the Erlang distribution port, not HTTP. An optional read-only ops API (cluster + node status, health) may be added later and is folded into **E12** (the web/API surface epic).

## 7. Runbooks

Operator runbooks follow `udr`'s `docs/manual/operations` format:

**Sections (in order):** Purpose · Pre-conditions · Inputs · Steps · Verify · Rollback · Related

**Rule:** runbook IDs are stable and never renumber. A retired runbook gets a tombstone, not a gap.

Standard runbook set for each NF (author with the `documenting-network-functions` skill — `nf-docs` plugin):

| Runbook | Content |
| --- | --- |
| Deploy | Initial deploy, ConfigMap wiring, first pod start |
| Lifecycle | Start / stop / restart a single pod |
| Rolling update | Pod-by-pod update procedure, readiness verification |
| Cluster recovery | Recovery from split-brain or total-cluster restart |
| Backend | DB backend connect/disconnect, failover |
| Observability | Metrics scrape, log aggregation, alert routing |

## 8. Cross-links and tracking

- **E3 (clustering):** delivers the automatic-rejoin / partition-heal mechanism behind §4 recovery. See [next-nf#6](https://github.com/next-nf/next-nf/issues/6).
- **E12 (ops API / web surface):** absorbs the optional read-only ops API from §6. See [next-nf#20](https://github.com/next-nf/next-nf/issues/20).
- **E1 (OTP-29 / container base):** smf's erlang:29 Alpine image, bundled with the OTP-29 upgrade.
- **E4 (OTEL metrics):** the metrics served by the admin listener on port 9464. See [`observability.md`](observability.md) and [next-nf#7](https://github.com/next-nf/next-nf/issues/7).

> **Tracking:** conformance is tracked in [next-nf#21](https://github.com/next-nf/next-nf/issues/21) (epic + per-repo children).
