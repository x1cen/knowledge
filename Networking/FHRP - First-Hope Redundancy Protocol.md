# First-Hop Redundancy Protocol (FHRP)

**FHRP** is a networking mechanism designed to provide **default-gateway redundancy**. 
The concept is straightforward: if the router acting as the primary default gateway fails, a backup router seamlessly takes over. **The hosts on the network do not need to change their default gateway IP address.**

> 💡 **Why is FHRP critical in VLANs?**
> Having multiple redundant physical routers is useless if client hosts are hardcoded to rely on a single physical router's IP as their gateway. FHRP solves this by virtualizing the gateway.
>
> 📌 **Note on Stacking:** If you can physically stack the gateway switches (making them operate as one single logical device), you can skip FHRP altogether.

### Quick Protocol Comparison

| Protocol | Abbreviation | Vendor | Versions / Standards |
| :--- | :--- | :--- | :--- |
| **Hot Standby Router Protocol** | **HSRP** | Cisco | v1, v2 |
| **Virtual Router Redundancy Protocol** | **VRRP** | IETF | v2 (**RFC 3768**), v3 (**RFC 5798**) |
| **Gateway Load Balancing Protocol** | **GLBP** | Cisco | GLBP |

---

## How Does FHRP Work?

The entire FHRP process relies on sharing a **Virtual IP** and a **Virtual MAC address**. Here is the step-by-step process:

1. **ARP Request:** You configure a Virtual IP as the default gateway on client hosts. A host sends an ARP request for this Virtual IP.
2. **Active Router Responds:** The currently active/master router in the FHRP group answers the ARP request with a **Virtual MAC address**.
3. **Traffic Forwarding:** The host uses this Virtual MAC as the destination MAC address to send traffic to the gateway.

* **HSRP & VRRP** use a **single** Virtual MAC address for the entire group, owned by the Active/Master router.
* **GLBP** provides load balancing by assigning **different** Virtual MAC addresses to different hosts, distributing the traffic across multiple active routers.

---

## 1. HSRP (Hot Standby Router Protocol)

HSRP is a Cisco-proprietary protocol. One router acts as **Active** (handles traffic), while another acts as **Standby** (waits to take over if the Active fails).

* **HSRP Roles:** Active, Standby, Listen.
* **HSRP States:** Initial, Learn, Listen, Speak, Standby, Active.

### Basic Setup
Ensure the Virtual IP is unique. The group number and Virtual IP must match on all participating routers. *You can run multiple HSRP groups on a single interface.*

```cisco
! Enable HSRP and assign the primary Virtual IP
SW(config-if)# standby <group-number> ip <ip-address>

! (Optional) Add a secondary Virtual IP
SW(config-if)# standby <group-number> ip <ip-address> secondary
```

### Priority & Election
The router with the highest priority becomes Active. 
* **Default Priority:** 100.
* **Tie-breaker:** If priorities are equal, the router with the **highest IP address** wins.

```cisco
SW(config-if)# standby <group-number> priority <priority-value>
```

### Preemption
**Disabled by default.** Without preemption, a newly booted router with a higher priority will *not* take over the Active role unless the current Active router fails. Configuring a delay automatically enables preemption.
* **Minimum Delay:** Waits specified seconds after the interface comes up.
* **Reload Delay:** Waits specified seconds after the entire router reboots.

```cisco
SW(config-if)# standby <group-number> preempt 
SW(config-if)# standby <group-number> preempt delay minimum <value> reload <value>
```

### Timers & Authentication
By default, Hello packets are sent every 3 seconds (UDP 1985, Multicast `224.0.0.2`), and Hold time is 10 seconds.

```cisco
! Set timers (in seconds or milliseconds)
SW(config-if)# standby <group-number> timers <hello-time> <hold-time>
SW(config-if)# standby <group-number> timers msec <hello-time> msec <hold-time>

! Authentication Options
SW(config-if)# standby <group-number> authentication text <text-string>
SW(config-if)# standby <group-number> authentication md5 key-string <key-string> 
```

### Track Objects
You can track an object (like an uplink interface). If it goes down, HSRP dynamically lowers the router's priority, forcing a failover.

```cisco
SW(config-if)# standby <group-number> track <object-number> decrement <value>
SW(config-if)# standby <group-number> track <object-number> shutdown
```

### Virtual MAC & Verification
```cisco
! Manually assign a Virtual MAC
SW(config-if)# standby <group-number> mac-address <value>

! Verify HSRP
SW# show standby brief
SW# show standby neighbors
```

### HSRPv1 vs HSRPv2

| Feature | HSRPv1 | HSRPv2 |
| :--- | :--- | :--- |
| Group IDs | `0–255` | `0–4095` |
| Multicast IP | `224.0.0.2` | `224.0.0.102` |
| Virtual MAC Format | `0000.0C07.ACXX` | `0000.0C9F.FXXX` |
| IPv6 Support | No | Yes |

```cisco
! Enable HSRPv2
SW(config-if)# standby <group-number> version 2
```

---

## 2. VRRP (Virtual Router Redundancy Protocol)

VRRP is an open-standard protocol (IETF). It uses a **Master** router (forwards traffic) and **Backup** routers (monitor and take over if needed).

* **VRRP Roles:** Master, Backup.
* **VRRP States:** Initialize, Master, Backup.
* **Virtual MAC Format:** `0000.5E00.01XX` (XX = VRRP group in Hex).

### Basic Setup
```cisco
SW(config-if)# vrrp <group-number> ip <ip-address> [secondary]
```

### Priority & Preemption
* **Default Priority:** 100 (Highest priority becomes Master. Tie-breaker is the highest IP).
* **Preemption:** **Enabled by default.** You can add a delay to allow routing protocols to initialize before taking over.

```cisco
SW(config-if)# vrrp <group-number> priority <priority-value>
SW(config-if)# vrrp <group-number> preempt delay minimum <value>
```

### Timers & Authentication
Advertisements are sent every 1 second (Multicast `224.0.0.18`, IP protocol 112). 
Hold Time is calculated as: $3 \times (\text{hello-interval}) + \frac{256 - \text{priority}}{256}$

```cisco
! Set Advertisement Timer
SW(config-if)# vrrp <group-number> timers advertise <value>

! Backup learns timer from Master
SW(config-if)# vrrp <group-number> timers learn

! Authentication (VRRPv2 ONLY)
SW(config-if)# vrrp <group-number> authentication text <text-string>
```

### Track Objects & Verification
```cisco
SW(config-if)# vrrp <group-number> track <object-number> decrement <value>

! Verify VRRP
SW# show vrrp brief
```

### VRRPv2 vs VRRPv3

| Feature | VRRPv2 (RFC 3768) | VRRPv3 (RFC 5798) |
| :--- | :--- | :--- |
| Address Support | IPv4 Only | IPv4 and IPv6 |
| Authentication | Supported | Removed from standard |

**VRRPv3 Setup Example:**
```cisco
SW(config)# fhrp version vrrp v3
SW(config-if)# vrrp <group-number> address-family ipv4
SW(config-if-vrrp)# address <ip> 
SW(config-if-vrrp)# priority <value>
```

---

## 3. GLBP (Gateway Load Balancing Protocol)

GLBP is a Cisco-proprietary protocol that offers both redundancy and **active load balancing**. Multiple routers actively forward traffic simultaneously.

* **AVG (Active Virtual Gateway):** Answers ARP requests and assigns MAC addresses (Elected by Priority).
* **AVF (Active Virtual Forwarder):** The routers actually forwarding traffic (Determined by Weighting).
* **Virtual MAC Format:** `0007.b400.XXYY` (XX = group, YY = AVF number).

### Basic Setup & Priority
Like HSRP/VRRP, highest priority wins the AVG role. Tie-breaker is highest IP.

```cisco
SW(config-if)# glbp <group-number> ip <ip-address> [secondary]
SW(config-if)# glbp <group-number> priority <priority-value>
```

### Preemption
**Disabled by default.** Setting a delay enables it automatically.

```cisco
SW(config-if)# glbp <group-number> preempt delay minimum <value>
```

### Timers & Redirect
Default Hello is 3s (UDP 3222, Multicast `224.0.0.102`), Hold time is 10s.

```cisco
SW(config-if)# glbp <group-number> timers <hello-time> <hold-time>

! Redirect Timers (Example: Wait 600s to redirect, old MAC valid for 7200s)
SW(config-if)# glbp <group-number> timers redirect 600 7200
```

### Weighting & Load Balancing Methods
Weighting dictates which switches act as AVFs (Forwarders). Load balancing methods dictate how the AVG hands out Virtual MACs.

* **Round-Robin (Default):** Sequentially hands out MACs to hosts.
* **Weighted:** Distributes traffic based on AVF configured weight.
* **Host-Dependent:** Ensures a specific host always gets the same MAC.

```cisco
! Configure AVF weighting and tracking
SW(config-if)# glbp <group-number> weighting <value> lower <value> upper <value> 
SW(config-if)# glbp <group-number> weighting track <object-number> decrement <value> 

! Choose load balancing method
SW(config-if)# glbp <group-number> load-balancing [host-dependent | round-robin | weighted]
```

### Authentication & Verification
```cisco
SW(config-if)# glbp <group-number> authentication md5 key-string <key-string> 

! Verify GLBP
SW# show glbp brief
```

---

## ⚠️ Crucial Design Rule: FHRP & STP Alignment

When implementing FHRP in a Layer 2 network, the **Spanning Tree Protocol (STP)** is actively preventing loops. 

To prevent traffic from taking inefficient, sub-optimal paths (tromboning), **the HSRP Active / VRRP Master / GLBP AVG router MUST align with the STP Root Bridge** for the corresponding VLAN. Always tune your STP priority so that the primary routing switch is also the center of your Layer 2 topology.
