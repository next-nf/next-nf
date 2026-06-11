<!--
TEMPLATE: Configuration Reference
Copy this file, remove the HTML comments, and fill in every field.
Follow ../documentation-style.md. Do not reorder or omit the numbered sections.
One Configuration Reference documents one subsystem (one OTP application).
The worked entries below are illustrative (an S6a Diameter listener); replace them
with the parameters of the application you are documenting.
-->

# Configuration Reference: <Subsystem / OTP application>

**Applies to:** <component> <version> · **Revised:** <YYYY-MM-DD>

## 1. Scope

<One paragraph: which application this covers (e.g. `smf_*`, `udr_diameter`,
`pcf_sbi`, `chf_core`), and what it configures. State what is out of scope and
link to the related reference.>

## 2. Terms

<Define any abbreviation or term used below that is not in the shared glossary.
A definition states meaning only — no requirements. Example:>

- **Origin-Host** — the Diameter identity (DiameterIdentity AVP) this node presents to peers.

## 3. Where configuration lives

<State the file(s) and how they are loaded. For this project, runtime config is in
`config/sys.config` under the application key, applied at boot. Show the block:>

```erlang
{udr_diameter, [
  {origin_host,  "hss.epc.mnc001.mcc001.3gppnetwork.org"},
  {origin_realm, "epc.mnc001.mcc001.3gppnetwork.org"},
  {listen, [{tcp, {127,0,0,1}, 3868}]}
]}
```

## 4. Parameter reference

<One row per parameter. Every column is mandatory. Defaults and allowed values are
not optional — see ../documentation-style.md §7. "Since" is the version the
parameter was introduced or last changed meaning.>

| Parameter | Type | Default | Allowed values | Unit | Description | Effect | Since |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `origin_host` | string | *none (required)* | a DiameterIdentity (FQDN) | — | Identity this node presents to Diameter peers. | Peers match this against their configured Destination/Origin host. | 0.1.0 |
| `origin_realm` | string | *none (required)* | a Diameter realm (FQDN) | — | Realm this node belongs to. | Used in realm-based routing. | 0.1.0 |
| `listen` | list of `{tcp, IpV4, Port}` | `[{tcp,{127,0,0,1},3868}]` | one or more listener tuples | port | Transport endpoints the listener binds. | Determines which addresses a peer can connect to. | 0.1.0 |

## 5. Parameter detail

<For each parameter whose correct use carries constraints, add a subsection.
Use shall/should/may/can. Set background apart with an admonition of the right type — see ../documentation-style.md §3.>

### 5.1 `listen`

`listen` is a list of transport endpoints. Each entry is `{tcp, IpV4Address, Port}`.

- For a peer on a separate host, `listen` `shall` include a routable address; an entry bound to `{127,0,0,1}` `shall not` be the only entry.
- The standard Diameter port is `3868`; a non-standard port `may` be used where the peer is configured to match.

> [!NOTE]
> The default binds to loopback so a fresh checkout is safe by default and exposes no port to the network.

## 6. Example

<A complete, valid configuration block for a realistic deployment, with a one-line
note on what it achieves.>

```erlang
{udr_diameter, [
  {origin_host,  "hss01.epc.mnc001.mcc001.3gppnetwork.org"},
  {origin_realm, "epc.mnc001.mcc001.3gppnetwork.org"},
  {listen, [{tcp, {10,0,0,5}, 3868}]}
]}
```
<This binds the listener on 10.0.0.5:3868 so an external peer in the same realm can connect.>

## 7. Verify

<How to confirm the configuration loaded and took effect — observable outcomes only
(see ../documentation-style.md §8). Example:>

- From the node, confirm the listener is bound: `ss -ltn '( sport = :3868 )'` shows a socket on the configured address.
- A peer's CER `shall` be answered with a CEA carrying the configured `origin_host`.
- A representative request against the interface produces the matching OpenTelemetry span (e.g. `s6a.AIR`, `gx.CCR`).
