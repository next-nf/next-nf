<h1 align="center">Hi, I'm Nathan 👋</h1>

<p align="center">
  <b>Telco Engineer by day, hobbyist coder by night.</b><br>
  Deep diving into 3GPP and packet core logic. 📍 Berlin
</p>

<p align="center">
  🛠️ <b>Current mission:</b> Learning Erlang to build scalable mobile-core systems.
</p>

---

### About me

I've been building **telco software for ~20 years**. I discovered **Erlang in 2011** and have
reached for it on and off ever since — it was born at Ericsson to run telephone switches, so
using it for 3GPP network functions feels like bringing the workload home.

More recently I came across **erGW**, Travelping's Erlang PGW/GGSN, and was immediately
intrigued — and then saddened to find it seemingly abandoned, no longer publicly maintained.
I wanted to play with it, and to genuinely learn and use Erlang for 3GPP systems. So I forked
it and started building.

Along the way I found that **combining AI-assisted "vibe coding" with deep domain expertise is
a remarkably good match** — the domain knowledge keeps the AI honest, and the AI keeps the
momentum up. **next-nf** is where that combination plays out.

---

### 🚀 The `next-nf` mission

Resurrect and grow a forked PGW into a **full, open-source 3GPP core network in Erlang/OTP** —
spanning the 4G/EPC and 5GC worlds — and surround it with the control-plane and charging
functions a real core needs.

🌐 **Project site:** [next-nf — open-source 3GPP mobile core in Erlang/OTP](https://next-nf.github.io/)

| Component | What it is | Role | Key interfaces |
|---|---|---|---|
| 🧠 **[smf](https://github.com/next-nf/smf)** | GGSN / PDN-GW (forked from erGW), evolving toward a full **SMF** | User-plane session anchor | GTP-C/U, PFCP, Gx, Gy, RADIUS/Diameter Gi/SGi |
| 🗂️ **[udr](https://github.com/next-nf/udr)** | Converged **HSS + UDR/UDM** | Subscriber data | S6a Diameter, Nudr SBI, MILENAGE |
| 📐 **[pcf](https://github.com/next-nf/pcf)** | **PCF + PCRF** | Policy & PCC rules | Gx Diameter, Npcf SBI |
| 💳 **[chf](https://github.com/next-nf/chf)** | **CHF + OCS + OFCS** | Online/offline charging | Gy/Ro, Rf Diameter, Nchf SBI |
| 📦 **[support-containers](https://github.com/next-nf/support-containers)** | Reusable images (e.g. `srsran-4g`) | Interop & demos | podman/buildah → GHCR |

**Where it's headed:**
- 🔧 Maintain and enhance the forked PGW — and grow it into a **full SMF** (combined SMF+PGW).
- 🧩 Build out the supporting functions: **PCRF/PCF**, **OCS/OFCS/CHF**, and converged subscriber data.
- 🧪 Prove it works against the real open-source stack — **srsRAN** UEs and **Open5GS**, with CI-gated interop demos.

---

### 🧰 Toolbox

`Erlang/OTP` · `rebar3` · `DIAMETER` · `GTP-C/U` · `PFCP` · `3GPP SBI (HTTP/2)` · `MILENAGE` ·
`Mnesia` · `OpenTelemetry` · `podman` · `srsRAN` · `Open5GS`

---

### 🤝 Commercial support?

I'm **considering** offering commercial support for these components — maintenance, integration
help, feature work — but I **haven't decided yet**. If that's something you'd find useful, I'd
love to talk it through and see if there's a fit.

📫 Let's have a discussion: **next-nf@proton.me**

---

<p align="center"><i>Putting modern 3GPP network functions back on the runtime that was designed for telecom in the first place.</i></p>
