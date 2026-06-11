# Erlang/OTP Diameter guide

> Grounded as of 2026-06-11. **`udr` is the reference implementation** (mostly correct — its one gap is ENUMs, §4). This guide captures the rules every component's Diameter stack must follow, with the current per-repo state and the fix.

Diameter in next-nf uses **OTP's own `diameter` application** (not a third-party stack). Dictionaries are written as `.dia` files and compiled to Erlang codec modules. The rules below are about wiring that stack correctly so decode errors, result codes, and AVP/ENUM names behave.

## 1. RFC 6733 base dictionary — two separate concerns

There are two different "use RFC 6733" levers, and they are easy to confuse. They do unrelated things; you generally want both, for different reasons.

### 1a. `@inherits` — compile-time only

```
@inherits diameter_gen_base_rfc6733
```

In a `.dia`, `@inherits` is a **dictionary-compilation** directive: it makes the RFC 6733 base AVP definitions available to *that generated codec module*, so your dictionary can reference base AVPs (`Result-Code`, `Origin-Host`, …). It governs **only the generated dictionary**. It has **no effect on how the running `diameter` application behaves** — it does not, by itself, change error handling or which result codes can be answered.

### 1b. Common application — the runtime lever

Register `diameter_gen_base_rfc6733` as the **common application (Application Id 0)** in the service config:

```erlang
{application, [{alias, common},
               {dictionary, diameter_gen_base_rfc6733},
               {module, <comp>_diameter_<iface>}]},
```

The common application is what the stack uses for base-protocol messages (CER/CEA, DWR/DWA, DPR/DPA) and for **answers the stack originates itself**. Combined with `{request_errors, answer}` (§2), this is the lever that lets the stack emit **5xxx**.

**Why RFC 6733 specifically.** Per the OTP `diameter` docs, with `{request_errors, answer}` *"even 5xxx errors are answered without a callback unless the connection ... has configured the RFC 3588 common dictionary"* — and *"`answer` has the same semantics as `answer_3xxx` when the transport ... has been configured with `diameter_gen_base_rfc3588` as its common dictionary"*, because *"RFC 3588 allows only 3xxx result codes in an answer-message."* So with the RFC 3588 default common dictionary, malformed requests are silently only ever answered with 3xxx; the RFC 6733 common dictionary is required for the stack to answer with 5xxx.

**The stack originates these errors — your callbacks do not.** When a request fails to decode, the **diameter application itself** builds and sends the 3xxx/5xxx protocol-error answer using the common-application dictionary. Per-application callback modules (Ro/Rf, S6a, Gx, …) neither construct nor handle these — they only ever see cleanly-decoded requests (§2).

> [!NOTE]
> This corrects an earlier version of this guide, which claimed the application returns a 5xxx answer that the stack rejects as `invalid_return`. That is not the mechanism: the **stack** generates the 5xxx, driven by the common application + `{request_errors, answer}`. The Ro/Rf (and other) callbacks do not produce protocol-error answers.

**Client behavior is unaffected.** The common application governs base-protocol messages and the encoding of stack-originated answers to **inbound** requests. It does not change how this node behaves as a **client** issuing requests (e.g. CCR on Ro/Rf/Gx). Choosing the RFC 6733 common dictionary is a server-side error-answering concern, not a client one.

| Repo | `@inherits` (compile) | RFC 6733 as `common` App-Id 0 (runtime) |
| --- | --- | --- |
| `udr` | ✅ | ✅ (reference — `apps/udr_diameter/src/udr_diameter_srv.erl`) |
| `smf` | ✅ | ⚠️ verify the service registers it as common |
| `pcf` | ✅ (also `-include_lib` the rfc6733 `.hrl`) | ⚠️ verify |
| `chf` | ✅ | ⚠️ verify |

## 2. Set `{request_errors, answer}`

**Rule:** set `{request_errors, answer}` on each Diameter application in the service config:

```erlang
{application, [{alias, s6a},
               {dictionary, diameter_3gpp_s6a},
               {module, udr_diameter_s6a},
               {request_errors, answer}]}
```

**Why.** `request_errors` chooses who answers a request that fails to decode. There are three values:

- `callback` — the half-decoded request is handed to your `handle_request/3`; you reimplement protocol error handling yourself. Avoid.
- `answer_3xxx` (the OTP default) — the **stack** answers automatically, but only with **3xxx** codes.
- `answer` — the **stack** answers automatically and may also use **5xxx** codes (e.g. 5001 `DIAMETER_AVP_UNSUPPORTED`).

Set `answer` so the stack emits the correct permanent-failure codes for malformed requests without a callback, and your callbacks only ever see cleanly-decoded requests. **It works together with §1b:** `answer` only yields 5xxx when the common dictionary is RFC 6733 — with the RFC 3588 default it silently degrades to `answer_3xxx`. So §1b and §2 are jointly required to get stack-generated 5xxx.

| Repo | `{request_errors, answer}` | Action |
| --- | --- | --- |
| `udr` | ✅ set on the `s6a` app | reference |
| `smf` | ❌ uses `{answer_errors, callback}` instead | add `{request_errors, answer}` to each app (Gx/Ro/Rf/NASREQ) |
| `pcf` | ❌ not set | add it to the Gx app |
| `chf` | ❌ not set | add it to the Gy/Ro and Rf apps |

> [!NOTE]
> `answer_errors` and `request_errors` are different knobs. `request_errors` governs **inbound request** decode failures (what this rule is about). `answer_errors` governs how *your client* reacts to a malformed **answer** it receives — independent of the 5xxx server-side concern above.

## 3. Use the generated dictionary — do not hand-redefine AVP/ENUM names

**Rule:** the `.dia` → codec compilation produces the AVP names, ENUM values, record/map definitions, and result-code macros. **Use them.** Do not hand-write `-define(...)` constants that shadow what the dictionary already provides.

- Use map-format messages (`{decode_format, map}`, as `udr` does) or the generated records; reference AVPs by their generated names (`'User-Name'`, `'Visited-PLMN-Id'`, …).
- For result codes and ENUMs, use the generated macros from the dictionary `.hrl` (e.g. `?'DIAMETER_BASE_RESULT-CODE_SUCCESS'`), not a local `-define(DIAMETER_SUCCESS, 2001)`.

| Repo | State | Action |
| --- | --- | --- |
| `udr` | ✅ map format; no hand-redefinitions | reference |
| `smf` | ✅ no hand AVP redefs; gets App-Ids via `?DICT:id()` | fine |
| `pcf` | ✅ uses generated records via `-include_lib` | fine |
| `chf` | ❌ **hand-redefines ENUMs/result codes** | fix |

`chf`'s violations to remove (use the generated dictionary instead):
- `apps/chf_diameter/src/chf_diameter_avp.erl`: `-define(END_USER_E164, 0)`, `-define(END_USER_IMSI, 1)`.
- `apps/chf_diameter/src/chf_diameter_rf.erl`: `-define(ART_EVENT,1)`/`ART_START`/`ART_INTERIM`/`ART_STOP`.
- `apps/chf_diameter/src/chf_diameter_gy.erl`: `-define(DIAMETER_SUCCESS, 2001)` and similar — these duplicate `enum/2` data and macros already in the generated `diameter_3gpp_ts32_299_*` and base modules.

## 4. Define ENUMs properly in the `.dia` files

**Rule:** integer-valued AVPs that have a defined enumeration **shall** be declared as enums in the `.dia`, with `@enum` sections — not as bare `Unsigned32` with magic integers in the code.

This is the one place the reference repo falls short: `udr`'s `diameter_3gpp_s6a.dia` has **no `@enum` section** — AVPs like `Subscriber-Status`, `PDN-Type`, `Cancellation-Type`, `RAT-Type` are `Unsigned32`, and the code uses integer literals with comments (`'Cancellation-Type' => 2  %% SUBSCRIPTION_WITHDRAWAL`). Improve this: add the enum definitions so the symbolic names come from the dictionary (`enum/2`) and §3 can be honored everywhere.

Sketch of the `.dia` form:

```
@avp_types
    Cancellation-Type   1420  Enumerated  MV

@enum Cancellation-Type
    MME_UPDATE_PROCEDURE          0
    SGSN_UPDATE_PROCEDURE         1
    SUBSCRIPTION_WITHDRAWAL       2
    UPDATE_PROCEDURE_IWF          3
    INITIAL_ATTACH_PROCEDURE      4
```

(Use the exact codes/names from the relevant 3GPP TS.)

## 5. Reaching dictionary-only identifiers

Some identifiers exist only as **exports of the generated dictionary module** — there is no other source of truth for them. The generated module exports `name/0`, `id/0`, `vendor_id/0`, `vendor_name/0` (and the codec functions). Call them rather than hard-coding:

- **App-Id** for `{application, ...}` registration or message routing: `<dict>:id()` — e.g. `smf` does `-define(DIAMETER_APP_ID_GX, ?DIAMETER_DICT_GX:id())`. Do this instead of writing `16777238` by hand.
- **Vendor-Id** for AVP construction: `<dict>:vendor_id()` — e.g. `smf_aaa_gx.erl` uses `diameter_3gpp_base:vendor_id()`.
- `name/0` and `vendor_name/0` are likewise available when you need the dictionary's name or `'3GPP'`.

These are normally invoked by the OTP stack internally, but call them explicitly wherever you would otherwise embed a literal App-Id or Vendor-Id.

## 6. Build toolchain: `diameter_make`, not `rebar3_diameter_compiler`

The third-party `rebar3_diameter_compiler` plugin **does not build on OTP-29**. The org standard is OTP's own compiler, driven by a small pre-hook (the `udr` approach):

```erlang
%% apps/<comp>_diameter/rebar.config
{pre_hooks, [
    {compile, "sh gen_dict.sh"},
    {eunit,   "sh apps/<comp>_diameter/gen_dict.sh"},
    {ct,      "sh apps/<comp>_diameter/gen_dict.sh"}
]}.
```

where `gen_dict.sh` calls:

```sh
erl -noshell \
  -eval 'case diameter_make:codec("dia/<dict>.dia", [{outdir,"src"}]) of
             ok -> ok; E -> io:format(standard_error,"~p~n",[E]), halt(1) end' \
  -s init stop
```

(generated `.erl` to `src/`, the `.hrl` moved to `include/`).

| Repo | Toolchain | Action |
| --- | --- | --- |
| `udr` | ✅ `diameter_make` via `gen_dict.sh` | reference |
| `smf` / `pcf` / `chf` | ❌ `rebar3_diameter_compiler` | switch to `diameter_make` during the OTP-29 upgrade ([`erlang-otp.md`](erlang-otp.md) §1) |

> [!TIP]
> Decide whether generated codec modules are checked into `src/` (as `smf`/`pcf`/`chf` do today) or regenerated each build (as `udr` does). The `udr` regenerate-on-build approach keeps generated artifacts out of git; prefer it for consistency.

## 7. Checklist for a Diameter interface

- [ ] `.dia` inherits `diameter_gen_base_rfc6733`; service registers it as `common` (App-Id 0).
- [ ] `{request_errors, answer}` set on every application.
- [ ] No hand-defined AVP/ENUM/result-code constants — all from the generated dictionary.
- [ ] Enumerated AVPs defined with `@enum` in the `.dia`.
- [ ] App-Id / Vendor-Id obtained via `<dict>:id()` / `<dict>:vendor_id()`, never literals.
- [ ] Built with `diameter_make` (OTP-29 compatible), not `rebar3_diameter_compiler`.
- [ ] `{decode_format, map}` (preferred) or generated records used in the callback.
