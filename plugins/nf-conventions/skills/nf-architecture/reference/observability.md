# Observability: OpenTelemetry policy

> Grounded as of 2026-06-11. Set policy: **OpenTelemetry is the standard for metrics and traces; Prometheus is being retired.** `udr` is the reference (it is already OTEL-native).

## 1. The policy

1. **Use OpenTelemetry**, not Prometheus, for metrics and traces. Instrument with the OTEL API directly (counters/histograms via `otel_meter`, spans via `?with_span`), the way `udr` does.
2. **Do not use the BEAM `telemetry` library** as a metrics indirection layer. Instrument against the OTEL API directly.
   - **Exception:** the special OTEL handler `udr` already uses — `opentelemetry_cowboy_h` (a Cowboy stream handler that turns HTTP requests into OTEL spans) — is sanctioned and should be reused for HTTP instrumentation. It is not the `telemetry` library; it is the approved way to get HTTP spans.
   - `CONFIRM`: this reads the maintainer's "do not use telemetry (not the special handler already used by the UDR)" as "no `telemetry` library, but keep the special OTEL Cowboy handler udr uses." Adjust this section if the intended exception is different.
3. **Convert existing Prometheus handlers to OTEL** metrics, and expose them through the **OTEL Prometheus exporter** where a `/metrics` scrape endpoint is still wanted. i.e. metrics are *defined* as OTEL instruments; the Prometheus endpoint, if kept, is just an OTEL exporter view — not hand-rolled `prometheus_*` collectors.
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

Reference dep set (from `udr/rebar.config`):

```erlang
{opentelemetry_api,              "1.5.0"},
{opentelemetry,                  "1.7.0"},
{opentelemetry_exporter,         "1.10.0"},
{opentelemetry_api_experimental, "0.5.1"},   %% metrics API
{opentelemetry_experimental,     "0.5.1"},   %% metrics SDK
{opentelemetry_cowboy_h, {git_subdir, "https://github.com/travelping/opentelemetry_cowboy_h.git", {ref, "..."}, "apps/opentelemetry_cowboy_h"}}
```

Add the OTEL **Prometheus exporter** dep where a `/metrics` scrape surface must remain during/after migration.

> [!NOTE]
> Dropping Prometheus also removes the `{override, prometheus, [...nowarn_deprecated_catch]}` hack that `pcf`/`chf`/`smf` need today (prometheus 6.1.2 trips `warnings_as_errors` on OTP-28+).

## 3. OTEL semantic naming

- Name instruments with **dot-separated, lowercase namespaces** following OTEL semantic conventions: `<area>.<thing>.<unit-or-aggregate>` — e.g. `s6a.handler.duration`, `gx.requests`, `pfcp.sessions.active`.
- Set the **unit** on the instrument (`s` for seconds, `1` for counts, `By` for bytes) and use base units (seconds, not milliseconds) for durations — `udr` records latency in seconds.
- Use OTEL **attributes** for dimensions (`s6a.command`, `s6a.result`) rather than baking them into the metric name.
- Reuse 3GPP interface names as the namespace (`s6a`, `gx`, `gy`, `rf`, `pfcp`, `nchf`, `npcf`, `nudr`) so metrics line up with the interface map.
- Pick one span/metric naming scheme org-wide: `<iface>.<command>` for spans (`s6a.AIR`), `<iface>.<subject>` for metrics. Keep it identical across components.

## 4. Document every metric

Each component `shall` carry a `METRICS.md` (or a section in its docs) listing, per instrument: name, type (counter/histogram/observable gauge), unit, attributes/dimensions, what it measures, and since-version. Author it with the `documenting-network-functions` skill (`nf-docs` plugin) — metrics are part of the operator contract.

## 5. Grafana dashboards

- Maintain **one Grafana dashboard per component**, checked into that repo (e.g. `dashboards/` or `priv/grafana/`), as JSON model files so they diff in review.
- Dashboards are driven from the OTEL metrics (via the OTLP→backend pipeline or the OTEL Prometheus exporter).
- Keep dashboards in sync with `METRICS.md` — a metric added without a panel, or a panel referencing a removed metric, is a defect.
- `TODO`: choose the metrics backend the dashboards target (e.g. Prometheus-compatible store fed by the OTEL Prometheus exporter, or an OTLP-native store) and state it here so dashboards are portable across components.

## 6. Checklist for instrumenting a component

- [ ] Metrics defined as OTEL instruments (no `prometheus_*` collectors).
- [ ] HTTP spans via `opentelemetry_cowboy_h`; no BEAM `telemetry` library.
- [ ] Names/units/attributes follow OTEL semantic conventions (§3).
- [ ] OTLP exporter wired; OTEL Prometheus exporter added if `/metrics` is kept.
- [ ] `METRICS.md` updated.
- [ ] Grafana dashboard added/updated and committed.
