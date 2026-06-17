---
name: nf-architecture
description: Use when working on any next-nf component repo (smf, udr, pcf, chf) and you need the org-wide architecture, design rationale, or engineering conventions — the Erlang/OTP umbrella layout, the pluggable-DB pattern, how the network functions interconnect (which NF speaks which 3GPP interface), build/release conventions, or the deployment philosophy. Read this before proposing structural changes or adding a new sub-application.
---

# next-nf architecture & conventions

Shared knowledge for the **next-nf** open-source 3GPP core, built in Erlang/OTP. The org is growing a forked PGW (erGW) into a full SMF and surrounding it with the control-plane and charging functions a real core needs. The components are separate repos but share a deliberate set of patterns — this skill is the source of truth for those patterns so an agent working in any one repo applies them consistently.

> [!IMPORTANT]
> **These are target conventions.** The reference files state the rule and link to the tracking epic; per-repo state and migration actions live in GitHub issues. When you touch a repo, move it toward the convention — do not copy an existing deviation.

Set policies (authoritative — see the reference files for detail):

- **OTP-29 is the org target.** Upgrade every component to it; prefer **native records** over classic records in hand-written code.
- **Uniform app layout** for SBI / OAM / web (`<comp>_sbi` / `<comp>_api` / `<comp>_web`).
- **OpenTelemetry over Prometheus** for all metrics; Grafana dashboards per component.
- **Diameter** follows the RFC 6733 + `request_errors` + generated-dictionary discipline (`udr` is the reference).
- **Common Test only** (no EUnit); **OTP `json` module** with binary keys (not atoms); single-pass data conversion.
- **Unified data layer:** one `<comp>_db` document/CAS behaviour over Mnesia (ram/disc) + MongoDB; aggregate-per-document; `syn`+CAS. (`udr` reference.)
- **Per-repo conventions work is filed as GitHub issues.** This skill states the rule and links the tracking epic; per-repo state and migration actions live in the issues. See [`reference/issue-tracking.md`](reference/issue-tracking.md).

## The components

| Repo | Network function(s) | One-liner |
| --- | --- | --- |
| `smf` | GGSN / PDN-GW → SMF | The forked erGW; user-plane session anchor. The original codebase the org grew from. |
| `udr` | HSS + UDR/UDM | Converged subscriber data; S6a + 5G SBI. |
| `pcf` | PCF + PCRF | Policy and PCC rules; Gx + 5G SBI. |
| `chf` | CHF + OCS + OFCS | Online/offline charging; Gy/Rf + 5G SBI. |

## Reference (load on demand)

Read the file that matches what you are doing — do not load all of them at once.

| You need… | Read |
| --- | --- |
| The OTP umbrella layout, sub-app roles, the **SBI/api/web naming convention**, pluggable-DB pattern, boot order | [`reference/architecture.md`](reference/architecture.md) |
| **OTP-29 target + native records** (a new OTP-29 concept — read before writing records) and the Diameter build toolchain | [`reference/erlang-otp.md`](reference/erlang-otp.md) |
| The **in-depth Diameter guide** (RFC 6733 base, `request_errors`, ENUMs, generated dictionaries) | [`reference/diameter.md`](reference/diameter.md) |
| The **observability policy** (OpenTelemetry, no Prometheus, OTEL semantic naming, Grafana) | [`reference/observability.md`](reference/observability.md) |
| **JSON & data handling** (OTP `json` module, binary keys not atoms, single-pass conversion) | [`reference/data-handling.md`](reference/data-handling.md) |
| The **unified data layer** (`<comp>_db` document/CAS contract, Mnesia+Mongo backends, aggregate discipline, migration) | [`reference/database.md`](reference/database.md) |
| Which NF speaks which 3GPP interface, and how they interconnect | [`reference/interfaces-map.md`](reference/interfaces-map.md) |
| Build/release commands, lint policy, clustering, ports, licensing | [`reference/conventions.md`](reference/conventions.md) |
| Why the core runs on bare metal rather than Kubernetes | [`reference/deployment-philosophy.md`](reference/deployment-philosophy.md) |
| How to file, route, and label GitHub issues for conventions work | [`reference/issue-tracking.md`](reference/issue-tracking.md) |

## How this fits with the other skills

- For **writing operator documentation**, use the `documenting-network-functions` skill (the `nf-docs` plugin) — that is the org-wide doc standard. This skill is about how the systems are *built*; that one is about how they are *documented*.
- **Component-specific** design (e.g. erGW's internal session model in `smf`) belongs in that component's own repo as its own skill, not here. Keep this skill org-wide.

## When proposing structural change

Before adding a sub-application, renaming an app, or introducing a new dependency:

1. Check [`reference/architecture.md`](reference/architecture.md) for the established naming and role of sub-apps — match it, do not invent a new shape.
2. Keep the database access behind the pluggable DB behaviour; do not couple core logic to a specific backend.
3. Preserve `warnings_as_errors` — code `shall` compile clean (see [`reference/conventions.md`](reference/conventions.md)).
4. If the change crosses components (a new shared interface or pattern), it belongs in this skill — update the relevant reference file in the same change.
