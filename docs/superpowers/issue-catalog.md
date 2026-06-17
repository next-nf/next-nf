# Issue catalog — org consistency program (pre-filing review)

**Generated:** 2026-06-17 by per-repo grounding agents (read-only) against the actual repos.
**Gate:** nothing is filed until you approve this list.

## Headline finding: the skill's per-repo tables were stale

The spec catalog inherited the `nf-conventions` skill's per-repo "Action" tables. Grounding
against the live repos shows **chf, pcf, and udr have already completed most of the conventions
migration.** The real remaining work is far smaller than the spec implied and concentrates in
**smf** (the erGW fork) plus program-level items (docs, demo, tests, UI, OSS hygiene). This
strengthens the Task 6 skill refactor — those tables aren't just mutable, they're **wrong** — and
the convention-only rewrite removes them.

### Status matrix (✅ done · ● work remains · – reference/n/a)

| Epic | chf | pcf | smf | udr |
|---|:--:|:--:|:--:|:--:|
| E1 OTP-29 + diameter_make | ✅ | ✅ | ● | ✅ |
| E2 native records | ✅ | ✅ | ● | ✅ |
| E3 data-layer unification | ●¹ | ● | ● | ●² |
| E4 OTEL instruments | ● | ● | ● | – |
| E5 app/module naming | ✅ | ✅ | ● | ✅ |
| E6 JSON binary keys | ✅ | ✅ | ● | ✅ |
| E7 diameter discipline | ✅ | ●³ | ● | ✅ |
| E8 docs parity | ● | ● | ● | #25 |
| E9 CI | ✅ | ● | ● | ✅ |
| E12 minimum UI | ● | ●⁴ | ● | ● |
| E13 OSS hygiene + tag | ● | ● | ●⁵ | ● |

¹ chf: Mongo backend + `chf_cluster` already exist; remaining = CDR outbox + aggregate redesign (fold reservations into balance doc).
² udr: closest to target; remaining = widen 6→11-callback contract, ETS→Mnesia default, schema_version + accessors (this is the reference build-out).
³ pcf: RFC6733 inherited + good discipline; remaining = add `{request_errors, answer}` to the Gx app (small).
⁴ pcf: `pcf_web` already serves dashboard/health/sessions/subscribers; remaining = explicit `/health` + config display (minor).
⁵ smf: LICENSE is **GPLv2** (others are AGPL-3.0) — needs a legal review (spike), not normalization.

**Discarded as wrong:** the chf E5 "rename" the grounding agent proposed was backwards — chf already
has `chf_sbi` (SBI) + `chf_api` (provisioning), which *is* the convention. No chf naming issue.

---

## Conventions epics (E1–E7)

Each epic is a `type:epic` issue in `next-nf` (labels `type:epic` + its `area:*` + `convention`). The
epic body states the convention + the per-repo status from the matrix; children are filed only where
work remains. The skill's matching reference section links here (Task 6).

### E1 — OTP-29 + diameter build toolchain `area:otp29`
Status: chf/pcf/udr done. Child:
- **smf** `type:feature`: *Upgrade smf 26.2 → OTP-29 and switch diameter build to `diameter_make`.* Root + per-app `rebar.config` set `minimum_otp_vsn "29"`; drop `rebar3_diameter_compiler`; replicate `smf_aaa`'s working `provider_hooks {diameter, …}` / gen-dict pattern; all 18 `smf_aaa/src/*.dia` compile clean on 29. Accept: all configs ≥29; plugin removed; dictionaries build; (CI from E9) green on 29.

### E2 — Native records `area:records`
Status: chf/pcf/udr done. Child:
- **smf** `type:feature`: *Migrate ~31 hand-written classic records to native, owned by their modules.* Move 19 records from `smf_core/include/smf.hrl`, 1 from `3gpp.hrl` (`qos`), 2 from `smf/include/smf.hrl` into owning modules; **generated diameter records (862) excluded**; Mnesia row records (if any) stay classic. Accept: `.hrl` record defs removed/relocated; xref clean; suites pass.

### E3 — Data-layer unification `area:db` (sequence udr → pcf → chf → smf)
- **udr** `type:feature` (reference build-out): widen `udr_db_backend` 6→11 callbacks (add `find_by/fold/take/count/create` + index decl), replace `#{set,inc}` with functional `update/3`, make `udr_db_mnesia` the default (ETS retired), add `schema_version` + accessor modules; preserve `syn` per-IMSI lock + CAS. Files: `apps/udr_db/src/udr_db_backend.erl`, `udr_db.erl`, `udr_db_ets.erl`, `udr_db_mongo.erl`, `apps/udr_cluster/src/udr_cluster.erl`.
- **pcf** `type:feature`: domain records (`#subscriber/#pcc_rule/#policy_session`) → documents; promote `msisdn`/`charging_key` indexes; entity callbacks → generic 11-callback contract; `session_update_atomic` → `update/3`; add `pcf_db_mongo` + `pcf_cluster` (neither exists today). Files: `apps/pcf_db/`.
- **chf** `type:feature`: Mongo + `chf_cluster` already exist — remaining is the **CDR outbox** + **aggregate redesign** (fold reservations into the balance doc; descriptive session doc). Files: `apps/chf_db/src/chf_db.erl`, `chf_db_mongo.erl`. Accept: outbox saga implemented + tested; balance aggregate holds reservations.
- **smf** `type:feature` (heaviest, benchmark-gated): route `smf_aaa` raw-ETS (`smf_aaa_session_reg.erl` `ets:lookup/tab2list/new`) through `smf_db`; cursor `select/3` on hot paths; benchmark dirty reads. Full record→document reshape may be a later sub-effort. Files: `apps/smf_core/src/smf_db*.erl`, `apps/smf_aaa/src/smf_aaa_session_reg.erl`.

### E4 — OTEL instruments `area:otel`
Status: udr reference; all three others already on the OTEL framework — remaining is **domain instruments** (+ smf's collector port).
- **chf** `type:feature`: add charging-path instruments (Gy/Ro CCR/CCA latency + grants/denials, Rf ACR/ACA, balance op latency/counts). Files: `apps/chf_diameter/src/chf_diameter_{gy,rf}.erl`, `apps/chf_core/src/chf_{online,offline}.erl`.
- **pcf** `type:feature`: add policy-path instruments (Gx CCR/CCA, session counts). Files: `apps/pcf_diameter/src/pcf_diameter_gx.erl`, `pcf_core` session path.
- **smf** `type:feature` (largest): port custom Prometheus (`smf_prometheus.erl` 60+ decls, `smf_prometheus_collector.erl` PFCP/Sx gauges, `smf_aaa_prometheus_collector.erl`) to OTEL observable instruments; **wire collectors into app startup** (today only registered in test suites); `/metrics` via OTEL exporter. Files: as listed + `smf_core_app.erl`, `apps/smf/src/http_api_handler.erl`.

### E5 — App/module naming `area:naming`
Status: chf/pcf/udr done. Child:
- **smf** `type:feature`: split the top-level `smf` app's HTTP surfaces into `smf_sbi` (`sbi_nbsf_handler` → `smf_sbi_nbsf_h`), `smf_api` (`http_api_handler`/`http_status_handler` → `smf_api_*_h`), `smf_web` (`swagger_ui_handler` → `smf_web_*_h`); rename `_handler` → `_h`; `smf_sbi_client` (outbound) stays. Files: `apps/smf/src/*_handler.erl`, `smf_http_api.erl`.

### E6 — JSON binary keys `area:data`
Status: chf/pcf/udr done. Child:
- **smf** `type:feature`: replace `jsx` 3.1.0 with OTP `json`; keep SBI/provision payloads binary-keyed; atoms only for internal config vocab. Files: `rebar.config`, `apps/smf/src/http_api_handler.erl` (7×), `smf_compat.erl`, test suites.

### E7 — Diameter discipline `area:diameter`
Status: chf/udr done.
- **pcf** `type:feature` (small): add `{request_errors, answer}` to the Gx app (RFC6733 already inherited). File: `apps/pcf_diameter/src/pcf_diameter_gx.erl` / service opts.
- **smf** `type:feature`: add `{request_errors, answer}` to Gx/Ro/Rf/NASREQ (today `{answer_errors, callback}`); confirm RFC6733 App-Id 0 as common. Files: `apps/smf_aaa/src/smf_aaa_{gx,ro,rf,nasreq}.erl`, `smf_aaa_diameter.erl`.

---

## Program epics (E8–E14, E-deploy)

### E8 — Documentation parity `area:docs` (apply the `nf-docs` skill; udr is the bar)
- **chf** `type:feature`: write README (none); add design + manual docs alongside existing `docs/{clustering,configuration}.md`.
- **pcf** `type:feature`: write README (none); expand `docs/` (only `METRICS.md` today) to design + manual.
- **smf** `type:feature`: create `docs/` (none); consolidate scattered `doc/`, `CONFIG.md`, `METRICS.md`, `RADIUS.md`; trim the 557-line README to a landing page.
- **udr**: covered by existing issue **#25** (mint endpoint + op/default_amf docs) — keep/close that, no duplicate.

### E9 — CI `area:ci`
Status: chf/udr done.
- **pcf** `type:chore`: add `.github/workflows` (none); OTP-29; tests + dialyzer; shared template.
- **smf** `type:chore`: add `.github/workflows` (none); OTP-29; build/test all five apps.
- **next-nf** `type:chore` (optional): factor a shared CI template referenced by all repos.

### E10 — Global integration demo `area:demo` (next-nf-led; depends on E15)
- **next-nf** epic + `type:feature` child: compose-based all-NF demo wiring SBI/Diameter reference points; `udr`'s `demo-*` workflows are the per-repo reference.

### E11 — Global integration tests `area:integ-tests` (next-nf-led; depends on E15)
- **next-nf** epic + `type:feature` child: cross-NF integration harness.

### E12 — Minimum UI `area:ui` (health · config display · provisioning)
- **udr** `type:feature`: add `udr_web` (no web app today).
- **smf** `type:feature`: add a web surface (none today) / extend `http_api_handler` with `/health`, `/config`, provisioning.
- **chf** `type:feature`: add `/health` + config display to `chf_web` (dashboard/sessions/CDRs/subscribers already exist).
- **pcf** `type:feature` (minor): add explicit `/health` + config display to `pcf_web` (dashboard already present).

### E13 — OSS hygiene + tagging `area:oss`
- **chf / pcf / udr** `type:chore` (one each): add `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`; cut first semver tag. (All three are AGPL-3.0.)
- **smf** `type:spike` (E13a): **review the GPLv2 license** — obligations, copyleft, dependency compatibility, divergence from the org's AGPL-3.0; recommendation only, no normalization.
- **smf** `type:chore` (E13b): add the three governance files (consistent with the review) + first tag.

### E14 — Release / versioning / compatibility `area:release` (next-nf conceptual)
- **next-nf** epic: define semver-per-repo + inter-NF compatibility matrix; per-repo application follows once defined.

### E-deploy — Deployment guidance `area:deploy` (next-nf conceptual)
- **next-nf** epic: per-component deployment guidance (bare-metal vs container); document/link the bare-metal procedure; capture DPDK/SR-IOV/CPU-pinning needs when concrete (`type:spike`).

---

## Conceptual epics (next-nf only)

- **E15 — Inter-NF interface contracts** `area:integ-tests`: enumerate which reference points are wired between NFs. **Prerequisite for E10/E11.**
- **E16 — Consistency scorecard / definition-of-done**: measurable "as good as udr"; also the canonical `<comp>_core` supervision-tree shape (open convention item).
- **E17 — next-nf forks upkeep**: lifecycle of org-owned forks (OTEL libs, `semantic-conventions`).

---

## Count

~18 epic issues in `next-nf` + ~25 child issues across repos (smf carries ~12 of them). Far below the
spec's ~50 estimate, because conventions work is largely done outside smf.
