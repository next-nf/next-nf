# Deployment philosophy: the case for bare metal

> Status: scaffold. This captures the stance articulated in the `smf` repo. TODO: confirm how broadly it applies — it is strongest for the user-plane/signalling core (`smf`); the control-plane and data NFs (`pcf`, `chf`, `udr`) may tolerate orchestration better.

## Position

The core packet-processing and signalling engine runs on **bare metal**, not Kubernetes. The architecture prioritizes **deterministic latency** and **hardware-line-rate throughput** — two metrics frequently compromised by the abstraction tax of orchestrators. A network function is not a microservice; it is a high-performance network element.

## Why K8s is discouraged for the core

- **Hardware affinity & the NUMA gap.** High-capacity user planes need a strict 1:1 mapping between NIC queues, memory channels, and CPU cores. The K8s scheduler is designed to *move* workloads — antithetical to the static, pinned environment that DPDK/SR-IOV stability requires.
- **The networking "death spiral".** Telco protocols like SCTP and GTP-U need stable identity and multi-homing. In K8s, a minor liveness-probe failure or CNI hiccup can evict a pod and drop hundreds of thousands of active subscriber sessions in a "self-healing" loop that causes more downtime than it prevents.
- **Troubleshooting opacity.** When milliseconds matter, you cannot afford to peel back Multus, sidecars, and overlay tunnels to find a bottleneck. On bare metal the path from the wire to the process is direct and observable.

## Why Erlang/OTP fits this stance

Erlang was created at Ericsson to run telephone switches: soft real-time, massively concurrent, and built to stay up through faults and hot upgrades. The runtime provides the supervision, fault isolation, and live-upgrade story that operators would otherwise reach to an orchestrator for — so the core gets resilience *without* the orchestration tax.

## Scope and nuance

> [!NOTE]
> "No K8s" is a stance about the **core data and signalling plane**, not a blanket ban. Build tooling, interop demos, and supporting images already use containers (`support-containers` builds with podman/buildah). The question for each NF is whether it sits on the latency-critical path.

## TODO for the maintainer

- [ ] State per-component deployment guidance (bare metal vs container-tolerant) for `pcf`, `chf`, `udr`.
- [ ] Document the supported bare-metal deployment procedure (or link the `smf` deployment runbook).
- [ ] Note any DPDK/SR-IOV/CPU-pinning requirements once they are concrete.
