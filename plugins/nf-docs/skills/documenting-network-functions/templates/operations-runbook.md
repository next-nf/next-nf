<!--
TEMPLATE: Operations Runbook
Copy this file, remove the HTML comments, and fill in every field.
Follow ../documentation-style.md. One runbook may hold several procedures;
each procedure uses the full structure below. Imperative voice is allowed ONLY
inside the numbered Steps. Pre-conditions come before steps. Every procedure ends
with an observable Verify step. The worked steps below are illustrative
(subscriber provisioning); replace them with the task you are documenting.
-->

# Operations Runbook: <Area, e.g. Subscriber Provisioning>

**Applies to:** <component> <version> · **Revised:** <YYYY-MM-DD>

## Scope

<One paragraph: which operational tasks this runbook covers and for whom.>

---

## <PROC-ID>: <Procedure title>

<Give each procedure a stable ID, e.g. RUN-PROVISION-001. Do not renumber on revision.>

### Purpose

*(Informative.)* <What this procedure achieves and when an operator runs it.>

### Pre-conditions

*(Normative.)* The following `shall` hold before starting:

- <e.g. The node is running and the provisioning API is reachable on its configured `ip`/`port` (default `127.0.0.1:8090`).>
- <e.g. The IMSI to provision is known and not already provisioned, or overwrite is intended.>

### Inputs and privileges

- <Required inputs, e.g. IMSI, Ki, OPc.>
- <Required access, e.g. network reach to the provisioning port; the API is unauthenticated and `should` therefore be bound to a trusted interface only.>

### Steps

<Numbered, imperative. One action per step. Keep each step checkable.>

1. <Set the variables: `IMSI=001010000000001`.>
2. <Send the create request:>
   ```sh
   curl -sS -X PUT "http://127.0.0.1:8090/provision/v1/subscribers/${IMSI}" \
     -H 'content-type: application/json' \
     -d '{ "ki": "...", "opc": "...", "amf": "8000" }'
   ```
3. <Any follow-up actions.>

### Verify

*(Observable outcome — see ../documentation-style.md §8.)*

- <The `PUT` returns `201 Created` (or `200 OK` on overwrite).>
- <A `GET` on `/provision/v1/subscribers/${IMSI}` returns `200 OK` with the stored profile.>
- <A representative signalling exchange for that subscriber succeeds and emits its OTel span.>

### Rollback / on failure

<What to do if a step fails or Verify does not pass. State what the operator
observes on failure and how to return to the prior state, e.g. `DELETE` the
subscriber. Specify error behaviour precisely.>

### Related

<Links to related procedures, configuration references, or troubleshooting entries.>

---

<!-- Repeat the block above for each procedure. Suggested procedures, by component:
     ALL:  deploy/start the release · stop/restart · upgrade to a new version ·
           back up and restore the data store · switch the data backend ·
           form/verify a cluster.
     SMF/PGW:  bring up a PFCP association with the UPF · drain/quiesce sessions.
     UDR/HSS:  provision a subscriber · delete a subscriber.
     PCF/PCRF: install/update a PCC rule set.
     CHF:      provision a balance · reconcile offline (Rf) records. -->
