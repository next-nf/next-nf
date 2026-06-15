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

The common application is what the stack uses for base-protocol messages (CER/CEA, DWR/DWA, DPR/DPA) and for **protocol-error answers the stack sends itself**. Whether the stack sends a **5xxx** answer *directly* — instead of handing it to your callback — depends on this dictionary together with `{request_errors, answer}` (§2).

**Why RFC 6733 specifically.** Per the OTP `diameter` docs, with `{request_errors, answer}` *"even 5xxx errors are answered without a callback unless the connection ... has configured the RFC 3588 common dictionary"* — and *"`answer` has the same semantics as `answer_3xxx` when the transport ... has been configured with `diameter_gen_base_rfc3588` as its common dictionary"*, because *"RFC 3588 allows only 3xxx result codes in an answer-message."* So with an RFC 3588 common dictionary the stack will not send 5xxx **directly**; it **delegates them to the application callback** (§2). The 5xxx is **not lost** — the stack still detects the error and assigns the result code — it just falls to your `handle_request/3` to answer. **RFC 6733 is the OTP-29 default common dictionary**, so the stack already answers 5xxx directly out of the box; registering it explicitly as `common` is intent-making and defensive (it guards against a transport ever being configured with RFC 3588), not the thing that "switches on" 5xxx.

**The stack detects these errors; who *sends* the answer varies.** When a request fails to decode, the **diameter application itself** determines the 3xxx/5xxx protocol error. With RFC 6733 common + `{request_errors, answer}` the stack also **builds and sends** that answer using the common-application dictionary, and per-application callbacks (Ro/Rf, S6a, Gx, …) only ever see cleanly-decoded requests. In any other combination the stack hands a 5xxx to the callback to answer (§2). Prefer the stack-direct path unless an interface specifically needs to shape its own 5xxx answer.

> [!NOTE]
> This corrects an earlier version of this guide, which claimed the application returns a 5xxx answer that the stack rejects as `invalid_return`. That is not the mechanism, and 5xxx are never dropped: the stack **detects** the error and assigns the result code in every case. The common dictionary + `{request_errors, answer}` only decide whether the stack **sends the 5xxx directly** (RFC 6733 common + `answer`) or **hands it to the callback** to answer (every other combination — see §2).

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

**Why.** `request_errors` chooses *who sends the answer* for a request that fails to decode — the stack detects the error and assigns its result code either way. There are three values:

- `callback` — every decode-error request is handed to your `handle_request/3`; you build all protocol-error answers yourself. Avoid.
- `answer_3xxx` (the OTP default) — the **stack** sends **3xxx** answers directly, with no callback; **5xxx-class errors are still passed to your callback** (the stack does not auto-send them).
- `answer` — the **stack** additionally sends **5xxx** answers directly (e.g. 5001 `DIAMETER_AVP_UNSUPPORTED`) — **but only when the common dictionary is RFC 6733**. With an RFC 3588 common dictionary `answer` behaves exactly like `answer_3xxx`, and 5xxx fall back to the callback.

So the stack sends 5xxx **directly** in exactly one configuration: **RFC 6733 common (§1b) + `{request_errors, answer}`**. In every other combination the 5xxx is **delegated to your callback — it is not lost**. Set `answer` (with the RFC 6733 common dictionary, which is the OTP-29 default) so malformed requests are answered by the stack and your callbacks only ever see cleanly-decoded requests. Prefer this stack-direct path unless a specific interface needs to construct its own 5xxx answer.

| Repo | `{request_errors, answer}` | Action |
| --- | --- | --- |
| `udr` | ✅ set on the `s6a` app | reference |
| `smf` | ❌ uses `{answer_errors, callback}` instead | add `{request_errors, answer}` to each app (Gx/Ro/Rf/NASREQ) |
| `pcf` | ❌ not set | add it to the Gx app |
| `chf` | ❌ not set | add it to the Gy/Ro and Rf apps |

> [!NOTE]
> **Grounded (chf, OTP 29).** With `request_errors` left unset — i.e. the `answer_3xxx` default — the stack still **detected** a 5xxx (5001 for an unknown mandatory AVP) but **handed it to the callback**, which returned it as the answer, so the client did receive a 5xxx. This is why a 5xxx was observed without registering `common` or setting `answer`: the answer came from the callback, not the stack. Setting `{request_errors, answer}` (with the RFC 6733 common dictionary) moves that answer onto the stack and keeps the callback handling only clean requests — which is why the convention recommends `answer` even though 5xxx are not "broken" without it.

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
- `apps/chf_diameter/src/chf_diameter_gy.erl`: `-define(DIAMETER_SUCCESS, 2001)` and similar — these duplicate the `-define` macros already in the generated `diameter_3gpp_ts32_299_*` and base `.hrl` files. (Enum/result-code values stay **integers**; use the generated macros, see §4.)

## 4. Define ENUMs properly in the `.dia` files

**Rule:** an AVP that has a defined enumeration **shall** be declared with type `Enumerated` and an `@enum` section in the `.dia` — not as a bare `Unsigned32`/`Integer32` with magic integers scattered through the code.

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

### What `@enum` actually produces (important — values stay integers)

`@enum` does **not** create an atom-valued type or an `enum/2` lookup for application code. The mechanism is the same `.hrl` macro mechanism as §3:

- For each `@enum` value the compiler emits a **`-define` macro** into the generated dictionary `.hrl`, mapping the symbolic name to its integer — e.g. `-define('S6A_RAT-TYPE_EUTRAN', 1004).`, `-define('S6A_CANCELLATION-TYPE_SUBSCRIPTION_WITHDRAWAL', 2).`
- `Enumerated` is **`Integer32` at runtime and on the wire.** With `{decode_format, map}`, decode yields the **integer** (e.g. `1004`); encode **expects the integer**. Passing the symbolic atom to the encoder fails with a `diameter_gen,encode_avps` error.
- Decode does **not** validate against the enum set — any `Integer32` decodes. `@enum` is a source of named constants, **not** a closed whitelist. (A peer sending an un-enumerated value still decodes; it is not rejected.)

So the value of `@enum` is purely that code references the 3GPP value **by macro name instead of a magic integer** — the runtime and wire representation are unchanged. Consume it exactly like §3's result-code macros:

```erlang
-include_lib("diameter_3gpp_s6a.hrl").
...
Msg = #{'Cancellation-Type' => ?'S6A_CANCELLATION-TYPE_SUBSCRIPTION_WITHDRAWAL'}.  %% integer 2 at runtime
```

### Worked reference

`udr` is the worked example: `apps/udr_diameter/dia/diameter_3gpp_s6a.dia` now carries the `@enum` sections (`Subscriber-Status`, `PDN-Type`, `Cancellation-Type`, `RAT-Type`, …), the generated `include/diameter_3gpp_s6a.hrl` holds the `-define` macros, and `udr_diameter_codec.erl` references them by name (still integer 2/1004/… at runtime). A real Open5GS ULR wire capture decodes `RAT-Type` to the integer `1004`, confirming `Enumerated` is integer-valued.

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

- [ ] `.dia` inherits `diameter_gen_base_rfc6733`; service registers it as `common` (App-Id 0) so the stack can send 5xxx **directly** (RFC 6733 is the OTP-29 default common dictionary, but register it explicitly — defensive against an RFC 3588 transport).
- [ ] `{request_errors, answer}` set on every application, so stack-detected 5xxx are answered by the stack rather than delegated to the callback.
- [ ] No hand-defined AVP/ENUM/result-code constants — all from the generated dictionary.
- [ ] Enumerated AVPs declared `Enumerated` with `@enum` in the `.dia`, referenced via the generated `.hrl` macros (values stay integers at runtime/wire).
- [ ] App-Id / Vendor-Id obtained via `<dict>:id()` / `<dict>:vendor_id()`, never literals.
- [ ] Built with `diameter_make` (OTP-29 compatible), not `rebar3_diameter_compiler`.
- [ ] `{decode_format, map}` (preferred) or generated records used in the callback.
