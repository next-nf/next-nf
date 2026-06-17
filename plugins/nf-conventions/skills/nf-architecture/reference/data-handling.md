# Data handling: JSON keys & single-pass conversion

> Grounded as of 2026-06-11. Two set policies for any code that decodes JSON or transforms data structures (SBI handlers, provisioning APIs, Diameter↔JSON codecs).

## 1. JSON: the OTP `json` module, with binary keys

**Rule (decoder):** use the **OTP built-in `json` module** (`json:decode/1`, `json:encode/1`). Do not add or use third-party JSON libraries — replace `jsx` / `jsone` / `jason` / `jiffy` with `json`. It ships in the OTP base (OTP 27+, so free on the OTP-29 target — [`erlang-otp.md`](erlang-otp.md)), removes a dependency, and `json:decode/1` returns maps with **binary keys** by default, which is exactly what the next rule wants.

**Rule (keys — decoding):** when **decoding** inbound JSON, keep object keys as **binaries**. Do **not** convert decoded keys to atoms — not with `binary_to_atom`, and not with `binary_to_existing_atom` over a seeded atom set. This is where the hazard lives, because inbound keys can be arbitrary identifiers (see below).

> [!NOTE]
> **Encoding is not subject to this rule.** When you *build* a map to hand to `json:encode/1`, using **atom keys — or a mix of atom and binary keys — is fine**; `json:encode/1` renders both as JSON strings. The danger is only in the *decode* direction, where atomizing attacker- or operator-controlled keys causes the failures below. So: produce with whatever key style is convenient (atoms read nicely for fixed response shapes); consume with binaries.

### Why this matters (the mixed-key bug)

3GPP SBIs frequently specify JSON objects as **maps whose keys are arbitrary identifiers** — PCC rule names, session IDs, charging-data references, mapping objects keyed by operator-chosen strings. There is no fixed, closed set of key atoms for these. Converting keys to atoms has two failure modes:

1. **`binary_to_atom` on arbitrary keys** → unbounded atom creation → the atom table fills and the node crashes. A remote peer can trigger this — it is a denial-of-service vector.
2. **`binary_to_existing_atom` with seeded atoms** → the subtle one. You pre-create the atoms you "know about", then convert keys that already exist as atoms. An arbitrary identifier that *happens to coincide* with a seeded atom (or any atom already in the table) becomes an **atom**, while every other key stays a **binary**. You now have **one map with mixed atom and binary keys**. A later `maps:get(<<"foo">>, M)` misses a key stored as `foo`, and vice-versa — silently. This is exactly the hazard the initial "convert all JSON keys to atoms" setup introduced.

### Do this instead

- Decode with `json:decode/1` — it yields binary keys natively, so there is nothing extra to configure and no `{labels, atom}` knob to misuse.
- Reference known keys as **binary literals**: `maps:get(<<"imsi">>, M)`, match `#{<<"imsi">> := Imsi}`.
- When you want a typed structure, fold the binary-keyed map into a record/map (see §2) — do not atomize arbitrary keys to get there.

> [!NOTE]
> A genuinely **closed, internal vocabulary** (e.g. a static config file whose `type`/`handler` values map to a fixed set of module atoms) is a different case from arbitrary SBI payload keys, and a bounded `binary_to_existing_atom` there is defensible. The rule is absolute for **externally-supplied JSON object keys**.

The `udr` reference: OTP `json` decoder, binary keys throughout.

> **Tracking:** per-repo state and migration work is tracked in [next-nf#9](https://github.com/next-nf/next-nf/issues/9) (epic + per-repo children).

## 2. Convert data structures in a single pass

**Rule:** traverse any data structure **once**. The idiomatic "run over it several times" functional style — and the "look up every possible key" style — are both performance traps on the signalling path.

### Anti-pattern A — one `maps:get/3` per known key

Building a record/map by calling `maps:get(Key, Map, Default)` for each of N fields does N independent lookups, and handles unknown keys awkwardly:

```erlang
%% BAD: N lookups, and silently drops/ignores anything not enumerated
to_rec(M) ->
    #foo{a = maps:get(<<"a">>, M, 0),
         b = maps:get(<<"b">>, M, <<>>),
         c = maps:get(<<"c">>, M, false)}.
```

**Do this instead:** initialize the result with its **defaults**, then `maps:fold/3` **once** over the input, dispatching each key to its field:

```erlang
%% GOOD: defaults live in #foo{}; one pass over M
to_rec(M) ->
    maps:fold(
      fun (<<"a">>, V, Acc) -> Acc#foo{a = V};
          (<<"b">>, V, Acc) -> Acc#foo{b = V};
          (<<"c">>, V, Acc) -> Acc#foo{c = V};
          (_K,      _V, Acc) -> Acc          %% ignore unknown keys explicitly
      end, #foo{}, M).
```

The record's own default values are the single source of defaults; the fold visits each present key exactly once.

### Anti-pattern B — chained whole-structure passes

`maps:map/2` then `maps:filter/2` then `maps:to_list/1` (or `lists:map` then `lists:filter` then `lists:foldl`) traverses the structure once **per stage**. Fuse them into a single `maps:fold/3` / `lists:foldl/3` that maps, filters, and accumulates in one pass.

### Why it matters

These conversions run **per request, at signalling rates**. An O(k·N) multi-pass transform or per-field lookup table is a direct latency and throughput cost — the opposite of the deterministic-latency goal behind the bare-metal stance ([`config-deploy-ops.md`](config-deploy-ops.md) §1). Single-pass conversion keeps the hot path flat.

> [!TIP]
> This composes with §1: because keys are binaries, the fold matches binary literals (`<<"a">>`). With OTP-29 **native records** ([`erlang-otp.md`](erlang-otp.md)), `#foo{}` default-init plus in-fold update works identically — keep the record definition as the one place defaults are declared.

## 3. Checklist

- [ ] JSON handled with the OTP `json` module (no `jsx`/`jsone`/`jason`/`jiffy`).
- [ ] JSON decoded with binary keys; no `binary_to_atom`/`binary_to_existing_atom` on external payload keys.
- [ ] Known keys referenced as binary literals (`<<"...">>`).
- [ ] Each data structure traversed once — defaults pre-set, single `maps:fold`/`lists:foldl`.
- [ ] No per-field `maps:get/3` table where a fold would do.
- [ ] No `map`→`filter`→`to_list` chains that re-traverse.
