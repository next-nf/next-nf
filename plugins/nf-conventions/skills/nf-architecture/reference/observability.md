# Observability: OpenTelemetry policy

> Grounded as of 2026-06-15. Set policy: **OpenTelemetry is the standard for metrics and traces; Prometheus is being retired.** `udr` is the reference (it is already OTEL-native).

## 1. The policy

1. **Use OpenTelemetry**, not Prometheus, for metrics and traces. Instrument with the OTEL API directly (counters/histograms via `otel_meter`, spans via `?with_span`), the way `udr` does.
2. **Do not use the BEAM `telemetry` library** as a metrics indirection layer. Instrument against the OTEL API directly.
   - **Exception:** the OTEL Cowboy stream handler `udr` already uses — `opentelemetry_cowboy_h` (turns HTTP requests into OTEL spans), plus its metrics counterpart `opentelemetry_cowboy_experimental_h` — is sanctioned and should be reused for HTTP instrumentation. These are not the `telemetry` library; they are the approved way to get HTTP spans and server metrics. Use the **next-nf forks** of both (§2).
3. **Convert existing Prometheus handlers to OTEL** metrics, and expose them through the **OTEL Prometheus exporter** (provided by the next-nf `opentelemetry-erlang` fork, §2) where a `/metrics` scrape endpoint is still wanted. i.e. metrics are *defined* as OTEL instruments; the Prometheus endpoint, if kept, is just an OTEL exporter view — not hand-rolled `prometheus_*` collectors. The exporter is not turnkey — wiring it (second reader + vendored cowboy handler, and the broken inets path to avoid) is spelled out in [`instrumentation-metrics.md`](instrumentation-metrics.md) §5.
4. **Follow OTEL semantic conventions** for naming (§3).
5. **Document every metric** (§4).
6. **Maintain a Grafana dashboard per component** (§5).

## 2. Current state & migration

| Repo | Today | Migration |
| --- | --- | --- |
| `udr` | ✅ OTEL-native. `udr_otel` app: `otel_meter:create_counter('s6a.requests')`, `create_histogram('s6a.handler.duration')`; spans via `?with_span` (`s6a.AIR`/`ULR`/`PUR`); HTTP via `opentelemetry_cowboy_h`; OTLP exporter. No Prometheus, no `telemetry`. | None — **the reference.** |
| `smf` | ❌ Prometheus, with **real custom metrics**: `smf_core/src/smf_prometheus.erl` + `smf_prometheus_collector.erl` (GTP-C/U, PFCP gauges), `smf_aaa` Diameter collectors, `/metrics` via `prometheus_cowboy2_handler`. | Largest conversion — port each metric to an OTEL instrument; replace the custom `prometheus_collector`s with OTEL observable instruments. Also: custom collectors are only registered in test suites today — wire them (as OTEL) into app startup. |
| `pcf` | ❌ Prometheus deps + `/metrics` via `prometheus_cowboy_handler` on `pcf_web` (:8081), **but no custom metrics defined**. | Light — swap the exporter for OTEL + OTEL Prometheus exporter; add OTEL instruments for the policy path (Gx CCR/CCA, session counts). |
| `chf` | ❌ Prometheus deps + `/metrics` on `chf_web` (:8081), **no custom metrics**; a separate JSON dashboard endpoint reads VM stats. | Light — same as pcf; add OTEL instruments for the charging path (Gy/Ro, Rf, balance ops). |

### Dependency set — use the next-nf forks

All components **shall** depend on the **next-nf forks** of the OpenTelemetry Erlang libraries (maintained pending upstream), **not** the hex releases or the upstream/travelping git sources. The core libraries come from the `feature/prometheus-exporter-for-upstream` branch of `next-nf/opentelemetry-erlang`, which **carries the OTEL Prometheus exporter** — the next-nf addition being prepared for upstream. That is how the Prometheus `/metrics` surface is served from OTEL instruments; there is no separate Prometheus-exporter package to add.

```erlang
%% rebar.config — OpenTelemetry core (next-nf fork; this branch carries the Prometheus exporter)
{opentelemetry,                  {git_subdir, "https://github.com/next-nf/opentelemetry-erlang.git", {branch, "feature/prometheus-exporter-for-upstream"}, "apps/opentelemetry"}},
{opentelemetry_api,              {git_subdir, "https://github.com/next-nf/opentelemetry-erlang.git", {branch, "feature/prometheus-exporter-for-upstream"}, "apps/opentelemetry_api"}},
{opentelemetry_experimental,     {git_subdir, "https://github.com/next-nf/opentelemetry-erlang.git", {branch, "feature/prometheus-exporter-for-upstream"}, "apps/opentelemetry_experimental"}},      %% metrics SDK
{opentelemetry_api_experimental, {git_subdir, "https://github.com/next-nf/opentelemetry-erlang.git", {branch, "feature/prometheus-exporter-for-upstream"}, "apps/opentelemetry_api_experimental"}},  %% metrics API
{opentelemetry_exporter,         {git_subdir, "https://github.com/next-nf/opentelemetry-erlang.git", {branch, "feature/prometheus-exporter-for-upstream"}, "apps/opentelemetry_exporter"}},

%% HTTP instrumentation (next-nf fork) — spans + experimental HTTP server metrics
{opentelemetry_cowboy_h,              {git_subdir, "https://github.com/next-nf/opentelemetry_cowboy_h.git", {branch, "main"}, "apps/opentelemetry_cowboy_h"}},
{opentelemetry_cowboy_experimental_h, {git_subdir, "https://github.com/next-nf/opentelemetry_cowboy_h.git", {branch, "main"}, "apps/opentelemetry_cowboy_experimental_h"}},

%% BEAM/VM, Diameter + process instrumentation (next-nf contrib fork)
{opentelemetry_beam,               {git_subdir, "https://github.com/next-nf/opentelemetry-erlang-contrib.git", {branch, "main"}, "instrumentation/opentelemetry_beam"}},
{opentelemetry_diameter,           {git_subdir, "https://github.com/next-nf/opentelemetry-erlang-contrib.git", {branch, "main"}, "instrumentation/opentelemetry_diameter"}},
{opentelemetry_process,            {git_subdir, "https://github.com/next-nf/opentelemetry-erlang-contrib.git", {branch, "main"}, "instrumentation/opentelemetry_process"}},
{opentelemetry_process_propagator, {git_subdir, "https://github.com/next-nf/opentelemetry-erlang-contrib.git", {branch, "main"}, "propagators/opentelemetry_process_propagator"}}
```

> [!IMPORTANT]
> **The git deps above do not resolve on their own — you must also ship the `overrides` below.** The fork apps declare their inter-app dependencies as hex-style version constraints; rebar3 honours those constraints over the top-level git deps, so a verbatim `rm rebar.lock && rebar3 compile` either hard-fails with `Package not found: opentelemetry_api_experimental ~> 0.5.2` or — where hex *does* have a matching version — silently pulls the **hex** build instead of the fork (you end up with a hex/fork mix and **no Prometheus exporter**). The `overrides` strip those constraints so the git sources win:
>
> ```erlang
> %% rebar.config — required alongside the OTEL deps above
> {overrides, [
>   {override, opentelemetry_api,              [{deps, []}]},
>   {override, opentelemetry_api_experimental, [{deps, []}]},
>   {override, opentelemetry,                  [{deps, []}]},
>   {override, opentelemetry_experimental,     [{deps, []}]},
>   {override, opentelemetry_exporter, [{deps, [{grpcbox, ">= 0.0.0"}, {tls_certificate_check, "~> 1.18"}]}]}
> ]}.
> ```

> [!NOTE]
> **Re-sourcing OTEL in an existing repo:** changing the `source` of an already-locked dep in `rebar.config` is **ignored** — rebar3 keeps the locked source. After switching to these forks (or moving a branch/ref) you must `rm rebar.lock` (or `rebar3 upgrade <dep>`) for the new sources to take effect.

What the libraries beyond the core SDK give you:
- `opentelemetry_cowboy_h` — HTTP requests → OTEL **spans** (the handler `udr` already uses); `opentelemetry_cowboy_experimental_h` adds HTTP **server metrics**.
- `opentelemetry_beam` — BEAM/VM runtime metrics (schedulers, memory, run-queue, GC).
- `opentelemetry_diameter` — OTEL spans/metrics for the OTP `diameter` stack (requests/answers per interface — S6a, Gx, Gy, Rf). This is the Diameter-side counterpart to the Cowboy handler's HTTP/SBI instrumentation; pair it with the Diameter discipline in [`diameter.md`](diameter.md).
- `opentelemetry_process` — per-process metrics; `opentelemetry_process_propagator` carries OTEL context across process boundaries (spawned workers, pool processes) so spans stay correlated.

> [!IMPORTANT]
> **What metrics each library emits — names, units, attributes, setup, and gotchas — is captured in [`instrumentation-metrics.md`](instrumentation-metrics.md).** The cowboy HTTP metrics are spelled out there (they are effectively undocumented upstream, and the cowboy metrics are **opt-in** — spans are automatic, metrics need `init()` + a `metrics_cb`); the `opentelemetry_diameter` and BEAM/process metrics are linked to their own docs. Read it before instrumenting or building a dashboard.

> [!NOTE]
> These forks **supersede** the earlier reference dep set (hex `opentelemetry*` plus the `travelping/opentelemetry_cowboy_h` git source). Pin via `{branch, ...}` as above; move to a tag / `{ref, ...}` once the Prometheus-exporter work lands upstream and the forks can be retired.

> [!NOTE]
> Dropping Prometheus also removes the `{override, prometheus, [...nowarn_deprecated_catch]}` hack that `pcf`/`chf`/`smf` need today (prometheus 6.1.2 trips `warnings_as_errors` on OTP-28+).

## 3. OTEL semantic naming

- Name instruments with **dot-separated, lowercase namespaces** following OTEL semantic conventions: `<area>.<thing>.<unit-or-aggregate>` — e.g. `s6a.handler.duration`, `gx.requests`, `pfcp.sessions.active`.
- Set the **unit** on the instrument (`s` for seconds, `By` for bytes) and use base units (seconds, not milliseconds) for durations — `udr` records latency in seconds.
- **Do not use `1` as the unit for counts.** Under the OTEL→Prometheus transform, unit `1` is treated as a dimensionless ratio and gets a `_ratio` suffix — a request counter becomes `foo_requests_ratio_total`, not `foo_requests_total`. For counts, use an **annotation unit** naming what is counted in braces (`{request}`, `{connection}`, `{session}`); the Prometheus exporter strips annotation units, leaving the clean `foo_requests_total`. The org's own `opentelemetry_diameter` already does this — follow it, not the older `1`-for-counts advice.
- Use OTEL **attributes** for dimensions (`s6a.command`, `s6a.result`) rather than baking them into the metric name.
- Reuse 3GPP interface names as the namespace (`s6a`, `gx`, `gy`, `rf`, `pfcp`, `nchf`, `npcf`, `nudr`) so metrics line up with the interface map.
- Pick one span/metric naming scheme org-wide: `<iface>.<command>` for spans (`s6a.AIR`), `<iface>.<subject>` for metrics. Keep it identical across components.

## 4. Document every metric

Each component `shall` carry a `METRICS.md` (or a section in its docs) listing, per instrument: name, type (counter/histogram/observable gauge), unit, attributes/dimensions, what it measures, and since-version. **This includes the metrics that the instrumentation libraries emit, not just hand-defined ones** — see [`instrumentation-metrics.md`](instrumentation-metrics.md). Author it with the `documenting-network-functions` skill (`nf-docs` plugin) — metrics are part of the operator contract.

## 5. Grafana dashboards

- Maintain **one Grafana dashboard per component**, checked into that repo (e.g. `dashboards/` or `priv/grafana/`), as JSON model files so they diff in review.
- Dashboards are driven from the OTEL metrics (via the OTLP→backend pipeline or the OTEL Prometheus exporter).
- Keep dashboards in sync with `METRICS.md` — a metric added without a panel, or a panel referencing a removed metric, is a defect.
- **The same dashboard works against either path** — OTLP→Alloy *or* the in-process OTEL Prometheus exporter — because both apply the **same OTEL→Prometheus name transform** (the `_total` counter suffix, unit suffixes, `_ratio` for unit `1`, annotation-unit stripping). The condition for portability is that the **suffix settings match on both sides**: align `add_total_suffix` and `add_metric_suffixes` between the in-process Prometheus reader and Alloy's converter (Alloy defaults to adding `_total`; set the reader's `add_total_suffix => true` to match — see [`instrumentation-metrics.md`](instrumentation-metrics.md)). With suffixes aligned, panel queries are identical across both backends.
- `TODO` (remaining): pick which path is the deployment default per environment (OTLP-native store vs. Prometheus scrape of the in-process exporter). Either works for the dashboards once suffixes are aligned, so this is an ops/topology choice, no longer a portability blocker.

## 6. Checklist for instrumenting a component

- [ ] Metrics defined as OTEL instruments (no `prometheus_*` collectors).
- [ ] HTTP spans via `opentelemetry_cowboy_h`; no BEAM `telemetry` library.
- [ ] HTTP **metrics** enabled (`opentelemetry_cowboy_experimental_h:init()` + `metrics_cb`) — they are opt-in; see [`instrumentation-metrics.md`](instrumentation-metrics.md).
- [ ] Library-emitted metrics (cowboy / diameter / beam / process) documented in `METRICS.md`, per [`instrumentation-metrics.md`](instrumentation-metrics.md).
- [ ] Names/units/attributes follow OTEL semantic conventions (§3).
- [ ] Deps point at the **next-nf OTEL forks** (§2), not hex/travelping/upstream — **and the `overrides` block is present** (the git deps don't resolve without it).
- [ ] OTLP exporter wired; OTEL Prometheus exporter enabled if `/metrics` is kept — second reader + vendored cowboy handler with `add_total_suffix => true` ([`instrumentation-metrics.md`](instrumentation-metrics.md) §5).
- [ ] `METRICS.md` updated.
- [ ] Grafana dashboard added/updated and committed.
