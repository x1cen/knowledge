# Network Knowledge Base

A collection of **CCNP-level Cisco networking notes**, built topic by topic. Each topic lives in its own folder with a complete, self-contained note (`README.md`) covering the theory, design rules, and step-by-step configuration — and topics with hands-on practice also include **EVE-NG/UNetLab lab files** (`.unl` topology) with a PDF lab guide.

## Topics

| Topic | Focus | Labs |
| :--- | :--- | :--- |
| **[FHRP](Networking/FHRP/README.md)** — First-Hop Redundancy Protocol | Default-gateway redundancy with **HSRP**, **VRRP**, and **GLBP**: virtual IP/MAC, elections, timers, and load balancing | ✅ 5 labs (HSRP, VRRP, GLBP) |
| **[IP SLA](Networking/IP%20SLA/README.md)** — IP Service Level Agreement | Active probes to measure reachability, latency, and jitter — the sensor half of automated failover | — |
| **[Track Object](Networking/Track%20Object/README.md)** — Cisco Object Tracking | Tracking objects that bind to IP SLA results or interface state and drive conditional routing decisions | — |
| **[Static Route](Networking/Static%20Route/README.md)** | Static routing fundamentals: recursive vs. directly attached routes, floating statics, and verification | — |
| **[VRF](Networking/VRF/README.md)** — Virtual Routing and Forwarding | Splitting one router into multiple isolated routing tables | — |
| **[Private VLAN](Networking/Private%20VLAN/README.md)** — PVLAN | Layer-2 host isolation inside a single VLAN: primary, isolated, and community sub-VLANs | — |
| **[VACL](Networking/VACL/README.md)** — VLAN Access Map | Filtering traffic **inside** a VLAN at Layer 2 — where Router ACLs can't reach | — |

> [Note] **How the topics connect:**
> **IP SLA** (the probe) → **Track Object** (the decision) → **Static Route / FHRP** (the action). Together they form the classic automatic-failover design: when a tracked path goes down, the backup route or backup gateway takes over without any manual intervention.

## Repository Structure

```
Networking/
└── <Topic>/
    ├── README.md      ← the full note (theory + configuration)
    └── Labs/          ← EVE-NG (.unl) topologies + PDF guides, when available
```

## Notes on the Labs

Lab archives marked **[Done]** contain the completed topology; the plain archive is the starting point you load into EVE-NG and build out yourself using the matching PDF guide.
