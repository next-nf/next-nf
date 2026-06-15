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

## 4. Recording your own metrics — the experimental API shape

Hand-defined instruments use the **experimental** metrics API (`opentelemetry_api_experimental` / `opentelemetry_experimental`). The fork's API differs from the hex-era one — two things bit `udr_otel`:

- **Record by `(meter, name)`, not by an instrument handle.** The hex-era API where `create_counter/…` returned a handle you later passed to `add/2` is **gone**. You create the instrument once, then record against the **meter + the instrument's name**, passing the current OTEL context as the first argument. The recording calls are `otel_counter:add/5` and `otel_histogram:record/5`, both `(Ctx, Meter, Name, Number, Attributes)` — **there is no 4-arg variant**, so a `add(Meter, Name, Number, Attrs)` call fails with an undefined-function error. Use **atom** attribute keys (as the cowboy/diameter instrumentation libraries do — not the binary keys used for JSON data):

  ```erlang
  %% create once at startup
  otel_meter:create_counter(Meter, 's6a.requests', #{unit => '{request}', description => "..."}),
  %% record on the hot path — by name, against the meter, with the current OTEL context
  Ctx = otel_ctx:get_current(),
  otel_counter:add(Ctx, Meter, 's6a.requests', 1, #{'s6a.command' => <<"AIR">>}).
  %% histograms mirror it: otel_histogram:record(Ctx, Meter, Name, Number, Attributes)
  ```

- **Resolve the meter via `opentelemetry:get_application_scope(?MODULE)`, not a bare module atom.** Passing a bare module atom to `otel_meter:get_meter/1` **dialyzer-fails** the fork's `get_meter/1` spec. Use:

  ```erlang
  Meter = opentelemetry_experimental:get_meter(opentelemetry:get_application_scope(?MODULE)),
  ```

  `get_application_scope/1` returns the proper instrumentation-scope term the spec expects (name + version derived from the owning application).

## 5. Serving `/metrics` from the OTEL Prometheus exporter

[`observability.md`](observability.md) §1/§5 say to expose `/metrics` "through the OTEL Prometheus exporter" but leave the *how* unspecified — and the path has landmines. Concretely:

- **The exporter lives inside `opentelemetry_experimental`** (`otel_metric_reader_prometheus` plus its serializer). You wire it as a **second metric reader** alongside the OTLP reader — the Prometheus reader holds the latest snapshot, which you serialize on each scrape.
- **Set `add_total_suffix => true` on the reader.** This makes the in-process exporter add the `_total` counter suffix the same way Alloy's OTLP→Prometheus converter does by default, so the **same dashboard works against both backends** (see [`observability.md`](observability.md) §5). Keep `add_total_suffix` / `add_metric_suffixes` aligned on both sides.
- **Do not use the standalone inets path (`otel_metric_reader_prometheus_httpd`) — it is broken in the fork.** Its `do/1` reads `otel_config` from the `ConfigDb` without the list-unwrap its sibling lookups use (inets stores custom config list-wrapped), so it passes `[Config]` instead of `Config` into the serializer and every `GET /metrics` 500s with `{badmap, …}` in `maps:merge/2`. Don't waste time on it. *(Known fork bug — a one-line `[Config] = …` unwrap fixes it; see the §5 note below.)*
- **The working HTTP surface is a cowboy handler that ships only as a `samples/` file — you must vendor it.** Copy the sample cowboy Prometheus handler (`otel_cowboy_prometheus_h`, a standard `cowboy_rest` module taking `#{server_name => <reader_name>}` in its route opts) into the component (e.g. `<comp>_web`), point a route at it, and have it call the Prometheus reader's serializer. It is not packaged as a compiled module in any shipped app, so a fresh repo has to bring it in itself. *(Known fork packaging gap — to be promoted to a real module upstream.)*

> [!NOTE]
> Net effect: a component that wants `/metrics` adds the Prometheus reader (with `add_total_suffix => true`) as a second reader **and** vendors the sample cowboy handler. The OTLP reader stays for the OTLP→backend pipeline; the two readers are independent views of the same instruments.
>
> Both the broken inets path and the samples-only cowboy handler are **fork bugs pending an upstream fix**. Once they land, this workaround simplifies — the cowboy handler can be depended on directly instead of vendored, and the inets path becomes usable. Re-verify against the pinned branch and trim this section when that happens.

## 6. These count toward the per-component contract

Library-provided metrics are part of each component's metrics surface, so they belong in its `METRICS.md` and Grafana dashboard ([`observability.md`](observability.md) §4–5) exactly like hand-defined instruments. Document the cowboy/diameter/beam metrics you **actually enable** — do not list instruments you never `init()`, and don't promise `http.route`/`error.type` dimensions you haven't added.
