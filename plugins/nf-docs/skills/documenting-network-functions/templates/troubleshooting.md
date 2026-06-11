<!--
TEMPLATE: Troubleshooting guide
Copy this file, remove the HTML comments, and fill in every field.
Follow ../documentation-style.md. One entry per symptom. Order likely causes
most-probable first. Diagnosis steps state the command AND the expected vs actual
observation. Resolution is normative (shall/should/may). The worked entry below is
illustrative (a Diameter peer that will not connect); replace it with your symptoms.
-->

# Troubleshooting: <Subsystem or area>

**Applies to:** <component> <version> · **Revised:** <YYYY-MM-DD>

## Scope

<One paragraph: which class of problems this guide covers.>

---

## <TS-ID>: <Symptom as the operator observes it>

<Give each entry a stable ID, e.g. TS-DIAM-001. Title the entry by the observable
symptom, not the cause — that is what the operator searches for.>

### Symptom

<What the operator sees. Be concrete: an exact error, a missing response, a log line.
e.g. "A peer's CER is not answered; no CEA is returned and the peer stays in the
closed state.">

### Affected component

<Which app/interface, e.g. the Diameter listener.>

### Likely causes

*(Ordered, most probable first.)*

1. <e.g. The `listen` address is bound to loopback and is unreachable from the peer host.>
2. <e.g. `origin_realm` does not match the realm the peer expects.>
3. <e.g. The configured port is firewalled.>

### Diagnosis

<For each candidate cause: the command to run, the expected observation, and what a
failing observation looks like.>

1. Confirm the listener address:
   ```sh
   ss -ltn '( sport = :3868 )'
   ```
   - Expected: a socket on the routable address from the app's `listen`.
   - If it shows `127.0.0.1:3868`, the listener is unreachable externally — see cause 1.
2. <Next diagnosis step…>

### Resolution

*(Normative.)*

- For cause 1: `listen` `shall` be set to a routable address (see the relevant Configuration Reference §5.1) and the release restarted.
- For cause 2: `origin_realm` `shall` match the peer's expected realm.

### Prevention

*(Informative.)* <How to avoid recurrence — e.g. validate `listen` against the
deployment network in the deploy runbook's Verify step.>

### Related

<Links to the relevant Configuration Reference, runbook procedure, or other entries.>

---

<!-- Repeat for each symptom. Candidate entries, by component:
     ALL:      release will not boot · data backend unreachable at boot ·
               OTel spans/metrics not arriving at the collector · cluster split.
     SMF/PGW:  PFCP association will not establish · GTP-U bearer not forwarding ·
               session setup rejected.
     UDR/HSS:  S6a peer will not connect (no CEA) · AIR returns no vectors (IMSI
               not provisioned) · ULR rejected (unknown subscriber) · SBI GET 404.
     PCF/PCRF: Gx CCR-I rejected · PCC rule not installed at the gateway.
     CHF:      Gy CCR returns no granted units · Rf record not written. -->
