# Database & persistence: the unified data layer

> Grounded as of 2026-06-15. Set policy: every NF persists through one `<comp>_db` behaviour — a **document store** (binary-keyed maps in named collections) with **single-document CAS**, over **Mnesia (ram/disc) + MongoDB** backends. `udr` is the reference (closest to target); `chf`/`pcf`/`smf` migrate toward it.

## 1. The policy

Nine principles govern every NF's data layer:

- **P1 — One contract, every NF.** The `<comp>_db` behaviour is the only persistence interface; nothing above the facade shall touch a backend directly. See [`architecture.md`](architecture.md) §3.
- **P2 — Documents, not records.** The stored unit is a binary-keyed map in a named collection. Classic or native records are for *transient in-module* state only — never the persisted form. See [`erlang-otp.md`](erlang-otp.md) §2.4 on the Mnesia/native-record constraint.
- **P3 — One aggregate = one document.** All state that must change atomically lives in one document. Cross-aggregate flows use explicit saga/compensation — never multi-key transactions.
- **P4 — Coordination via `syn`, correctness via CAS.** A per-key `syn` lock serializes writers cluster-wide; the `version` field guards against lost updates regardless. Both layers are required.
- **P5 — Two backends, four tiers.** `<comp>_db_mnesia` (ram = dev/CI/cache; disc = light prod) and `<comp>_db_mongo` (geo-redundant prod). Selected by config, cached in `persistent_term`.
- **P6 — Backend-native secondary indexes.** Declared per collection at `ensure_collection`, maintained by the backend. No separate index documents; single-doc CAS remains sufficient.
- **P7 — One API for everything, with a documented dirty-read fast path.** Even SMF's hot per-node registries use the API. Reads may be lock-free to avoid transaction cost; the call site shall document the concern.
- **P8 — Backend-agnostic conformance.** One shared conformance suite is the contract's executable spec. Every backend (Mnesia ram/disc + Mongo) shall pass every scenario.
- **P9 — Versioned documents.** Every document carries `<<"schema_version">>`. Evolution is additive; reshaping is upgrade-on-read. See [`data-handling.md`](data-handling.md) for binary-key conventions.

## 2. The `<comp>_db` contract

### 2.1 Types

```erlang
-type collection() :: atom().                  %% subscriber, balance, gtp_context…
-type key()        :: binary().                %% imsi, account_id, session_id…
-type doc()        :: #{binary() => term()}.   %% the document (P2)
-type version()    :: non_neg_integer().       %% CAS token (metadata, not a doc field)
-type selector()   :: #{binary() => term()}.   %% equality match on doc fields
-type index()      :: binary().                %% an indexed doc-field name
-type coll_opts()  :: #{indexes => [index()], storage => ram_copies | disc_copies}.
```

### 2.2 Backend callbacks

Each backend module shall implement all eleven callbacks:

| Callback | Semantics |
| --- | --- |
| `child_spec(Opts)` | Start the backend (Mnesia table owner / Mongo connection pool); one child under `<comp>_db_sup`. |
| `ensure_collection(Coll, CollOpts)` | Create the collection and declare indexes, idempotent. |
| `get(Coll, Key)` | `{ok, doc(), version()} \| {error, not_found}`. May use a dirty/lock-free read (P7). |
| `put(Coll, Key, Doc)` | Unconditional upsert → `{ok, version()}`. |
| `cas_put(Coll, Key, ExpectedVersion, Doc)` | Write iff stored version == ExpectedVersion, bump version → `{ok, version()} \| {error, version_conflict} \| {error, not_found}`. |
| `delete(Coll, Key)` | `ok`, idempotent. |
| `take(Coll, Key)` | Atomic read-and-delete → `{ok, doc(), version()} \| {error, not_found}`. |
| `find(Coll, Selector)` | Equality-selector query → `{ok, [doc()]}`. Uses an index when the selector hits one, else scans. |
| `find_by(Coll, Index, Value)` | **Guaranteed indexed read** → `{ok, [doc()]}`. Errors if `Index` is not declared. Hot-path primitive. |
| `fold(Coll, Selector, Fun, Acc)` | Streaming cursor iteration → `{ok, Acc}`. |
| `count(Coll, Selector)` | `{ok, non_neg_integer()}`. |

### 2.3 Facade helpers

The `<comp>_db` facade module is written once, backend-agnostic. It resolves the backend from `application:get_env(<comp>_db, backend, …)`, caches it in `persistent_term`, and dispatches `(backend()):Op(…)`. It also provides two higher-level primitives:

- **`update(Coll, Key, Fun) -> {ok, doc(), version()} | {error, {aborted, Reason}} | {error, not_found} | {error, max_retries}`** — the functional CAS and the primary write primitive. `Fun :: fun((doc()) -> {ok, doc()} | {abort, term()})`. Bounded-retry loop: `get → Fun → cas_put`; retries on `version_conflict`; surfaces `{aborted, Reason}` (a business rejection from the `Fun`) immediately without retry. Portable across backends — the `Fun` runs in Erlang; only the conditional write is atomic.
- **`create(Coll, Key, Doc) -> {ok, version()} | {error, exists}`** — insert-if-absent.

> [!IMPORTANT]
> **Contract semantics — read before writing callers:**
> - **Atomicity is single-document only.** `cas_put` and `update/3` are atomic on one document. No multi-key or multi-collection atomic operation exists in the contract; cross-aggregate flows use explicit saga (§6.3).
> - **`version` is invisible to domain code** in the common path. Reads use `get`; writes use `update/3`. `schema_version` (P9) is a *document field* — distinct from the CAS `version` metadata token.
> - **Reads are not serializable** w.r.t. concurrent writes except through `update/3`. The callbacks `get`, `find`, `find_by`, `fold`, and `count` may return stale data (P7); read-modify-write always goes through `update/3`.
> - **Indexes are backend-maintained**, declared at `ensure_collection`. The declarative `#{set, inc}` mutation form is deliberately absent from the contract; it may return later as a semantically-equivalent optimization but is deferred.

## 3. Backends & deployment tiers

### 3.1 `<comp>_db_mnesia`

Mnesia cannot index arbitrary map fields — Mnesia indexes operate on record field positions. The backend resolves this via a **generic envelope with promoted index columns**:

```
attributes = [key | DeclaredIndexFields] ++ [version, doc]
```

`keypos` is `key`. Each declared index field is a real column with a Mnesia index. On write the backend extracts declared field values from the doc and stores them as columns; the doc map remains the source of truth. `find_by` maps to `mnesia:index_read/3`.

Op mapping:

| Operation | Mnesia call | Notes |
| --- | --- | --- |
| `get` / `find` / `find_by` / `count` | dirty ops | Lock-free read path (P7). |
| `cas_put` / `take` | `mnesia:transaction` | Write-lock, version check, write `version+1`. |
| `fold` | `mnesia:activity(async_dirty, …)` | Streaming, no write lock. |

Durability and clustering are controlled by `storage => ram_copies | disc_copies` in `coll_opts`:

- **`ram_copies`** — dev, unit tests, CI. Fast; data is lost on node restart. Erlang-distribution replication gives in-VM multi-node for CAS/`syn`/failover tests with no external infra.
- **`disc_copies`** — light prod. A small Erlang cluster *is* the durable replicated database. New nodes call `add_table_copy` + `wait_for_tables`. A `cas_put` transaction is distributed ACID within the cluster.

Bootstrap sequence: `create_schema` (disc only) → `ensure_collection` per collection → `wait_for_tables`.

See [`erlang-otp.md`](erlang-otp.md) §2.4 on why domain records are not used as Mnesia rows (native records cannot back Mnesia tables; the generic envelope avoids the problem entirely).

### 3.2 `<comp>_db_mongo`

One Mongo collection per logical collection; `_id` = key; `version` a top-level field; declared indexes via `createIndex` at `ensure_collection`.

Op mapping to MongoDB:

| Callback | MongoDB call |
| --- | --- |
| `get` | `findOne` |
| `put` | `replaceOne(upsert: true)` |
| `cas_put` | `updateOne({_id, version}, $set doc + $inc version)`; `n=0` disambiguated into `not_found` vs `version_conflict` by re-read |
| `delete` | `deleteOne` |
| `take` | `findOneAndDelete` |
| `find` / `find_by` | `find` |
| `fold` | batched cursor |
| `count` | `countDocuments` |

Erlang nodes are **stateless clients** (connection pool cached in `persistent_term`). Clustering and geo-replication are delegated to the Mongo replica set. `syn` provides per-key coordination across nodes.

`backend_opts` knobs: replica-set URI, database, auth, pool size, write/read concern. Charging `cas_put` **shall** use `majority` write concern.

The driver ships `{<comp>_db_mongo, load}` (load-not-start) so Mnesia-only deployments never pull it at runtime.

### 3.3 Deployment tiers

| Tier | Backend | Storage | Replication | For |
| --- | --- | --- | --- | --- |
| Dev / unit / CI-fast | `mnesia` | `ram_copies` | single node | local dev, fast CT |
| CI cluster | `mnesia` | `ram_copies` | Erlang multi-node | multi-node CAS / `syn` / failover tests, no infra |
| Light prod | `mnesia` | `disc_copies` | small Erlang cluster | small/single-site operators, durable |
| Serious prod | `mongo` | replica set | Mongo + stateless app nodes | large / geo-redundant |

Selection: `{<comp>_db, backend, <module>}` + `{<comp>_db, backend_opts, #{…}}`.

> [!NOTE]
> The generic map envelope forfeits compile-time/Dialyzer typing on stored document shapes. This is a deliberate trade-off; per-aggregate accessor modules (§4.4) restore static reasoning at the aggregate boundary without requiring Mnesia-typed records.

## 4. Data model & aggregate discipline

### 4.1 Document conventions

- **Collection name:** lowercase singular domain noun atom — `subscriber`, `balance`, `gtp_context`, `pcc_rule`.
- **Primary key:** the natural 3GPP identifier as a binary — `imsi`, `account_id`, `session_id`. See [`data-handling.md`](data-handling.md) for binary-key rules.
- **Reserved fields:** `<<"schema_version">>` (P9); `<<"created_at">>`; `<<"updated_at">>`. All field names are binary literals throughout — consistent with the binary-key policy.

### 4.2 Aggregate-per-document (P3)

An aggregate is the unit of atomic change — exactly one document. Draw boundaries so everything that must change together is one doc. Anything that spans documents is an explicit saga (§6.3). When in doubt, include more in the aggregate; splitting is a later optimization.

### 4.3 Worked example — CHF money aggregate

Money collapses into one **balance aggregate** (collection `balance`, key = `account_id`) that holds outstanding reservations inline:

```erlang
#{ <<"schema_version">> => 1,
   <<"account_id">>     => AccountId,
   <<"total">>          => Total,          %% micro-units
   <<"reservations">>   => #{ SessionId => #{<<"rating_group">> => RG, <<"amount">> => Amt} },
   <<"updated_at">>     => Ts }
%% available = total - sum(reservations)   (derived, never stored)
```

`reserve`, `commit`, and `refund` each become one `update(balance, AccountId, Fun)` whose `Fun` enforces invariants:
- `reserve`: `insufficient_balance` abort if `available < Requested`.
- `commit`: amount clamped to the held reservation.
- `refund`: `total_below_reserved` guard.

The `charging_session` document is **descriptive** (lifecycle, granted/used units for reporting). It is never authoritative for money; its updates are independent single-doc CAS, and divergence is reconstructable from the balance log.

### 4.4 Accessor module per aggregate

Each aggregate gets a dedicated module (`chf_balance`, `udr_auth`, `smf_gtp_context`, …) that owns:

- The document shape: `from_doc/1` / `to_doc/1`, typed getters, binary field-name literals in one place, defaults for missing fields.
- The invariant `Fun`s passed to `update/3`.
- `schema_version` tracking and upgrade-on-read logic.

Domain code shall go through the accessor — never through raw map operations. A module may convert to a native record for *transient* manipulation (P2); the persisted form is always the map.

### 4.5 Schema evolution (P9)

- **Additive fields:** free. Accessor `from_doc/1` defaults missing fields to their zero value.
- **Reshaping:** upgrade-on-read. `from_doc/1` detects older `schema_version` values and migrates the doc in memory; the next `update/3` persists the new shape. Zero-downtime; no `transform_table` call needed.
- **Bulk migration runner** (`fold`-based, for force-migrating all documents): deferred. Upgrade-on-read covers the common case.

## 5. Concurrency & coordination

### 5.1 Two-layer model

The protocol is two layers, always used together:

**Layer 1 — `syn` per-key lock (coordination).** Each NF has a `<comp>_cluster` app exposing `with_entity(Scope, Key, Fun)` (generalization of UDR's `with_session/2`). This acquires a cluster-wide exclusive lock for one key, runs `Fun`, then releases; it auto-releases on process death or node-down. In the common case this makes mutation single-writer per entity, so CAS contention is near zero.

**Layer 2 — `version` CAS (correctness).** `cas_put` detects lost updates regardless of locking. Layer 1 makes conflicts rare; Layer 2 makes them impossible to lose silently.

### 5.2 Rules

- **Wrap the procedure, not a lone write.** Acquire the `syn` lock around a multi-step sequence (read → decide → write, possibly spanning multiple IO steps). A single `put` does not need the lock; `update/3` already retries on `version_conflict`.
- **Lock the aggregate's key.** For a multi-aggregate saga, lock the primary entity's key.
- **Keep lock scope tight.** No global locks; no holding a lock across unrelated operations.
- **`update/3` retries `version_conflict`** up to N times (default 100). `{aborted, Reason}` is a business rejection from the `Fun` and is never retried.
- **Uniform across tiers.** Even though a Mnesia `cas_put` transaction is already ACID, `syn` still covers multi-step procedures so the model is identical across Mnesia and Mongo deployments.

### 5.3 SMF node-local carve-out

The `syn` cluster lock is for entities with **cluster-wide identity** (subscribers, accounts, sessions). SMF's hot per-node runtime indices — which are node-local by design — skip Layer 1 and use plain `cas_put` or dirty `put`. SMF cluster-wide context-ownership routing is an SMF concern and is not a DB-layer convention.

## 6. Consistency & error handling

### 6.1 Error taxonomy

Every backend **shall** normalize its native errors to this set before returning to callers:

| Return value | Meaning |
| --- | --- |
| `{ok, …}` | Success (with doc / version / list / count as appropriate). |
| `{error, not_found}` | No document at (Coll, Key). |
| `{error, version_conflict}` | `cas_put` found a different version than expected. `update/3` retries this internally. |
| `{error, {aborted, Reason}}` | The `Fun` returned `{abort, Reason}`. Never retried — treat as a business rejection. |
| `{error, max_retries}` | `update/3` exhausted its retry budget (default 100). |
| `{error, exists}` | `create` on an already-present key. |
| `{error, Reason}` | Infrastructure error: connection failure, timeout, system abort. |

Backends shall never swallow infrastructure errors. The DB layer surfaces them to the caller, which maps them to a Diameter result code or SBI 5xx as appropriate — see [`diameter.md`](diameter.md) §2 and [`data-handling.md`](data-handling.md).

### 6.2 Idempotency

- `delete` is idempotent — repeated calls return `ok`.
- `put` is an upsert — idempotent on content.
- `create` returns `{error, exists}` on a repeated call — callers use this for at-most-once semantics.
- **Money-moving and other non-idempotent operations** shall carry an idempotency token (e.g. the CCR request-id) checked inside the `Fun` against the aggregate. A retransmit returns the prior result instead of re-applying the operation.

### 6.3 Cross-aggregate flows: the CDR outbox pattern

> [!IMPORTANT]
> **The CDR outbox is the one sanctioned saga and the general template for any future cross-aggregate need.** Never reach for multi-key transactions — use this pattern instead.
>
> 1. **Stamp intent atomically:** the balance `update/3` that charges also appends a `pending_cdr` marker into the balance document (same single-doc CAS).
> 2. **Drain:** a background drainer finds aggregates with pending markers (via an indexed flag), writes each CDR via `create(cdr, CdrId, …)` (idempotent on `cdr_id`), then clears the marker with a follow-up balance `update/3`.
> 3. **Crash-safety:** the marker persists in the durable document until drained. A re-drain's `create` returns `{error, exists}` → the drainer clears the marker. At-least-once drain + idempotent `create` = exactly-once effect.
>
> General form: **stamp intent in the source aggregate (atomic) → drain to the target with an idempotent write → clear on success.**

### 6.4 Read consistency, durability, and readiness

- Reads (`get`, `find`, `find_by`, `fold`) are dirty/lock-free and may be stale (P7). Read-modify-write always goes through `update/3`. Read-then-act-externally should hold `with_entity` for the duration.
- Write/read concern is documented at the call site. Charging `cas_put` **shall** use Mongo `majority` write concern. Mnesia `disc_copies` is durable on commit.
- **Readiness gate:** an NF **shall not** accept signaling traffic until `<comp>_db` reports ready — collections and indexes ensured; Mongo connected, or Mnesia `wait_for_tables` returned.

## 7. Current state & migration

### 7.1 Per-NF status

| NF | Today | Target shift | Effort |
| --- | --- | --- | --- |
| **`udr`** | Generic document store; 6-callback behaviour over ETS (dev) + Mongo (prod); optimistic CAS via `version`; `syn` per-IMSI lock; conformance suite. Closest to target. | Widen behaviour to the unified 11-callback contract (add `find_by`, `fold`, `take`, `count`, `create`, index declaration; functional `update/3` replacing `#{set,inc}`); swap ETS → `udr_db_mnesia`; add `schema_version` + accessor modules; `with_session` → `with_entity`. | Low — becomes the reference implementation. |
| **`pcf`** | Domain-specific callbacks; Mnesia `disc_copies`; classic records; `mnesia:activity` transactions. Templated from CHF. | Records → documents (`subscriber`/`pcc_rule`/`policy_session`, promoted indexes `msisdn`/`charging_key`); entity callbacks → generic contract; `session_update_atomic` → `update/3`; add `pcf_db_mongo`, `pcf_cluster`. | Medium — clean reshape, no charging atomicity. |
| **`chf`** | Same lineage as PCF: domain-specific 14-callback API; Mnesia `disc_copies`; classic records; `session_transaction` + write locks; no clustering; no migrations. | Records → documents **plus aggregate redesign** (reservations into balance doc; session descriptive; CDR outbox); balance ops + `session_transaction` → `update/3`; add `chf_db_mongo`, `chf_cluster`; fix ram-test/disc-prod gap. | High — safety-critical; the real CAS/outbox test. |
| **`smf`** | erGW fork: 10-callback table-generic `smf_db` over ETS (node-local) + Mnesia `ram_copies` (cluster-global); classic records; multi-index runtime registries; mostly dirty ops; `smf_aaa` bypasses `smf_db` with raw ETS. | Registries → collections with declared indexes; records → documents via accessors on Mnesia-ram + dirty reads; route `smf_aaa`/`smf_core` raw-ETS through `smf_db`; drop erGW artifacts. Benchmark-gated. | Highest — most code, hot-path perf risk. |

Cross-references: [`conventions.md`](conventions.md) §6 (data layer policy), [`architecture.md`](architecture.md) §3 (pluggable-DB pattern).

### 7.2 Migration sequencing: UDR → PCF → CHF → SMF

**1. UDR first.** Prove the full 11-callback contract, the conformance suite, and the accessor pattern on the repo that is already closest to target. UDR becomes the org reference implementation.

**2. PCF second.** Validate the record → document reshape on a repo with no charging safety-criticality.

**3. CHF third.** Apply the aggregate redesign and CDR outbox once the pattern is proven on PCF. This is the safety-critical step — test thoroughly before migrating charging in production.

**4. SMF last.** The heaviest migration; hot-path performance must be benchmark-gated against raw-ETS before any registry moves to the API. This may be its own sub-effort.

Each NF migration is its own spec → plan → implementation cycle under this umbrella design. Do not begin a later NF until the earlier one's conformance suite passes on all backends.

## 8. Testing

### 8.1 Backend-agnostic conformance suite

Generalize `udr_db_conformance` into a shared suite that every backend must pass. The scenarios it **shall** cover:

- CRUD: `put`, `get`, `delete` (idempotent), `find` by selector.
- `create` — success on first call, `{error, exists}` on repeat.
- `cas_put` — match (success + version bump), stale version (`version_conflict`), key absent (`not_found`).
- `update/3` — abort path (`Fun` returns `{abort, R}` → `{aborted, R}` returned, no retry); retry path (`version_conflict` → retries to N, succeeds).
- `find_by` — guaranteed indexed read; error on undeclared index.
- `fold` — streaming iteration over all matching documents.
- `take` — atomic read-and-delete; `not_found` on absent key.
- `count` — by selector, including empty result.
- Delete-idempotency: repeated `delete` returns `ok`.

The suite **shall** run against: Mnesia `ram_copies` (always, fast), Mnesia `disc_copies` (CI), Mongo testcontainers (CI parity subset).

### 8.2 Per-NF domain tests

Run against Mnesia `ram_copies`. A parity subset of meaningful domain scenarios also runs against Mongo testcontainers to catch semantic gaps between backends.

### 8.3 Cluster and failover

- **Mnesia-ram multi-node CT** (in-VM, no external infra): CAS contention, `syn` lock handoff, node-down recovery.
- **Mongo replica-set testcontainers**: geo-redundancy and failover acceptance.

### 8.4 Charging correctness (CHF)

- **Concurrency / property tests** on balance invariants: extend CHF's N-process no-over-deduction test to cover `reserve`/`commit`/`refund` interleaving under the new `update/3` model.
- **Outbox exactly-once under crash injection**: kill the drainer at each step; confirm the CDR appears exactly once and the marker is cleared.

### 8.5 SMF performance gate

Benchmark Mnesia-ram vs raw-ETS on the hot registry operations before migrating them through `smf_db`. The benchmark result and the acceptable regression threshold shall be documented and committed before migration begins.

### 8.6 Shared CT helper

Replace the copy-pasted `setup_mnesia/0` found across repos with a shared Common Test helper. New tests shall use it.

## 9. Checklist for a component's data layer

- [ ] All persistent state accessed through `<comp>_db` only — no direct Mnesia / ETS / Mongo calls above the facade.
- [ ] Stored units are binary-keyed maps (documents), not records.
- [ ] Each aggregate fits in one document; cross-aggregate flows use the outbox/saga pattern (§6.3).
- [ ] Secondary indexes declared in `ensure_collection`; never maintained by hand.
- [ ] Writers use `<comp>_cluster:with_entity/3` for multi-step procedures; correctness confirmed by `version` CAS.
- [ ] Error returns normalized to the taxonomy in §6.1; infrastructure errors bubbled to the caller, never swallowed.
- [ ] Backend-agnostic conformance suite passes on Mnesia `ram_copies`, Mnesia `disc_copies`, and Mongo.
- [ ] Readiness gate in place — NF rejects signaling until `<comp>_db` reports ready.
- [ ] Every document carries `<<"schema_version">>` (P9).
- [ ] Per-aggregate accessor module owns `from_doc/1` / `to_doc/1`, invariant `Fun`s, and upgrade-on-read logic.
