# SafeNet Ops: Architecture & System Blueprint

![Architecture](https://img.shields.io/badge/Architecture-Zero_Trust-blue)
![Kernel](https://img.shields.io/badge/Kernel-eBPF%2FXDP-orange)
![Control_Plane](https://img.shields.io/badge/Control_Plane-FastAPI-009688)
![UI](https://img.shields.io/badge/UI-Next.js-black)
![Status](https://img.shields.io/badge/Status-Private_Development-lightgrey)

> **Notice:** The core source code for SafeNet Ops is currently housed in a private repository as it is an ongoing final-year Cyber Security engineering capstone and the subject of provisional intellectual property filings. This repository serves as the public-facing architectural blueprint, technical proof-of-work, and system specification.

## Abstract

**SafeNet Ops** is a self-contained, SOHO-grade Zero-Trust Network Access (ZTNA) gateway. It introduces a hybrid defense architecture that combines WireGuard-based `/32` host isolation, eBPF/XDP line-rate anomaly detection, and a graduated, reversible SOAR (Security Orchestration, Automation, and Response) remediation loop. 

Driven by a Continuous Adaptive Risk & Trust Assessment (CARTA) scoring engine, the system autonomously identifies and mathematically isolates threats at the network driver level (Layer 2/Layer 4/Layer 7) before they consume heavy compute resources on the gateway or reach the internal network.

---

## High-Level System Topology

The architecture is strictly separated into three isolated planes to ensure that volumetric attacks against the data plane cannot exhaust the compute resources required for the control or logging planes.

```text
                     ┌───────────────────────────┐
                     │       CONTROL PLANE       │
                     │  FastAPI · SQLite (WAL)   │
                     │  JWT · MFA Captive Portal │
                     └─────────────┬─────────────┘
                                   │ policy push / identity
                     ┌─────────────▼─────────────┐
                     │         DATA PLANE        │
                     │  eBPF/XDP (WAN + LAN)     │
                     │  WireGuard (/32 isolation)│
                     │  Suricata (AF_PACKET IDS) │
                     └─────────────┬─────────────┘
                                   │ map telemetry / alerts
                     ┌─────────────▼─────────────┐
                     │   LOCAL AUTOMATION PLANE  │
                     │  Python CARTA Engine      │
                     │  Dynamic BPF Map Mutator  │
                     └───────────────────────────┘

```

---

## Core Threat Model & eBPF Mitigations

Traditional firewalls rely on OS-level network stacks (like `netfilter`/`iptables`), which allocate memory (`sk_buff`) for every incoming packet. Under volumetric load, this exhausts the gateway's CPU. SafeNet Ops mitigates this by pushing threat remediation down to the eBPF/XDP layer, dropping malicious packets directly inside the NIC driver.

### 1. Layer 4: SYN Flood & State Exhaustion

* **The Threat:** Attackers flood the gateway with TCP SYN packets to exhaust the connection-tracking (conntrack) table.
* **The Architecture:** An XDP program (`xdp_wan.c`) maintains a high-speed, stateless rate-limiting mechanism using bounded BPF maps. It tracks packet velocities per IP. If the velocity exceeds standard threshold heuristics, packets are dropped at line-rate, preserving gateway compute.

### 2. Layer 2: Zero-Trust Identity & ARP Poisoning

* **The Threat:** Internal network mapping and Man-in-the-Middle (MitM) attacks via ARP cache poisoning.
* **The Architecture:** Utilizing pre-seeded BPF maps (`xdp_lan.c`), the gateway verifies cryptographic-like MAC-to-IP bindings instantaneously. Any unauthorized ARP broadcasts or spoofed MAC addresses are dropped before the kernel's networking stack processes them.

### 3. Layer 7: DNS/NTP Covert Tunneling (Payload Exhaustion)

* **The Threat:** Attackers smuggle data out of the network by hiding it within abnormally large DNS (Port 53) or NTP (Port 123) requests, bypassing traditional port-blocking firewalls.
* **The Architecture:** A hybrid Zero-Trust heuristic inspects UDP payloads. Volumetric anomalies (e.g., payloads > 512 bytes on restricted ports exceeding 50 pkts/sec) are discarded, shielding the heavier SOC intrusion detection systems (Suricata) from payload-exhaustion attacks.

---

## The Local Automation Plane (CARTA)

Bridging the high-speed data plane with the analytical SOC plane is a custom Python daemon operating as the Local Automation Plane.

1. **Telemetry Polling:** The daemon continuously reads raw hex telemetry from pinned BPF maps (`/sys/fs/bpf/safenet/rate_limit_map`).
2. **CARTA Evaluation:** It applies Continuous Adaptive Risk and Trust Assessment (CARTA) logic to calculate reputation penalties for specific IPs based on anomalous traffic patterns.
3. **Dynamic Remediation:** If an IP's score falls below the trusted threshold, the daemon constructs a byte-order struct and injects it back into the kernel's `reputation_map`, permanently dropping the attacker's traffic in milliseconds.
4. **SIEM Ingestion:** Formatted JSON alerts are piped out for Wazuh/Suricata ingestion to maintain a permanent audit trail.

---

## Tech Stack & Design Specifications

### Backend / Data Plane

* **Kernel Space:** eBPF / XDP (C)
* **Control Plane API:** FastAPI (Python 3.12)
* **Local Automation:** Python, `bpftool`, PyRoute2
* **Database:** SQLite (configured with WAL mode, foreign keys, and custom busy-timeouts for asynchronous high-velocity reads/writes).
* **Network Infrastructure:** WireGuard, Linux Network Namespaces (`veth` pairs) for localized lab simulation.

### Frontend / SOC Dashboard

* **Framework:** Next.js (App Router), React, Tailwind CSS
* **Design Language:** A custom, ultra-premium SOC dashboard aesthetic blending linear, obsidian-dark canvases (`#050506`) with monospaced data streams (JetBrains Mono) and strict semantic alerts (Emerald for secure states, Crimson for critical drops).
* **Real-Time Visualization:** WebSockets for live threat streaming, React Flow for network topology, Recharts for line-rate packet drop metrics.

---

## Intellectual Property & Novelty Claims

This architecture introduces several novel networking concepts currently being structured for academic publication (IEEE) and provisional patent protection:

1. **Layer 4 Compute Preservation:** In-kernel stateful rate-limiting that bypasses traditional netfilter/conntrack overhead using bounded eBPF maps.
2. **Layer 2 Zero-Trust Identity:** Pre-emptive ARP cache poisoning mitigation utilizing pre-seeded BPF maps for instantaneous MAC-IP binding verification at the driver hook.
3. **Layer 7 Payload Exhaustion Prevention:** Hybrid heuristics for dropping volumetric covert tunneling at line-rate, strictly preserving Suricata's deep-string analysis compute for genuine threats.
4. **Hybrid System Architecture:** A localized, closed-loop system bridging stateless kernel-level packet inspection with stateful user-space CARTA scoring via pinned BPF maps.

---

## Engineering Team

* **Alvin Binoy** – Tech Lead | Kernel / Data Plane & Automation Architecture (eBPF/Python)
* **Shehzad S** – Backend Engineer | Control Plane (FastAPI/SQLite)
* **Abhinav K Ramesh** – Frontend Engineer | UI/UX & Core Dashboard Architecture (Next.js)
* **Elfred Sebastian** – Frontend Engineer | Real-Time Data, WebSockets & Topology (React Flow)
