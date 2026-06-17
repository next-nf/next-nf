# Engineering conventions

> Grounded as of 2026-06-11. General build/release/lint/clustering/ports/licensing conventions. The big topics have their own files: [`erlang-otp.md`](erlang-otp.md) (OTP-29 + native records), [`diameter.md`](diameter.md), [`observability.md`](observability.md), and the app-layout convention in [`architecture.md`](architecture.md).

## 1. Build & release (confirmed)

All components use **rebar3**:

```sh
rebar3 compile      # must produce zero warnings (warnings_as_errors is on)
rebar3 release      # builds _build/default/rel/<comp>/
rebar3 shell        # interactive dev console
```

Run a release in the foreground:

```sh
_build/default/rel/<comp>/bin/<comp> foreground
```

## 1a. Testing: Common Test, not EUnit

**Rule:** write tests as **Common Test** suites (`*_SUITE.erl` under `apps/<app>/test/`). Do **not** use EUnit (`*_tests.erl`). Run with `rebar3 ct`.

This is already the de-facto state — there are zero `*_tests.erl` in the org and tests are Common Test throughout (`smf` 24 suites, `udr` 30, `pcf` 7, `chf` 12). Keep it that way; do not introduce EUnit for new tests.

> [!NOTE]
> Some `<comp>_diameter/rebar.config` files list an `eunit` pre-hook — that only ensures the Diameter dictionary is generated before the `eunit` provider runs ([`diameter.md`](diameter.md) §6); it is not an EUnit test suite and does not contradict this rule.

## 1b. JSON & data structures

Use the OTP `json` module with binary keys; convert data structures in a single pass. **See [`data-handling.md`](data-handling.md).**

## 2. Lint policy (confirmed)

`warnings_as_errors` is enabled org-wide (`{erl_opts, [debug_info, warnings_as_errors]}` in each top-level `rebar.config`). Code **shall** compile clean.

Sanctioned overrides today (both go away with the migrations):
- **Generated Diameter code** is exempt from `warnings_as_errors`. (After moving to `diameter_make` per [`diameter.md`](diameter.md) §6, keep the generated `src/` out of the warnings-as-errors path.)
- **`prometheus` 6.1.2** needs `{override, prometheus, [{erl_opts,[debug_info,nowarn_deprecated_catch]}]}` on OTP-28+. Removed once Prometheus is dropped ([`observability.md`](observability.md)).

Do not extend either exemption to hand-written application code.

## 3. OTP version & records

OTP-29 is the target; prefer native records. **See [`erlang-otp.md`](erlang-otp.md).**

## 4. Diameter

RFC 6733 base, `{request_errors, answer}`, generated dictionaries, `diameter_make` toolchain. **See [`diameter.md`](diameter.md).**

## 5. Observability

OpenTelemetry (not Prometheus); OTEL semantic naming; documented metrics; Grafana per component. **See [`observability.md`](observability.md).**

## 6. Data layer

Access persistent state only through the `<comp>_db` behaviour (see [`architecture.md`](architecture.md) §3). Default backend keeps a fresh checkout runnable (Mnesia in `pcf`/`chf`); external backends (MongoDB in `udr`) are opt-in by configuration. See [`database.md`](database.md) for the unified document/CAS contract, backends, aggregate discipline, and the per-NF migration.

## 7. Clustering

Per-subscriber consistency across nodes via session locking — `udr` uses [`syn`](https://hex.pm/packages/syn) keyed per IMSI, in a dedicated `udr_cluster` app; `smf` has `smf_cluster`. The canonical clustering approach and per-subscriber locking for remaining NFs is tracked in [next-nf#6](https://github.com/next-nf/next-nf/issues/6).

## 8. Listener ports

The canonical port scheme (Diameter / SBI / provisioning API / web UI / a dedicated admin listener for metrics+health+ready) is defined in [`config-deploy-ops.md`](config-deploy-ops.md) §3. Document the real default in each component's Configuration Reference.

## 9. Licensing

Components are **AGPL-3.0** (`smf`, `udr`, `pcf`, `chf` ship a `LICENSE`). New code in those repos inherits it. License for shared assets is tracked in [next-nf#16](https://github.com/next-nf/next-nf/issues/16).

## 10. CI

All CI targets OTP-29.

> **Tracking:** per-repo state and migration work is tracked in [next-nf#12](https://github.com/next-nf/next-nf/issues/12) (epic + per-repo children).

## 11. Documentation

Operator-facing docs follow the org standard — use the `documenting-network-functions` skill (`nf-docs` plugin). Author diagrams in Mermaid.

## 12. Tracked items

Work that was previously listed as open items is now tracked as GitHub issues:

- App / module naming (`<comp>_sbi`/`_api`/`_web`, `pcf_http` name) → [next-nf#8](https://github.com/next-nf/next-nf/issues/8)
- Clustering and per-subscriber locking → [next-nf#6](https://github.com/next-nf/next-nf/issues/6)
- OSS hygiene and license for shared assets → [next-nf#16](https://github.com/next-nf/next-nf/issues/16)
- Supervision-tree shape for `<comp>_core` → [next-nf#19](https://github.com/next-nf/next-nf/issues/19)
