---
name: documenting-network-functions
description: Use when writing, reviewing, or updating operator-facing documentation for any next-nf network function (SMF/PGW, UDR/HSS, PCF/PCRF, CHF) — configuration references, interface references, operations runbooks, troubleshooting guides, or diagrams. Applies the org-wide ETSI-derived documentation standard and the matching template.
---

# Documenting next-nf network functions

Operator-facing documentation for every next-nf network function follows a single house standard, adapted from the ETSI guide *A Guide to Writing World Class Standards*. The goal: a reader who did not build the system can configure and operate it correctly, with no room for misinterpretation. The same standard applies whether you are documenting the SMF/PGW, the UDR/HSS, the PCF/PCRF, or the CHF.

**Source of truth:** [`documentation-style.md`](documentation-style.md). Read it before writing or reviewing. This file is the operational summary; the style guide is authoritative.

## Pick the template

Copy the matching template from [`templates/`](templates/) and fill it in. Do not invent structure — the fixed section order is part of the standard.

| You are documenting… | Use |
| --- | --- |
| What a parameter does, its default, and its allowed values | [`templates/configuration-reference.md`](templates/configuration-reference.md) |
| The contract of an interface (GTP-C, PFCP, Gx/Gy/Rf/S6a, an SBI, a provisioning API) | [`templates/interface-reference.md`](templates/interface-reference.md) |
| How to perform an operational task (deploy, provision, back up, upgrade) | [`templates/operations-runbook.md`](templates/operations-runbook.md) |
| A symptom an operator will hit and how to resolve it | [`templates/troubleshooting.md`](templates/troubleshooting.md) |
| A deployment, sequence, or state diagram | [`templates/diagram-conventions.md`](templates/diagram-conventions.md) |

## Authoring checklist

Work through this while drafting:

- [ ] Written for an operator who did not build the system; states *what* to do, not internal implementation.
- [ ] **Normative** text (what the operator must do to operate correctly) is separated from **informative** background; informative asides use a GitHub admonition of the right type (`> [!NOTE]`, `> [!TIP]`, `> [!IMPORTANT]`, `> [!WARNING]`, `> [!CAUTION]` — see the style guide §3).
- [ ] Verbal forms used precisely: `shall` / `should` / `may` / `can` in normative statements. No "must", "has to", "required to". Imperative voice ("Run…", "Set…") only inside numbered procedure steps.
- [ ] One requirement per statement. Lists, not long sentences.
- [ ] Every term and abbreviation is defined once, then used verbatim (IMSI, UE, Diameter, GTP-C, PFCP, S6a, Gx, Gy, SBI…).
- [ ] Every configuration parameter carries: name, app, type, default, allowed values/range, unit, meaning, effect, since-version. Values are precise — never "fast", "large", "soon".
- [ ] Every procedure and every config change ends with a **Verify** step tied to an observable outcome (a log line, an OTel span, an HTTP status, a Diameter answer, a PFCP response).
- [ ] Pre-conditions are stated before the steps. Error and exceptional behaviour is specified precisely.
- [ ] Section order matches the template. Parameter and procedure IDs are stable and not renumbered.
- [ ] Diagrams follow [`templates/diagram-conventions.md`](templates/diagram-conventions.md) and are marked normative or informative.

## Review checklist

When reviewing existing docs, flag any of these:

- Vague quantities ("responds quickly" → give the bound, e.g. "within 30 ms").
- "must" / "you must" / "required to" in normative text → rewrite with `shall`.
- A parameter without a default or without allowed values.
- A procedure with no verification step, or whose verification is not observable.
- An undefined abbreviation, or the same concept named two ways.
- Normative and informative text mixed in one paragraph.
- A condition whose scope is ambiguous ("If COND then REQ1 and REQ2 apply" — which requirements?).

## Before you call it done

A configuration reference is not done until every parameter has a default and a way to verify it took effect. A runbook procedure is not done until it has pre-conditions and an observable verification step. If you cannot write the verification, the documentation is not yet testable — fix that before finishing.
