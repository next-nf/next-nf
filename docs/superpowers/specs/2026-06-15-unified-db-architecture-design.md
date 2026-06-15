# Unified database architecture for the next-nf 3GPP core — design

> **Date:** 2026-06-15 · **Status:** design approved, pending spec review · **Scope:** org-wide data/persistence conventions for `smf`, `udr`, `pcf`, `chf`.

This spec defines a single persistence model for every next-nf network function. It is the **umbrella design**; each NF's migration is its own follow-on spec → plan → implementation cycle. The concrete deliverable that follows from this spec is a `reference/database.md` guide in the `nf-conventions` skill (plus the per-NF migrations).

## 1. Background

A four-repo catalog (see the data-layer investigation) found that all four NFs already share one skeleton — a `<comp>_db` **behaviour** + facade that resolves the backend from app env and caches it in `persistent_term` — but diverge sharply behind it:

- **UDR** (newest, reference): generic document store — 6 typed callbacks (`get`/`put`/`delete`/`find`/`update`-CAS) over collections of binary-keyed **maps**; backends **ETS** (dev) + **MongoDB** (prod); optimistic **CAS via a `version` field**; cluster-wide per-IMSI **`syn` lock**; backend-agnostic conformance suite. No records, no Mnesia.
- **CHF → PCF** (one lineage): domain-specific 14-callback API; **Mnesia `disc_copies`**; **classic records**; `mnesia:activity` transactions with write locks; single-node; no clustering; no migrations. PCF was templated from CHF.
- **SMF** (erGW fork): table-generic `smf_db` (10 callbacks: `insert`/`lookup`/`select`/`take`/`fold`…) over **ETS** (node-local) + **Mnesia `ram_copies`** (cluster-global); classic records; multi-index runtime registries; RAM-only; mostly dirty ops; abstraction only partially followed (`smf_aaa` uses raw ETS).

Common gaps across all four: **no schema migration/versioning**, three incompatible API granularities, records-vs-maps split, inconsistent concurrency and clustering models.

## 2. Decisions (resolved during brainstorming)

1. **Target model = UDR's interface:** document / single-doc CAS / binary-keyed maps, behind the shared `<comp>_db` behaviour.
2. **Backends = Mnesia (ram + disc) + MongoDB; ETS dropped.** Mnesia-ram is the dev/CI backend; Mnesia-disc is a legitimate light-prod tier; Mongo is serious geo-redundant prod.
3. **Atomicity = aggregate-per-document, single-doc CAS only.** No multi-key transactions in the contract; cross-aggregate flows use explicit saga/compensation.
4. **One API for everything** — SMF's hot per-node runtime state also goes through the API (Mnesia-ram), with a documented dirty-read fast path. (Perf to be benchmark-gated for SMF.)
5. **Contract shape = "document store + declared secondary indexes + cursor"** (Approach 2): backend-native indexes make single-doc CAS sufficient for multi-index data.

## 3. Guiding principles

| # | Principle |
|---|---|
| **P1** | **One contract, every NF.** The `<comp>_db` behaviour is the only persistence interface; nothing above the facade touches a backend directly. |
| **P2** | **Documents, not records.** Stored unit = binary-keyed map in a named collection; backends store it in a generic envelope. Classic/native records only for *transient in-module* state. |
| **P3** | **One aggregate = one document.** All atomically-co-mutated state lives in one document; cross-aggregate flows use explicit saga/compensation. |
| **P4** | **Coordination via `syn`, correctness via CAS.** Per-key `syn` lock serializes writers cluster-wide; the `version` field guards against lost updates regardless. |
| **P5** | **Two backends, four tiers.** `<comp>_db_mnesia` (ram = dev/CI/cache; disc = light prod) + `<comp>_db_mongo` (serious geo-redundant prod). Config-selected, `persistent_term`-cached. |
| **P6** | **Backend-native secondary indexes.** Declared per collection, maintained by the backend — no separate index documents, so single-doc CAS stays sufficient. |
| **P7** | **One API for everything, with a documented dirty-read fast path.** Even SMF's hot per-node indices use the API; reads may be lock-free to avoid transaction cost. |
| **P8** | **Backend-agnostic conformance.** One shared conformance suite is the contract's executable spec; every backend must pass it (Mnesia ram/disc + Mongo). |
| **P9** | **Versioned documents.** Every document carries `schema_version`; evolution is additive; reshaping is upgrade-on-read. |

## 4. The `<comp>_db` contract

### 4.1 Types
```erlang
-type collection() :: atom().                  %% subscriber, balance, gtp_context…
-type key()        :: binary().                %% imsi, account_id, session_id…
-type doc()        :: #{binary() => term()}.   %% the document (P2)
-type version()    :: non_neg_integer().       %% CAS token (metadata, not a doc field)
-type selector()   :: #{binary() => term()}.   %% equality match on doc fields
-type index()      :: binary().                %% an indexed doc-field name
-type coll_opts()  :: #{indexes => [index()], storage => ram_copies | disc_copies}.
```

### 4.2 Backend callbacks (each backend `shall` implement)
| Callback | Semantics |
|---|---|
| `child_spec(Opts)` | Start the backend (Mnesia table owner / Mongo conn pool); one child under `<comp>_db_sup`. |
| `ensure_collection(Coll, CollOpts)` | Create collection + declared indexes, idempotent. |
| `get(Coll, Key)` | `{ok, doc(), version()} \| {error, not_found}`. May use a dirty/lock-free read (P7). |
| `put(Coll, Key, Doc)` | Unconditional upsert → `{ok, version()}`. |
| `cas_put(Coll, Key, ExpectedVersion, Doc)` | **Atomicity primitive.** Write iff stored version == ExpectedVersion, bump version → `{ok, version()} \| {error, version_conflict} \| {error, not_found}`. |
| `delete(Coll, Key)` | `ok` (idempotent). |
| `take(Coll, Key)` | Atomic read-and-delete → `{ok, doc(), version()} \| {error, not_found}`. |
| `find(Coll, Selector)` | Equality-selector query → `{ok, [doc()]}`; uses an index if the selector hits one, else scans. |
| `find_by(Coll, Index, Value)` | **Guaranteed indexed read** → `{ok, [doc()]}`; errors if `Index` not declared. Hot-path primitive. |
| `fold(Coll, Selector, Fun, Acc)` | Streaming cursor iteration → `{ok, Acc}`. |
| `count(Coll, Selector)` | `{ok, non_neg_integer()}`. |

### 4.3 Facade helpers (`<comp>_db`, backend-agnostic, written once)
- **`update(Coll, Key, Fun) -> {ok, doc(), version()} | {error, {aborted, R}} | {error, not_found} | {error, max_retries}`** — the **functional CAS** and the primary write primitive. `Fun :: fun((doc()) -> {ok, doc()} | {abort, term()})`. Bounded-retry loop `get → Fun → cas_put`; retries on `version_conflict`; surfaces `{aborted, R}` (business rejection) immediately without retry. Portable across backends (the `Fun` runs in Erlang; only the conditional write is atomic).
- **`create(Coll, Key, Doc) -> {ok, version()} | {error, exists}`** — insert-if-absent.
- The facade resolves the backend from `application:get_env(<comp>_db, backend, …)`, caches it in `persistent_term`, dispatches `(backend()):Op(...)`.

### 4.4 Semantics to lock in
- **Atomicity = `cas_put`/`update` on one document only** (P3); no multi-key/multi-collection atomic op exists.
- **`version` is invisible to domain code** in the common path (reads use `get`; writes use `update/3`). `schema_version` (P9) is a *document field*, distinct from the CAS `version`.
- **Reads are not serializable** w.r.t. concurrent writes except through `update/3`; `get`/`find`/`find_by`/`fold` may be dirty (P7).
- **Indexes are backend-maintained**, declared at `ensure_collection`.
- The declarative `#{set, inc}` mutation form (UDR's current `update/4`) is **not** in the contract; it may return later as an optional backend optimization that is semantically equivalent to a functional `update`.

## 5. Backends & deployment tiers

### 5.1 `<comp>_db_mnesia` (dev / CI / light-prod)
- **Envelope with promoted index columns.** One table per collection. Because Mnesia indexes work on record field positions, declared index fields are **promoted to real columns** at `ensure_collection`:
  `attributes = [key | DeclaredIndexFields] ++ [version, doc]` (keypos = key; Mnesia index on each promoted column). The backend extracts declared fields from the doc on write; `find_by` → `mnesia:index_read`. The doc map stays the source of truth; promoted columns are derived. Single table, atomic, native index machinery, no separate index tables.
- **Op mapping:** `get`/`find`/`find_by`/`count` → dirty ops (lock-free read path, P7); `cas_put`/`take` → `mnesia:transaction` (write-lock, version check, write `version+1`); `fold` → `mnesia:activity(async_dirty, …)`.
- **Durability/clustering** via `storage => ram_copies | disc_copies`: ram = dev/unit/CI (and Erlang-distribution replication for in-VM multi-node tests); disc = light prod (small Erlang cluster *is* the durable replicated DB; new nodes `add_table_copy` + `wait_for_tables`; `cas_put` transaction = distributed ACID).
- **Bootstrap:** `create_schema` (disc only) → `ensure_collection` per collection → `wait_for_tables`.

### 5.2 `<comp>_db_mongo` (serious geo-redundant prod)
- One Mongo collection per logical collection; `_id` = key; `version` a top-level field; declared indexes via `createIndex` at `ensure_collection`.
- `get`→`find_one`; `put`→`replaceOne(upsert)`; `cas_put`→`updateOne({_id, version}, $set doc, $inc version)` with `n=0` disambiguated into `not_found`/`version_conflict` by re-read; `take`→`findOneAndDelete`; `find`/`find_by`→`find`; `fold`→batched cursor; `count`→`countDocuments`.
- **Clustering:** Mongo replica set (optionally sharded) does geo-replication; Erlang nodes are stateless clients (connection cached in `persistent_term`, pooled); `syn` provides per-key coordination across them.
- **Concern knobs** in `backend_opts`: replica-set URI, database, auth, pool size, write/read concern — charging `cas_put` `shall` use `majority` write concern.

### 5.3 Deployment tiers
| Tier | Backend | Storage | Replication | For |
|---|---|---|---|---|
| Dev / unit / CI-fast | `mnesia` | `ram_copies` | single node | local dev, fast CT |
| CI cluster | `mnesia` | `ram_copies` | Erlang multi-node | multi-node CAS/`syn`/failover tests, no infra |
| Light prod | `mnesia` | `disc_copies` | small Erlang cluster | small/single-site operators, durable |
| Serious prod | `mongo` | replica set | Mongo + stateless app | large / geo-redundant |

Selection: `{<comp>_db, backend, …}` + `{<comp>_db, backend_opts, #{…}}`. The Mongo driver ships `{<comp>_db_mongo, load}` (load-not-start, UDR's pattern) so Mnesia-only deployments never pull it at runtime.

**Trade-off accepted:** the generic map envelope forfeits compile-time/Dialyzer typing on stored shapes; mitigated by per-aggregate accessor modules (§6).

## 6. Data model & aggregate discipline

### 6.1 Conventions
- Collection = atom, lowercase singular domain noun; primary key = the natural 3GPP identifier as a binary.
- Reserved fields: `<<"schema_version">>` (P9), conventional `<<"created_at">>`/`<<"updated_at">>`. Binary keys throughout (consistent with `data-handling.md`).

### 6.2 Aggregate-per-document (P3)
An aggregate = the unit of atomic change = exactly one document. Draw boundaries so everything that must change together is one doc; anything across documents is an explicit saga (§7).

### 6.3 Worked example — CHF money
Money collapses into one **balance aggregate** (collection `balance`, key `account_id`) that holds outstanding reservations:
```erlang
#{ <<"schema_version">> => 1,
   <<"account_id">>     => AccountId,
   <<"total">>          => Total,          %% micro-units
   <<"reservations">>   => #{ SessionId => #{<<"rating_group">> => RG, <<"amount">> => Amt} },
   <<"updated_at">>     => Ts }
%% available = total - sum(reservations)   (derived)
```
`reserve`/`commit`/`refund` each become one `update(balance, AccountId, Fun)` whose `Fun` enforces the invariants (`insufficient_balance` abort, commit clamped to held amount, `total_below_reserved` guard) under single-doc CAS. The `charging_session` document becomes **descriptive** (lifecycle, granted/used units for reporting) — never authoritative for money; its updates are independent single-doc CAS and divergence is reconstructable.

### 6.4 Accessor module per aggregate
Each aggregate gets a module (`chf_balance`, `udr_auth`, `smf_gtp_context`, …) that owns the document shape (`from_doc/1`/`to_doc/1`, typed getters, binary field-name literals in one place, defaults), the invariant `Fun`s passed to `update/3`, and `schema_version` + upgrade-on-read. Domain code goes through the accessor, never raw maps. A module may convert to a native record for *transient* manipulation (P2); the persisted form is always the map.

### 6.5 Schema evolution (P9)
- Additive fields: free (accessors default missing fields).
- Reshaping: **upgrade-on-read** in `from_doc` (migrate older `schema_version` in memory; next write persists the new shape) — lazy, zero-downtime, no `transform_table`. A `fold`-based bulk runner is deferred (§9).

## 7. Concurrency, consistency & error handling

### 7.1 Two-layer coordination
- **Layer 1 — `syn` per-key lock (coordination).** Each NF has a `<comp>_cluster` app exposing **`with_entity(Scope, Key, Fun)`** (generalization of UDR's `with_session/2`). Acquires a cluster-wide exclusive lock for one key, runs `Fun`, releases; auto-releases on process death/node-down. Makes the common case single-writer per entity so CAS contention is ~0. It is coordination, not atomicity.
- **Layer 2 — `version` CAS (correctness).** `cas_put` detects lost updates regardless of locking. Lock makes conflicts rare; CAS makes them impossible to lose silently.

**Rules:** wrap a *multi-step procedure* (read-decide-write across steps/IO) in the lock, not a lone write; lock the aggregate's key (a multi-aggregate saga locks the primary entity); keep lock scope tight (no global locks); `update/3` retries `version_conflict` up to N (default 100), `{abort, R}` is a business rejection and is never retried. Used uniformly across tiers (Mnesia `cas_put` is already ACID, but `syn` still covers multi-step procedures and keeps the model identical).

**SMF carve-out:** the cluster lock is for entities with cluster-wide identity (subscriber/account/session). SMF's node-local runtime indices skip Layer 1 (plain `cas_put`/dirty `put`); SMF cluster-wide context-ownership routing is an SMF concern, not a DB convention.

### 7.2 Error taxonomy (every backend normalizes to this)
`{ok, …}` · `{error, not_found}` · `{error, version_conflict}` (raw `cas_put`; `update/3` retries internally) · `{error, {aborted, Reason}}` (business rejection from the `Fun`, never retried) · `{error, max_retries}` · `{error, exists}` (create on existing) · `{error, Reason}` (infra: connection/timeout/system abort). Backends map native errors into this set; the DB layer never swallows infra errors — they bubble up for the caller to map (Diameter result code / SBI 5xx).

### 7.3 Idempotency
`delete` idempotent; `put` upsert; `create` → `{error, exists}`. **Money-moving / non-idempotent operations carry an idempotency token** (e.g. the CCR request id) checked inside the `Fun` against the aggregate, so a retransmit returns the prior result instead of re-applying.

### 7.4 The CDR outbox (the one sanctioned saga; general cross-aggregate template)
1. **Stamp intent atomically:** the balance `update/3` that charges also appends a `pending_cdr` marker into the balance aggregate (same single-doc CAS).
2. **Drain:** a background drainer finds aggregates with pending markers (indexed flag), writes each CDR via `create(cdr, CdrId, …)` (idempotent on `cdr_id`), then clears the marker with a follow-up balance CAS.
3. **Crash-safety:** marker persists in the durable doc until drained; a re-drain's `create` returns `exists` → clears the marker. At-least-once drain + idempotent create = exactly-once effect.

General pattern for any future cross-aggregate need: **stamp intent in the source aggregate (atomic) → drain to the target with an idempotent write → clear on success** — never reach for multi-key transactions.

### 7.5 Read consistency, durability, readiness
- Reads are dirty/lock-free (P7) and may be stale; read-modify-write always goes through `update/3`; read-then-act-externally holds `with_entity`.
- Write/read concern is per call site (charging writes `shall` use Mongo `majority`); Mnesia `disc_copies` durable on commit. Concern is documented at the call site.
- **Readiness gate:** an NF `shall not` accept signaling until `<comp>_db` reports ready (collections + indexes ensured; Mongo connected / Mnesia `wait_for_tables` returned).

## 8. Per-NF mapping, sequencing & testing

### 8.1 Current → target
| NF | Target shift | Effort |
|---|---|---|
| **UDR** | Widen behaviour to the unified contract (add `find_by`/`fold`/`take`/`count`/`create`, index decl; functional `update/3` replacing `#{set,inc}`); swap ETS → `udr_db_mnesia`; add `schema_version` + accessor modules (evolve `udr_data`); `with_session`→`with_entity`. | Low — closest; becomes the reference impl. |
| **PCF** | Records→documents (`subscriber`/`pcc_rule`/`policy_session`, promoted indexes `msisdn`/`charging_key`); entity callbacks→generic contract; `session_update_atomic`→`update/3`; add `pcf_db_mongo`, `pcf_cluster`. | Medium — clean reshape, no charging atomicity. |
| **CHF** | Same reshape **plus the aggregate redesign** (reservations into balance; session descriptive; CDR outbox); balance ops + `session_transaction`→`update/3`; add `chf_db_mongo`, `chf_cluster`(`chf_account`); fix ram-test/disc-prod gap. | High — safety-critical; the real CAS test. |
| **SMF** | Registries→collections with declared indexes; records→documents via accessors on Mnesia-ram + dirty reads; route `smf_aaa`/`smf_core` raw-ETS through `smf_db`; drop erGW artifacts. Benchmark-gated. | Highest — most code, hot-path perf risk. |

### 8.2 Sequencing
**UDR → PCF → CHF → SMF.** Prove the contract + conformance suite + accessor pattern on UDR; validate the record→document reshape on PCF; do the safety-critical aggregate/outbox redesign on CHF once proven; do the heaviest, benchmark-gated SMF last (possibly its own sub-effort). Each NF is its own spec → plan → implementation cycle under this umbrella.

### 8.3 Testing
- **One backend-agnostic conformance suite** (generalize `udr_db_conformance`) is the executable spec; **every backend (mnesia-ram, mnesia-disc, mongo) must pass all scenarios** (CRUD, create-exists, CAS match/stale/not_found, functional `update` abort/retry, `find`, indexed `find_by`, `fold`, `take`, `count`, delete-idempotency).
- Per-NF domain tests run on mnesia-ram; a parity subset runs against Mongo testcontainers.
- **Cluster/failover:** mnesia-ram multi-node CT (in-VM, no infra) for CAS/`syn`/lock-handoff; Mongo replica-set testcontainers for geo-redundancy/failover acceptance.
- **Charging correctness:** concurrency/property tests on balance invariants (extend CHF's N-process no-over-deduction test) + outbox exactly-once under crash injection.
- **SMF perf gate:** documented benchmark of mnesia-ram vs raw-ETS on hot registries before migrating them.
- Replace copy-pasted `setup_mnesia/0` with a shared CT helper.

## 9. Deferred / out of scope (this round)
- `fold`-based bulk migration runner (upgrade-on-read covers the common case).
- SMF cluster-wide context ownership/routing (an SMF concern).
- Native `bag` support in the contract (model via composite keys / list-valued docs).
- An optional declarative `#{set, inc}` mutation fast-path for hot counters (must be equivalent to a functional `update`).

## 10. Next step
This design becomes (a) a `reference/database.md` guide in the `nf-conventions` skill capturing the contract, principles, backends, aggregate discipline, concurrency, and testing as org conventions; and (b) four per-NF migration specs in the recommended sequence. The implementation plan for (a) — and for the UDR-first migration — is produced via the writing-plans skill.
