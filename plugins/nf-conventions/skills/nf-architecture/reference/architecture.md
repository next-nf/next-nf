# Architecture: OTP umbrella layout, app naming & shared patterns

> Grounded against the repos as of 2026-06-11. States the **target convention**, the **current state**, and the **migration action**. Where a repo deviates, fix toward the convention — do not copy the deviation.

## 1. Umbrella structure

Every greenfield next-nf component is an **Erlang/OTP umbrella** project: sub-applications under `apps/`, built with `rebar3`, released as a single node. Sub-apps are named `<comp>_<role>`, with a top-level `<comp>` app that aggregates the others (boot logging, release entry point). `chf`/`pcf` are the canonical greenfield shape; `smf` is the erGW fork and differs (see §5).

### Standard sub-app roles

| Role | App | Responsibility |
| --- | --- | --- |
| Aggregator | `<comp>` | Config, startup logging, release entry point. |
| Engine | `<comp>_core` | Session/lifecycle logic, rule/charging/policy evaluation. |
| Data | `<comp>_db` | Pluggable data layer: a behaviour plus one or more backends. |
| Diameter | `<comp>_diameter` | Diameter interface(s) for the 4G/EPC reference points. |
| **SBI** | `<comp>_sbi` | **5G Service-Based Interface (HTTP/2). See §2.** |
| **OAM API** | `<comp>_api` | **Non-SBI APIs: provisioning, OAM. See §2.** |
| **Web** | `<comp>_web` | **Web UI and its backend handlers. See §2.** |
| Cluster | `<comp>_cluster` | Clustering / per-subscriber locking (where present). |

## 2. SBI / API / Web naming convention (target)

This is a **set policy**. Three distinct concerns, three distinct app prefixes, one handler-naming rule.

1. **`<comp>_sbi` — everything 3GPP SBI.** All Service-Based Interface surfaces live here. Each SBI API implementation is a Cowboy handler module named:

   ```
   <comp>_sbi_<apitype>_h.erl
   ```

   e.g. `apps/chf_sbi/src/chf_sbi_offline_h.erl`, `apps/chf_sbi/src/chf_sbi_converged_h.erl`.

2. **`<comp>_api` — non-SBI APIs.** Operations/OAM, provisioning, admin. Anything that is an API but **not** a 3GPP SBI. Handlers follow the same `_h.erl` suffix (`<comp>_api_<resource>_h.erl`).

3. **`<comp>_web` — the Web UI and its backend API handlers.** The human-facing dashboard plus the handlers that serve it.

Rules that apply to all three:

- Handler modules end in **`_h.erl`** (not `_handler.erl`).
- The module prefix matches the app it lives in (`<comp>_sbi_*` inside `<comp>_sbi`, etc.).
- An **outbound** SBI client (a component calling other NFs' SBIs) is a client, not a server surface — `<comp>_sbi_client` is the established name (`smf_sbi_client`) and is fine.

The `udr` reference: `udr_sbi` with `udr_sbi_am_h` / `_auth_h` / `_registration_h`.

> **Tracking:** per-repo state and migration work is tracked in [next-nf#8](https://github.com/next-nf/next-nf/issues/8) (epic + per-repo children).

## 3. Pluggable database pattern

Data access is always behind a **behaviour** in `<comp>_db`, with swappable backends. Core logic depends only on the behaviour, never on a concrete store.

- `pcf`, `chf`: behaviour + **Mnesia** backend.
- `udr`: behaviour (`udr_db`) + **MongoDB** backend (`udr_db_mongo`), with a Nudr-shaped access seam (`udr_data`).
- `smf`: no `smf_db` (erGW manages session state differently).

Rule: a new feature `shall` access persistent state only through the `<comp>_db` behaviour. Adding a backend means implementing the behaviour in a new module/app, not branching core logic.

## 4. Dependency / boot order

Apps start lowest-dependency first (from `pcf`/`chf`):

```
<comp>_db → <comp>_core → <comp>_diameter, <comp>_sbi, <comp>_api, <comp>_web → <comp>
```

DB first, then the engine, then the interface and management apps, then the top-level aggregator last.

## 5. `smf` is the erGW fork

`smf` is not greenfield — it is the forked **erGW** codebase, and its app shape reflects that:

```
apps: smf  smf_aaa  smf_cluster  smf_core  smf_sbi_client
```

- `smf_aaa` holds the Diameter/RADIUS stack (Gx/Ro/Rf/NASREQ) and the generated dictionaries.
- `smf_core` holds the GTP-C/GTP-U/PFCP data-plane logic.
- SBI/OAM/web concerns are collapsed into the top-level `smf` app (see §2 migration).

Evolve `smf` toward a full SMF on its own terms; align it to the naming convention opportunistically, but do not force-fit greenfield structure onto it. Document `smf`'s internal session model in the `smf` repo's own skill and link it here.

## 6. Open items

Tracked in GitHub issues:
- `pcf_http` naming (shared HTTP/JSON library) → [next-nf#8](https://github.com/next-nf/next-nf/issues/8)
- Canonical supervision-tree shape for `<comp>_core` → [next-nf#19](https://github.com/next-nf/next-nf/issues/19)
