# First-Hop Redundancy Protocols (FHRP)
**Master Study & Review Guide**

## 1. Core Concepts & Operation
FHRP provides **default-gateway redundancy** by sharing a **Virtual IP (VIP)** and a **Virtual MAC Address** among multiple routers.
*   **The Problem:** If a host's only physical gateway fails, the host loses connectivity.
*   **The FHRP Solution:** A backup router seamlessly takes over the Virtual IP/MAC, requiring **zero configuration changes on the hosts**.
*   **ARP Process:** Hosts ARP for the Virtual IP. The currently active/master router replies with the Virtual MAC.
*   **Stacking Exception:** If gateway switches are stacked (operating as a single logical device), FHRP is generally not required.

> ⚠️ **EXAM CRITICAL: The STP Golden Rule**
> In Layer 2 topologies, the FHRP Active/Master router **MUST** be the exact same switch as the **STP Root Bridge** for that VLAN. Misalignment causes suboptimal routing and inefficient Layer 2 paths.

---

## 2. Ultimate FHRP Comparison Matrix

| Feature | HSRP | VRRP | GLBP |
| :--- | :--- | :--- | :--- |
| **Standard** | Cisco Proprietary | IEEE Standard (IETF) | Cisco Proprietary |
| **Primary Roles** | Active, Standby, Listen | Master, Backup | AVG, AVF |
| **Load Balancing** | No (Active only) | No (Master only) | Yes (Multiple AVFs) |
| **Default Preemption**| **Disabled** | **Enabled** | **Disabled** |
| **Multicast IP** | v1: `224.0.0.2` <br> v2: `224.0.0.102` | IPv4: `224.0.0.18` | `224.0.0.102` |
| **Transport** | UDP 1985 | IP Protocol 112 | UDP 3222 |
| **Timers (Hello/Hold)**| 3s / 10s | 1s / ~3s | 3s / 10s |

---

## 3. HSRP (Hot Standby Router Protocol)

**Roles & Election:**
*   **Active:** Forwards traffic. Elected by **Highest Priority** (Default: 100), then **Highest IP Address**.
*   **Standby:** Monitors Active.
*   **Listen:** Other participating routers.

**HSRPv1 vs HSRPv2:**
*   **Groups:** v1 (0-255) vs. v2 (0-4095)
*   **Virtual MAC:** v1 (`0000.0C07.ACxx`) vs. v2 (`0000.0C9F.Fxxx`)
*   **IPv6:** Only v2 supports IPv6.

### HSRP Configuration Cheat Sheet
```cisco
! 1. Basic Setup & Version
SW(config-if)# standby 1 version 2
SW(config-if)# standby 1 ip 192.168.0.254
SW(config-if)# standby 1 ip 192.168.0.253 secondary

! 2. Election (Priority & Preemption)
SW(config-if)# standby 1 priority 110
SW(config-if)# standby 1 preempt delay minimum 30 reload 60

! 3. Authentication & Timers
SW(config-if)# standby 1 authentication md5 key-string MYSECRET
SW(config-if)# standby 1 timers 3 10       ! Or: timers msec 200 msec 750

! 4. Object Tracking
SW(config-if)# standby 1 track 10 decrement 20
SW(config-if)# standby 1 track 10 shutdown

! 5. Verification
SW# show standby brief
```

---

## 4. VRRP (Virtual Router Redundancy Protocol)

**Roles & Election:**
*   **Master:** Owns the virtual IP/MAC and forwards traffic. Elected by **Highest Priority** (Default: 100), then **Highest IP**.
*   **Backup:** Monitors the Master.

**VRRPv2 vs VRRPv3:**
*   **VRRPv2:** RFC 3768, IPv4 only, Authentication supported.
*   **VRRPv3:** RFC 5798, IPv4 & IPv6, Authentication REMOVED from specification.

### VRRP Configuration Cheat Sheet
```cisco
! 1. Basic Setup 
SW(config-if)# vrrp 1 ip 192.168.0.254

! 2. Election (Preemption is ON by default)
SW(config-if)# vrrp 1 priority 110
SW(config-if)# vrrp 1 preempt delay minimum 30

! 3. Timers (Backup can learn from Master)
SW(config-if)# vrrp 1 timers advertise 2
SW(config-if)# vrrp 1 timers learn

! 4. Authentication (VRRPv2 ONLY)
SW(config-if)# vrrp 1 authentication text MYSECRET

! 5. Object Tracking
SW(config-if)# vrrp 1 track 10 decrement 20

! 6. VRRPv3 Specific Setup
SW(config)# fhrp version vrrp v3
SW(config-if)# vrrp 1 address-family ipv4
SW(config-if-vrrp)# address 192.168.0.254
SW(config-if-vrrp)# priority 110

! 7. Verification
SW# show vrrp brief
```

---

## 5. GLBP (Gateway Load Balancing Protocol)

**Roles & Election:**
*   **AVG (Active Virtual Gateway):** Manages the group, replies to ARP. Elected by **Highest Priority** (Default: 100).
*   **AVF (Active Virtual Forwarder):** The routers actively routing traffic. Determined by **Weighting**.

**Load Balancing Methods:**
1.  **Round-Robin (Default):** Rotates MAC assignments per ARP request.
2.  **Weighted:** Traffic volume based on AVF weight configuration.
3.  **Host-Dependent:** Consistent MAC assigned to the same host over time.

### GLBP Configuration Cheat Sheet
```cisco
! 1. Basic Setup & Election (AVG)
SW(config-if)# glbp 1 ip 192.168.0.254
SW(config-if)# glbp 1 priority 110
SW(config-if)# glbp 1 preempt delay minimum 30  ! Preempt is OFF by default

! 2. Load Balancing Config
SW(config-if)# glbp 1 load-balancing round-robin

! 3. Weighting (For AVF roles)
SW(config-if)# glbp 1 weighting 100 lower 80 upper 95
SW(config-if)# glbp 1 weighting track 10 decrement 30
SW(config-if)# glbp 1 forwarder preempt delay minimum 10

! 4. Timers & Redirection
SW(config-if)# glbp 1 timers 3 10
! Redirect: 600s wait before redirecting, 7200s timeout for old MAC
SW(config-if)# glbp 1 timers redirect 600 7200

! 5. Verification
SW# show glbp brief
```
