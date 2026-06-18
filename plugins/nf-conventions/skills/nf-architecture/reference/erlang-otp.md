# Erlang/OTP: version target, native records & idiomatic style

> Grounded as of 2026-06-11. Set policy: **OTP-29 is the target**, **native records are preferred** over classic records in hand-written code, and §4 records the **idiomatic-style conventions** the org enforces in review.

## 1. OTP-29 is the target

Set `{minimum_otp_vsn, "29"}` in each repo's `rebar.config` and make CI build on OTP-29.

> **Tracking:** per-repo state and migration work is tracked in [next-nf#4](https://github.com/next-nf/next-nf/issues/4) (epic + per-repo children).

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
- **Native records cannot back Mnesia tables — keep Mnesia row records classic.** Verified on OTP 29 (ERTS 17.0.1): a `mnesia:dirty_write/1` of a native record fails with `{aborted, {bad_type, ...}}`, because a native record is a distinct datatype, not the plain tuple that Mnesia's storage, matching, and `record_info` machinery require. This is org-wide, not component-specific: any record used as a Mnesia table row **shall** stay a **classic** record (defined in a shared `.hrl` as usual — the "no `.hrl`" rule above is native-record-specific and does not apply). It affects every Mnesia-backed NF — e.g. `chf` (`#subscriber{}` / `#balance{}` / `#cdr{}` / `#charging_session{}` in `chf_db`) and `pcf`, both of which keep Mnesia as the default backend ([`conventions.md`](conventions.md) §6). The unified data layer sidesteps this entirely by persisting documents in a generic envelope rather than domain records — see [`database.md`](database.md).
- Keep types: annotate fields with `:: type()` as above.

### 2.5 Tooling caveat

`edoc`/`ex_doc` cannot parse native-record syntax yet. `udr`'s `rebar.config` carries a `source_suffix` workaround so `rebar3 ex_doc` still runs — reuse that approach in any repo that both adopts native records and publishes ExDoc.

> [!IMPORTANT]
> Native records are **experimental** in OTP-29 (and possibly OTP-30); incompatible changes are possible. That is an accepted risk for this project, which targets OTP-29 deliberately. Keep usage idiomatic so a future syntax change is a mechanical fix.

## 3. Hand-written record migration surface

Generated diameter records are excluded — they are emitted by `diameter_make` and regenerated each build (the large `-record(` counts in `*_diameter` apps are generated and are not migration surface). The rules in §2.4 govern which hand-written records may and should migrate to native.

> **Tracking:** per-repo state and migration work is tracked in [next-nf#5](https://github.com/next-nf/next-nf/issues/5) (epic + per-repo children).

## 4. Idiomatic code style (enforced in review)

These are general Erlang idioms, not udr-specific. They came out of code review and
apply to hand-written code in every NF. Keep them in mind when writing or changing code;
reviewers will flag deviations.

### 4.1 Match in the clause head, don't `case` on `maps:get`

When a function (or `fun`) dispatches on the **shape of an argument** — especially a map —
pattern-match in the head and provide a fallback clause, rather than fetching with a default
and branching in the body. The required-key path binds with `:=`; the absent path is its own
clause.

```erlang
%% prefer
advance_sqn_fun(N) ->
    fun(#{<<"sqn">> := Sqn} = Doc) -> {ok, Doc#{<<"sqn">> := Sqn + N}};
       (Doc)                       -> {ok, Doc#{<<"sqn">> => N}}
    end.

%% avoid
advance_sqn_fun(N) ->
    fun(Doc) ->
        Sqn = maps:get(<<"sqn">>, Doc, 0),
        {ok, Doc#{<<"sqn">> => Sqn + N}}
    end.
```

This does **not** mean replace every `case` — a `case` on the **result of a call**
(`case udr_db:put(...) of {ok,_} -> ...; {error,_} -> ... end`) is correct and idiomatic,
because you are dispatching on a return value, not on an argument's shape.

### 4.2 Map-update operators: `:=` updates, `=>` inserts

`Map#{K := V}` requires `K` to already be present and **crashes if it is absent** — a useful
assertion. `Map#{K => V}` inserts-or-updates. Use `:=` when the key is known present (e.g. you
just matched it in the head); use `=>` only when the key may be absent. Don't reach for `=>`
everywhere out of habit — `:=` documents and enforces the invariant that the key exists.

### 4.3 Build maps from defaults with `maps:merge`, not field-by-field overlay

To normalise a document to its defaults while preserving fields you don't know about, prefer
`maps:merge(Defaults, Doc)` (where `Doc` wins) over a hand-written overlay of
`Doc#{k1 => maps:get(k1, Doc, d1), ...}`.

```erlang
from_doc(Doc0) ->
    Doc = upgrade(Doc0),
    Defaults = #{<<"schema_version">> => 1, <<"status">> => <<>>, <<"purged">> => false},
    maps:merge(Defaults, Doc).      %% Doc wins; unknown fields pass through; absent ones default
```

**Caveat:** because `merge` lets `Doc` win, any field that must be **forced** to a canonical
value (e.g. re-stamping `schema_version` on upgrade-on-read) has to be normalised on `Doc`
*before* the merge — do it in the `upgrade/1` step, not by putting it in the defaults map (where
a stale stored value would survive). See [`database.md`](database.md) §4.4.

### 4.4 Propagate errors honestly; keep `-spec`s honest

Don't collapse a fallible call with `{ok, V} = Call` when the callee can return `{error, _}`:
that turns a real (infrastructure) error into a `badmatch` crash, and an upper layer often
mis-reports the crash (a storage failure surfacing as an HTTP `400` instead of `500` is the
concrete bug this rule came from). Match both cases and propagate `{error, Reason}`.

Equally, **do not narrow a `-spec` to hide an error the function can actually return.** A spec
of `{ok, version()}` on a function whose body can yield `{error, _}` makes dialyzer *stop*
flagging callers that dropped the error branch — the narrowing hides the bug instead of
surfacing it. Keep the spec as wide as the success typing (`{ok, version()} | {error, term()}`)
so the dropped path is caught. The data-layer error taxonomy is [`database.md`](database.md) §6.1.
