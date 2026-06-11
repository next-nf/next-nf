# next-nf Network-Function Documentation Standard

This is the house standard for operator-facing documentation of every next-nf network function — the SMF/PGW, the UDR/HSS, the PCF/PCRF, and the CHF: configuration references, interface references, operations runbooks, troubleshooting guides, and diagrams.

It adapts the discipline of the ETSI guide *A Guide to Writing World Class Standards* to operator manuals. ETSI's rules were written for requirements *standards*; this standard keeps their rigour (unambiguous, complete, verifiable statements) while allowing the task-oriented voice that operator documentation needs.

This document is the source of truth. The [`documenting-network-functions` skill](SKILL.md) is its operational summary.

---

## 1. Audience and purpose

Write for an operator or integrator who **did not build the system** and was not involved in any design discussion. State everything clearly and fully; assume no shared context.

Document *what* the operator configures and does, and *how to confirm it worked*. Do not document internal implementation — that belongs in code and API docs. The reader needs to know what to set and what to expect, not how the function is built.

> [!IMPORTANT]
> Misinterpretation caused by a deficient document is a leading cause of incorrect deployment. A single, clear interpretation is the whole job.

## 2. Document types

Each operator document is one of five types, and uses the matching template in [`templates/`](templates/):

| Type | Answers | Template |
| --- | --- | --- |
| Configuration Reference | "What does this parameter do, and what may I set it to?" | `configuration-reference.md` |
| Interface Reference | "What is the contract of this interface?" | `interface-reference.md` |
| Operations Runbook | "How do I perform this task safely?" | `operations-runbook.md` |
| Troubleshooting | "I see symptom X — what now?" | `troubleshooting.md` |
| Diagram | "How do these parts relate / interact?" | `diagram-conventions.md` |

The fixed section order in each template is part of the standard. Do not reorder or omit obligatory sections.

## 3. Normative versus informative

Distinguish the two kinds of text, and keep them visually separate:

- **Normative** — prescriptive. Tells the operator what they must do to configure or operate the system correctly. This is the part that has to be right.
- **Informative** — descriptive. Background, rationale, and context that aid understanding but impose no obligation.

Set informative asides apart with a [GitHub admonition](https://docs.github.com/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax#alerts) where appropriate (they render as callout boxes on GitHub); otherwise use a section titled as informative. Choose the admonition type by intent:

| Admonition | Use for |
| --- | --- |
| `> [!NOTE]` | An informative aside or context. |
| `> [!TIP]` | A recommendation or helpful rationale. |
| `> [!IMPORTANT]` | Key information the operator should not miss. |
| `> [!WARNING]` | A risk — the consequence of getting a setting or step wrong. |
| `> [!CAUTION]` | A hazard — data loss, outage, or security exposure. |

```markdown
> [!NOTE]
> The default binds the listener to loopback, so a fresh checkout exposes no port.
```

Never bury an obligation inside background prose, and never let background dilute an obligation. If informative text seems to carry a requirement, it is in the wrong place — move the requirement into normative text. Admonitions hold informative or cautionary material; they do not carry the primary normative requirements, which belong in the running normative text.

## 4. Verbal forms

Normative statements use exactly these verbs. They are not interchangeable.

| Form | Expresses | Negative form |
| --- | --- | --- |
| **shall** | a mandatory requirement | shall not |
| **should** | a recommendation (expected unless there is a strong reason not to) | should not |
| **may** | permission (allowed, optional) | need not |
| **can** | capability or possibility (a statement of fact, not a requirement) | cannot |

Rules:

- Do **not** use *must*, *has to*, *required to*, or *mandatory* (as a verb) for requirements. They blur the normative/informative line. Use `shall`.
- Do **not** use *can* to grant permission; use `may`. Do not use *can't*; use `cannot`.
- The imperative mood ("Run…", "Set…", "Confirm…") is permitted **only** inside the numbered steps of a runbook procedure, where the operator is being directed through a task. Everywhere else, use the verbal forms above.

Examples (drawn from across the network functions):

- Normative: *The control-plane listener `shall` bind to a reachable address; binding to `127.0.0.1` `shall not` be used when an external peer (MME, AMF, SMF) must connect.*
- Recommendation: *`origin_host` `should` be a fully-qualified domain name in the operator's realm.*
- Permission: *The MongoDB backend `may` be selected in place of the default in-memory backend.*
- Capability: *A single node `can` serve both its Diameter interface and its SBI.*

## 5. Sentences and structure

- Keep each statement as simple as possible. One requirement per sentence; one idea per paragraph.
- Prefer numbered or bulleted lists to long sentences. A list of conditions is clearer than a sentence with three "and"s.
- State the requirement in one place. Do not define a parameter's meaning in one section and its default in another.
- Order topics so earlier text supports later text. Do not refer to a parameter or procedure before it is introduced.

## 6. Terminology and abbreviations

- Define every specialised term and abbreviation **once**, then use it verbatim everywhere. Defined terms live in a Terms section per document, or in a shared glossary the document links to.
- Pick one name for each concept and never vary it. Not "subscriber ID" in one place and "user identity" in another. Not "Diameter peer", "MME", and "client" for the same actor.
- A definition states what a term means; it never contains a requirement.
- Use the established 3GPP names and expand each on first use. The names span the network functions:
  - **Nodes / NFs:** UE, eNB/gNB, MME, SGW, AMF, SMF, PGW, GGSN, UPF, HSS, UDR, UDM, PCRF, PCF, OCS, OFCS, CHF.
  - **Identifiers:** IMSI, MSISDN, IMPI/IMPU, SUPI, GPSI, TEID, Session-Id.
  - **Interfaces / reference points:** GTP-C, GTP-U, PFCP (Sxb/N4), Gx, Gy, Ro, Rf, S6a, Cx, Gi/SGi, and the 5G SBIs (Nudr, Nudm, Npcf, Nchf).
  - **Messages:** name the exact command per interface — e.g. AIR/ULR/PUR/CLR (S6a), CCR/CCA (Gx, Gy), PFCP Session Establishment/Modification/Deletion, and HTTP method + resource (SBI).

## 7. Configuration parameters: completeness and precision

Every configuration parameter is documented completely. A parameter entry **shall** specify all of:

| Field | Meaning |
| --- | --- |
| Name | The exact key, e.g. `origin_host`, and the OTP application it belongs to (e.g. `smf_*`, `udr_diameter`, `pcf_sbi`, `chf_core`). |
| Type | `string`, `integer`, `boolean`, `atom`, `ip4_address`, list, map, etc. |
| Default | The value used when the key is absent. If there is no default, say so explicitly. |
| Allowed values / range | The full set of valid values, or the numeric range and unit. |
| Unit | Where applicable: ms, s, bytes, port number. |
| Description | What the parameter controls (normative where it constrains correct operation). |
| Effect | What changes in observable behaviour when the value changes. |
| Since | The version in which the parameter was introduced or last changed meaning. |

Values are precise. "A large pool", "a short timeout", "responds quickly" are defects — give the number and unit. A requirement that cannot be checked against a value is not verifiable.

## 8. Verifiability

Testability is not optional. Every procedure and every configuration change ends with a **Verify** step whose outcome is **observable**:

- a specific log line,
- an OpenTelemetry span (e.g. `s6a.AIR`, `gx.CCR`, `pfcp.session.establish`) or metric,
- an HTTP status and body from an SBI or provisioning endpoint,
- a Diameter answer (e.g. a CEA in response to a CER, a CCA to a CCR),
- a PFCP response (e.g. a Session Establishment Response with a granted F-TEID),
- a process or port state (`epmd`, a listening socket).

"The node responds" is not verifiable. "A `GET` on `/nudr-dr/v1/subscription-data/{imsi}/provisioned-data/am-data` returns `200 OK` with the AM subscription document" is. If you cannot state an observable outcome, the instruction is not yet complete.

## 9. Conditions and error behaviour

- State pre-conditions **before** the requirement or steps they govern. Put the condition first: *When an external peer must connect, the listener `shall` bind to a routable address.*
- When a condition gates several requirements, make the scope unambiguous. "If COND then REQ1 and REQ2 apply" is ambiguous; write "If COND then **both** REQ1 and REQ2 apply", or split into two statements.
- Specify error and exceptional behaviour as precisely as normal behaviour. Imprecise error handling is a leading cause of interoperability failure. State what the operator observes on failure and what the system does.

## 10. Structure, identifiers, and versioning

- Use the template's section order. It is fixed so that readers and a set of documents stay consistent.
- Give each procedure and each parameter a stable identifier (e.g. `RUN-PROVISION-001`, or the parameter key itself). Identifiers let other documents reference a specific item.
- Do not renumber existing identifiers when revising a document. Add new ones; mark removed ones as withdrawn. Renumbering silently breaks references from other documents.
- Record the document's applicable software version and revision date.

## 11. Diagrams and specialised notation

- Use tables for parameter sets and structured comparisons — they are more precise than prose.
- Use diagrams where a relationship or interaction is hard to express in text. Follow [`templates/diagram-conventions.md`](templates/diagram-conventions.md): deployment diagrams for topology, sequence (message-sequence) diagrams for signalling flows (Attach, AIR/ULR, PFCP session establishment, a Gx/Gy credit-control exchange), state diagrams for lifecycles.
- Author diagrams in **Mermaid** so they render on GitHub and in ExDoc and are diffable in review.
- Mark each diagram normative or informative. Label edges with the interface they carry (GTP-C, PFCP, Gx, Gy, S6a, SBI, Nudr/Nchf). Use the defined abbreviations in labels.

## 12. English and spelling

- Use one variety of English consistently. This project uses **American** spelling. (ETSI's own default is British; the choice matters less than the consistency.)
- Keep the writing impersonal in normative text — no "I", "we", or "you". The imperative steps of a runbook are the one place a direct instruction is used, and even there name the actor where it adds clarity.
- Fix grammar and spelling. Errors do not just risk ambiguity; they undermine trust in the document.

## 13. Quick checklist

Before publishing any operator document:

1. A reader who did not build the system could follow it without guessing.
2. Normative and informative text are separated.
3. `shall` / `should` / `may` / `can` are used correctly; no "must".
4. Every term and abbreviation is defined once and used consistently.
5. Every parameter has name, type, default, allowed values, unit, description, effect, since.
6. Every procedure and config change has an observable **Verify** step.
7. Pre-conditions precede their steps; error behaviour is precise.
8. Section order matches the template; identifiers are stable.
9. Diagrams follow the conventions and are marked normative/informative.
10. Spelling and terminology are consistent throughout.

---

*Adapted from ETSI, "A Guide to Writing World Class Standards." The verbal forms, the normative/informative distinction, the completeness and testability rules, and the stable-numbering guidance are drawn from that guide and from the ETSI Drafting Rules it references.*
