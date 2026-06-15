# Observability: OpenTelemetry policy

> Grounded as of 2026-06-11. Set policy: **OpenTelemetry is the standard for metrics and traces; Prometheus is being retired.** `udr` is the reference (it is already OTEL-native).

## 1. The policy

1. **Use OpenTelemetry**, not Prometheus, for metrics and traces. Instrument with the OTEL API directly (counters/histograms via `otel_meter`, spans via `?with_span`), the way `udr` does.
2. **Do not use the BEAM `telemetry` library** as a metrics indirection layer. Instrument against the OTEL API directly.
   - **Exception:** the OTEL Cowboy stream handler `udr` already uses — `opentelemetry_cowboy_h` (turns HTTP requests into OTEL spans), plus its metrics counterpart `opentelemetry_cowboy_experimental_h` — is sanctioned and should be reused for HTTP instrumentation. These are not the `telemetry` library; they are the approved way to get HTTP spans and server metrics. Use the **next-nf forks** of both (§2).
3. **Convert existing Prometheus handlers to OTEL** metrics, and expose them through the **OTEL Prometheus exporter** (provided by the next-nf `opentelemetry-erlang` fork, §2) where a `/metrics` scrape endpoint is still wanted. i.e. metrics are *defined* as OTEL instruments; the Prometheus endpoint, if kept, is just an OTEL exporter view — not hand-rolled `prometheus_*` collectors.
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
- Set the **unit** on the instrument (`s` for seconds, `1` for counts, `By` for bytes) and use base units (seconds, not milliseconds) for durations — `udr` records latency in seconds.
- Use OTEL **attributes** for dimensions (`s6a.command`, `s6a.result`) rather than baking them into the metric name.
- Reuse 3GPP interface names as the namespace (`s6a`, `gx`, `gy`, `rf`, `pfcp`, `nchf`, `npcf`, `nudr`) so metrics line up with the interface map.
- Pick one span/metric naming scheme org-wide: `<iface>.<command>` for spans (`s6a.AIR`), `<iface>.<subject>` for metrics. Keep it identical across components.

## 4. Document every metric

Each component `shall` carry a `METRICS.md` (or a section in its docs) listing, per instrument: name, type (counter/histogram/observable gauge), unit, attributes/dimensions, what it measures, and since-version. **This includes the metrics that the instrumentation libraries emit, not just hand-defined ones** — see [`instrumentation-metrics.md`](instrumentation-metrics.md). Author it with the `documenting-network-functions` skill (`nf-docs` plugin) — metrics are part of the operator contract.

## 5. Grafana dashboards

- Maintain **one Grafana dashboard per component**, checked into that repo (e.g. `dashboards/` or `priv/grafana/`), as JSON model files so they diff in review.
- Dashboards are driven from the OTEL metrics (via the OTLP→backend pipeline or the OTEL Prometheus exporter).
- Keep dashboards in sync with `METRICS.md` — a metric added without a panel, or a panel referencing a removed metric, is a defect.
- `TODO`: choose the metrics backend the dashboards target (e.g. Prometheus-compatible store fed by the OTEL Prometheus exporter, or an OTLP-native store) and state it here so dashboards are portable across components.

## 6. Checklist for instrumenting a component

- [ ] Metrics defined as OTEL instruments (no `prometheus_*` collectors).
- [ ] HTTP spans via `opentelemetry_cowboy_h`; no BEAM `telemetry` library.
- [ ] HTTP **metrics** enabled (`opentelemetry_cowboy_experimental_h:init()` + `metrics_cb`) — they are opt-in; see [`instrumentation-metrics.md`](instrumentation-metrics.md).
- [ ] Library-emitted metrics (cowboy / diameter / beam / process) documented in `METRICS.md`, per [`instrumentation-metrics.md`](instrumentation-metrics.md).
- [ ] Names/units/attributes follow OTEL semantic conventions (§3).
- [ ] Deps point at the **next-nf OTEL forks** (§2), not hex/travelping/upstream.
- [ ] OTLP exporter wired; OTEL Prometheus exporter (from the next-nf fork, §2) enabled if `/metrics` is kept.
- [ ] `METRICS.md` updated.
- [ ] Grafana dashboard added/updated and committed.
