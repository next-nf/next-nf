# Metrics emitted by the OTEL instrumentation libraries

> Grounded as of 2026-06-15, read from the **next-nf forks** ([`observability.md`](observability.md) §2 lists the deps and branches). Some of these libraries are thinly documented upstream — the cowboy HTTP metrics below were read out of the fork source so you don't have to. Re-verify against the pinned branch if it moves.

## Why this file exists

`opentelemetry_cowboy_h` / `opentelemetry_cowboy_experimental_h` emit metrics that are effectively undocumented upstream; the names, units, and attributes here come straight from the source. `opentelemetry_diameter` and the BEAM/process libraries **are** documented elsewhere, so this file points to those rather than duplicating them.

## 1. HTTP server metrics — `opentelemetry_cowboy_h` + `opentelemetry_cowboy_experimental_h`

### Spans are automatic; metrics are opt-in

`opentelemetry_cowboy_h` (the stream handler) emits **spans** as soon as it is in the listener's `stream_handlers`. **Metrics require explicit wiring** — three steps:

1. depend on `opentelemetry_cowboy_experimental_h` (it needs the experimental metrics API/SDK, `opentelemetry_api_experimental` / `opentelemetry_experimental` — already in the §2 dep set);
2. call `opentelemetry_cowboy_experimental_h:init()` **once** at startup to create the instruments;
3. pass the callback into the listener's `otel_opts`:

```erlang
opentelemetry_cowboy_experimental_h:init(),
cowboy:start_clear(http, [{port, Port}], #{
    env => #{dispatch => Dispatch},
    otel_opts => #{metrics_cb => fun opentelemetry_cowboy_experimental_h:metrics_cb/5},
    stream_handlers => [opentelemetry_cowboy_h, cowboy_stream_h]
}).
```

Without `init()` + `metrics_cb` you get **spans but no metrics**.

### The instruments

All three are **histograms**, recorded once per request:

| Instrument | Type | Unit | When recorded |
| --- | --- | --- | --- |
| `http.server.request.duration` | histogram | `s` (seconds) | every request (native time converted to seconds) |
| `http.server.request.body.size` | histogram | `By` (bytes) | only when the request body size is a number |
| `http.server.response.body.size` | histogram | `By` (bytes) | every request |

### Attributes

Each histogram is recorded with `maps:with/2` over this set (the **stable** HTTP OTEL semantic conventions):

`http.request.method`, `url.scheme`, `error.type`, `http.response.status_code`, `http.route`, `network.protocol.name`, `network.protocol.version`, `server.address`, `server.port`.

> [!WARNING]
> Caveats found in the source — do not assume otherwise:
> - **`http.route` and `error.type` are in the filter but the handler never sets them.** Cowboy hands the handler no matched-route string, and `error.type` is not populated, so these two dimensions are **absent out of the box** — you will not get per-route latency breakdowns without adding the attribute yourself.
> - **`http.server.response.body.size` is recorded even when the size is `undefined`** (unlike request body size, which is guarded with `is_number/1`). Be skeptical of odd values on that histogram.
> - **Spans and metrics use different attribute vintages.** The span carries the *legacy* `http.*` keys (`http.method`, `http.scheme`, `http.status_code`, `http.flavor`); the metrics carry the *stable* semconv keys (`http.request.method`, `url.scheme`, …). Don't expect span and metric attribute keys to line up.

### Spans (for reference)

`opentelemetry_cowboy_h` starts one `SERVER`-kind span per request named `HTTP <METHOD>` (e.g. `HTTP GET`), sets `otel.status = error` for error status codes, and attaches the legacy `http.*` / `network.*` attributes. Spans are the automatic part; consult the handler source for the full attribute list.

## 2. Diameter metrics — `opentelemetry_diameter`

Documented in its own repo — **use those docs, don't duplicate them here**:
<https://github.com/next-nf/opentelemetry-erlang-contrib/tree/main/instrumentation/opentelemetry_diameter>

For quick orientation it defines (see the README's `## Metrics` for units/attributes): `diameter.application.count`, `diameter.connection.count`, `diameter.message.count`, `diameter.connection.io`, `diameter.connection.packets`, `diameter.error.count`. Map these onto the interface namespaces in [`observability.md`](observability.md) §3 and pair with the Diameter discipline in [`diameter.md`](diameter.md).

## 3. BEAM/VM & process metrics — `opentelemetry_beam`, `opentelemetry_process`

These have no dedicated docs in their repo; their metric names, units, and attributes are governed by the next-nf **semantic conventions (BEAM VM)** branch — treat it as the source of truth and align dashboards / `METRICS.md` to it:
<https://github.com/next-nf/semantic-conventions/tree/add/beam-vm>

(`opentelemetry_process_propagator` carries OTEL **context** across process boundaries — it emits no metrics.)

## 4. These count toward the per-component contract

Library-provided metrics are part of each component's metrics surface, so they belong in its `METRICS.md` and Grafana dashboard ([`observability.md`](observability.md) §4–5) exactly like hand-defined instruments. Document the cowboy/diameter/beam metrics you **actually enable** — do not list instruments you never `init()`, and don't promise `http.route`/`error.type` dimensions you haven't added.
