# Erlang/OTP: version target & native records

> Grounded as of 2026-06-11. Set policy: **OTP-29 is the target**, and **native records are preferred** over classic records in hand-written code.

## 1. OTP-29 is the target

| Component | `minimum_otp_vsn` today | Action |
| --- | --- | --- |
| `udr` | **29** | On target — the reference. |
| `pcf` | 28 | Upgrade to 29. |
| `chf` | 28 | Upgrade to 29. |
| `smf` | 26.2 | Upgrade to 29 (largest jump; it is the erGW fork). |

Set `{minimum_otp_vsn, "29"}` in each repo's `rebar.config` and make CI build on OTP-29.

> [!WARNING]
> **The OTP-29 upgrade is coupled to the Diameter build toolchain.** The third-party `rebar3_diameter_compiler` plugin **does not build on OTP-29**. `udr` already moved off it to OTP's own `diameter_make:codec/2` driven by a small `gen_dict.sh` pre-hook. `smf`, `pcf`, and `chf` still use `rebar3_diameter_compiler` and **must** switch to the `diameter_make` approach as part of the OTP-29 upgrade. See [`diameter.md`](diameter.md) §6.

Other OTP-29 friction already seen:
- `prometheus` 6.1.2 trips `warnings_as_errors` on OTP-28+ (deprecated `catch`); repos work around it with `{override, prometheus, [{erl_opts,[...,nowarn_deprecated_catch]}]}`. Moving off Prometheus (see [`observability.md`](observability.md)) removes this entirely.
- `edoc`/`ex_doc` cannot yet parse native-record syntax — see §2.5.

## 2. Native records (OTP-29) — prefer these

**Native records are new in OTP-29. Assume no prior knowledge — read this before writing or changing any record.** They are a distinct runtime datatype, not the classic tuple-with-a-`-record(...)`-macro. Classic records still work; for **new hand-written code, prefer native records**.

### 2.1 Why (the captured-definition property)

When you construct a native record, **its definition is captured into the value itself**. All later operations (match, access, update) use that captured definition, not the one currently in the code. Consequences:

- A record value **survives module reloads and hot code upgrades** — old values keep working even after the module's definition changes. This matters for a telecom system doing rolling cluster upgrades with live sessions.
- Native records are a **distinct, namespaced type** (module + record name are embedded), so `is_record/1` distinguishes them from plain tuples, and there is no field-order-coupling across modules.
- **No shared `.hrl` is needed** — which is also a rule: see §2.4.

### 2.2 Syntax

Definition (note the `#` before the name, and no parentheses/quoting of keyword-like field atoms):

```erlang
-record #Name{Field1 = Default1 :: type1(),
              Field2,
              FieldN = DefaultN}.
```

Defaults must be compile-time-evaluable literals/simple expressions (no variables, no function calls).

Export / import (private to the defining module by default):

```erlang
-export_record([Name1, Name2]).            %% in the defining module
-import_record(other_module, [Name1]).     %% in a consuming module
```

Construct, access, update, match:

```erlang
V  = #Name{Field1 = X},                 %% construct (local)
V2 = #other_module:Name{Field1 = X},    %% construct (external, fully qualified)
A  = V#Name.Field1,                      %% access
V3 = V#Name{Field1 = Y},                 %% update (returns a new value)
#Name{Field1 = P} = V,                   %% match
V#_.Field1                               %% anonymous access (any captured record)
```

Guards: **field access is the only native-record operation allowed in guards.** `is_record/1,2,3` are available (`is_record(T)` is `false` for classic tuple records).

### 2.3 The in-repo reference

`udr` already uses one — copy this style (`apps/udr_crypto/src/udr_crypto.erl`):

```erlang
%% OTP-29 native record: definition is captured into each value, so it survives
%% module reloads across a rolling cluster upgrade. Never define in a .hrl.
-record #eps_av{
    rand  = <<>> :: binary(),
    xres  = <<>> :: binary(),
    autn  = <<>> :: binary(),
    kasme = <<>> :: binary()
}.
-export_record([eps_av]).
```

### 2.4 Rules

- **Never define a native record in a `.hrl`.** The definition travels in the value; a shared header defeats the purpose and is explicitly discouraged. Define it in the module that owns it; `-export_record`/`-import_record` to share.
- **Prefer native records for new hand-written records.** Migrate classic hand-written records opportunistically when touching a module.
- **Do not convert generated Diameter dictionary records.** Those are emitted by `diameter_make` as classic records and are regenerated each build — leave them alone. (The large `-record(` counts in `*_diameter` apps — 826 in `chf`, 128 in `pcf`, ~1700 in `smf_aaa` — are all generated and are *not* migration surface.)
- Keep types: annotate fields with `:: type()` as above.

### 2.5 Tooling caveat

`edoc`/`ex_doc` cannot parse native-record syntax yet. `udr`'s `rebar.config` carries a `source_suffix` workaround so `rebar3 ex_doc` still runs — reuse that approach in any repo that both adopts native records and publishes ExDoc.

> [!IMPORTANT]
> Native records are **experimental** in OTP-29 (and possibly OTP-30); incompatible changes are possible. That is an accepted risk for this project, which targets OTP-29 deliberately. Keep usage idiomatic so a future syntax change is a mechanical fix.

## 3. Hand-written record migration surface (for planning)

Generated diameter records excluded — these are the real hand-written counts:

- `smf`: ~67 (concentrated in `smf_core`, incl. `smf.hrl`/`3gpp.hrl`) — the bulk of the work; many are in `.hrl` files that the native-record rule says to move into owning modules.
- `pcf`: 2 (Cowboy handler `#state{}`).
- `chf`: 2 (Cowboy handler `#state{}`).
- `udr`: 0 hand-written classic records; already uses a native record.
