# First-Hop Redundancy Protocol (FHRP)

**FHRP** is a networking mechanism designed to provide **default-gateway redundancy**.
The concept is straightforward: if the router acting as the primary default gateway fails, a backup router seamlessly takes over. **The hosts on the network do not need to change their default gateway IP address.**

> [Note] **Why is FHRP critical in VLANs?**
> Having multiple redundant physical routers is useless if client hosts are hardcoded to rely on a single physical router's IP as their gateway. FHRP solves this by virtualizing the gateway.
>
> **Note on Stacking:** If you can physically stack the gateway switches (making them operate as one single logical device), you can skip FHRP altogether.

### Quick Protocol Comparison

| Protocol | Abbreviation | Vendor | Versions / Standards | 
| ----- | ----- | ----- | ----- | 
| **Hot Standby Router Protocol** | **HSRP** | Cisco | v1, v2 | 
| **Virtual Router Redundancy Protocol** | **VRRP** | IETF | v2 (**RFC 3768**), v3 (**RFC 5798**) | 
| **Gateway Load Balancing Protocol** | **GLBP** | Cisco | GLBP | 

### Protocol Specifications (Under the Hood)
How do these routers talk to each other to know who is alive? They send Hello packets to specific Multicast IPs.

| Protocol | Multicast IP (IPv4) | Transport Protocol & Port | Default Timers (Hello / Hold) |
| ----- | ----- | ----- | ----- |
| **HSRP v1** | 224.0.0.2 | UDP Port 1985 | 3 seconds / 10 seconds |
| **HSRP v2** | 224.0.0.102 | UDP Port 1985 | 3 seconds / 10 seconds |
| **VRRP (v2/v3)** | 224.0.0.18 | IP Protocol 112 (Direct IP) | 1 second / ~3.6 seconds |
| **GLBP** | 224.0.0.102 | UDP Port 3222 | 3 seconds / 10 seconds |

*(Note: VRRP does not use TCP or UDP at all. It runs directly on top of IP using protocol number 112).*

## How Does FHRP Work?

The entire FHRP process relies on sharing a **Virtual IP** and a **Virtual MAC address**.

1. **ARP Request:** You configure a Virtual IP as the default gateway on client hosts. A host sends an ARP request for this Virtual IP.

2. **Active Router Responds:** The currently active/master router in the FHRP group answers the ARP request with a **Virtual MAC address**.

3. **Traffic Forwarding:** The host uses this Virtual MAC as the destination MAC address to send traffic to the gateway.

* **HSRP & VRRP** use a **single** Virtual MAC address for the entire group, actively owned by the Active/Master router.

* **GLBP** provides load balancing by assigning **different** Virtual MAC addresses to different hosts, distributing the traffic across multiple routers.

## Advanced Design Rules & Group Numbers

Before jumping into configuration, it is crucial to understand how FHRP Group Numbers and Virtual IPs work:

**1. Identical Configurations Required:**
For two routers to form an FHRP relationship, the **Virtual IP Address** and the **Group Number** MUST be absolutely identical on both devices.

**2. Group Numbers are Locally Significant per Interface:**
The group number (e.g., `standby 10`) is only significant to that specific interface/broadcast domain. This means you can reuse the exact same group number on different interfaces without conflict.
*Example: You can use `standby 10` on VLAN 10, and also use `standby 10` on VLAN 20. The generated MAC addresses will be the same, but because they exist in completely isolated Layer 2 domains, they will never collide.*

**3. Multiple FHRP Groups on a SINGLE Interface (Manual Load Balancing):**
You are not restricted to one FHRP group per interface! You can configure multiple groups (e.g., `standby 1` and `standby 2`) under the exact same VLAN.
*Why do this?* It is the classic way to achieve load balancing with HSRP/VRRP. You make Router A Active for Group 1, and Router B Active for Group 2. You then point half of your PCs to Group 1's Virtual IP, and the other half to Group 2's Virtual IP.

## 1. HSRP (Hot Standby Router Protocol)

HSRP is a Cisco-proprietary protocol. One router acts as **Active** (handles traffic), while another acts as **Standby** (waits to take over if the Active fails).

### HSRP States Table

| State | Description | 
| ----- | ----- | 
| **Initial** | HSRP is not running or the interface is down. | 
| **Learn** | Waiting to hear from the Active router. Has not yet determined the Virtual IP. | 
| **Listen** | Knows the Virtual IP, listens for Hellos, but is neither Active nor Standby. | 
| **Speak** | Actively sends Hello packets and participates in the election. | 
| **Standby** | Next in line to become Active. Sends periodic Hello packets. | 
| **Active** | Currently forwarding traffic sent to the Virtual MAC. Sends Hello packets. | 

* **Virtual MAC Format (v1):** `0000.0C07.ACXX` (XX = HSRP group in Hex)
* **Virtual MAC Format (v2):** `0000.0C9F.FXXX` (XXX = HSRP group in Hex)

### MAC Management & Failover in HSRP

The Active router owns and uses the single Virtual MAC address. If the Active router fails, the Standby router instantly takes over. To ensure the downstream Layer 2 switches know about this physical change, the new Active router broadcasts a **Gratuitous ARP (GARP)**. This GARP forces all connected switches to instantly update their MAC address tables, redirecting traffic to the new physical port.

### HSRP Configuration Examples

For these examples, we will use **HSRP Group 10** on **VLAN 10**.

**1. Basic Setup & Version:**

```
! Enter interface configuration
SW(config)# interface Vlan 10

! Upgrade this group to HSRP Version 2 (Supports IPv6 and millisecond timers)
SW(config-if)# standby 10 version 2

! Assign the Virtual IP for Group 10
SW(config-if)# standby 10 ip 192.168.10.254
! (Optional) Assign a secondary Virtual IP
SW(config-if)# standby 10 ip 192.168.10.253 secondary

! (Optional) Manually assign a specific Virtual MAC instead of the default
SW(config-if)# standby 10 mac-address 0000.1111.2222
```

**2. Priority & Preemption:**
*Default priority is 100. Highest becomes Active. Preemption is disabled by default.*

```
! Set priority to 110 (so this router becomes Active)
SW(config-if)# standby 10 priority 110

! Enable preemption (allow this router to take back the Active role)
SW(config-if)# standby 10 preempt

! Enable preemption with delays (Minimum: wait 30s after link up. Reload: wait 60s after reboot)
SW(config-if)# standby 10 preempt delay minimum 30 reload 60
```

**3. Timers & Authentication:**
*Default Hellos are 3s, Hold time is 10s.*

```
! Change timers to 5s Hello and 15s Hold
SW(config-if)# standby 10 timers 5 15
! Or use millisecond timers
SW(config-if)# standby 10 timers msec 200 msec 750

! Configure Plain Text Authentication
SW(config-if)# standby 10 authentication text MYSECRET
! Configure MD5 authentication using a key-string
SW(config-if)# standby 10 authentication md5 key-string Cisco123!
```

**4. Track Objects:**

```
! If tracked object 1 (e.g., a WAN link) goes down, reduce HSRP priority by 20
SW(config-if)# standby 10 track 1 decrement 20

! If tracked object 1 goes down, shut down HSRP on this interface completely
SW(config-if)# standby 10 track 1 shutdown
```

**5. Verification:**

```
SW# show standby
SW# show standby brief
```

## 2. VRRP (Virtual Router Redundancy Protocol)

VRRP is an open-standard protocol. It uses a **Master** router (forwards traffic) and **Backup** routers (monitor and take over if needed).

### VRRP States Table

| State | Description | 
| ----- | ----- | 
| **Initialize** | Waiting for a startup event or for the interface to come up. | 
| **Backup** | Monitors the Master router. Ready to transition to Master if it fails. | 
| **Master** | Actively forwards packets sent to the Virtual MAC and sends advertisements. | 

* **Virtual MAC Format:** `0000.5E00.01XX` (XX = VRRP group in Hex).

### MAC Management & Failover in VRRP

Similar to HSRP, the Master router holds the single Virtual MAC address. Upon failure, the Backup router transitions to the Master state, assumes the exact same Virtual MAC, and immediately sends a **Gratuitous ARP (GARP)** out to the network. This ensures all switches instantly map the Virtual MAC to the new Master router's physical port.

### VRRP Configuration Examples

For these examples, we will use **VRRP Group 20** on **VLAN 20**.

**1. Basic Setup & Election:**
*Preemption is ON by default in VRRP. Default priority is 100.*

```
SW(config)# interface Vlan 20

! Assign the Virtual IP 
SW(config-if)# vrrp 20 ip 10.0.20.254

! Set priority to 150 (Highest becomes Master)
SW(config-if)# vrrp 20 priority 150

! Delay preemption by 30 seconds to prevent network flapping
SW(config-if)# vrrp 20 preempt delay minimum 30
```

**2. Timers, Tracking & Authentication:**
*Default Hello is 1s. Hold time is calculated automatically based on priority and hello time.*

```
! Set advertisement interval to 2 seconds (or use msec)
SW(config-if)# vrrp 20 timers advertise 2

! Allow Backup routers to automatically learn the timer from the Master router
SW(config-if)# vrrp 20 timers learn

! Track an object and decrement priority by 60 if it fails
SW(config-if)# vrrp 20 track 5 decrement 60
```

**3. VRRPv3 Specific Setup (IPv4 & IPv6 Support):**
*Authentication was removed in VRRPv3. Requires global activation first.*

```
! 1. Globally enable VRRPv3 features
SW(config)# fhrp version vrrp v3

! 2. Enter interface and define the Address Family (IPv4 or IPv6)
SW(config)# interface Vlan 20
SW(config-if)# vrrp 20 address-family ipv4

! 3. Configure parameters inside the VRRPv3 sub-mode
SW(config-if-vrrp)# address 10.0.20.254
SW(config-if-vrrp)# priority 150
SW(config-if-vrrp)# track 5 decrement 60
```

**4. Verification:**

```
SW# show vrrp
SW# show vrrp brief
```

## 3. GLBP (Gateway Load Balancing Protocol)

GLBP is a Cisco-proprietary protocol offering both redundancy and **active load balancing**.

* **AVG (Active Virtual Gateway):** Manages the group, replies to ARP, assigns MACs.
* **AVF (Active Virtual Forwarder):** The actual routers forwarding the traffic.

### GLBP Load Balancing Methods

1. **Round-Robin (Default):** The AVG sequentially hands out the Virtual MAC addresses of the available AVFs to clients. *Best when all routers have identical capabilities.*
2. **Weighted:** Traffic is distributed proportionally based on a manually configured "weight" for each AVF. *Best when participating routers have different hardware capacities.*
3. **Host-Dependent:** The AVG guarantees that a specific host is always assigned the exact same AVF Virtual MAC address. *Crucial for stateful applications.*

### GLBP States Table (AVG Roles)

| State | Description | 
| ----- | ----- | 
| **Disabled** | GLBP is not configured or the interface is disconnected. | 
| **Initial** | Interface is up, GLBP is starting initialization. | 
| **Listen** | Receiving Hello packets. Ready to switch to Speak if current AVG fails. | 
| **Speak** | Actively attempting to become the Active Virtual Gateway (AVG). | 
| **Standby** | Next in line to become the AVG. | 
| **Active** | Functioning as the AVG (assigning Virtual MACs) or as an AVF. | 

* **Virtual MAC Format:** `0007.b400.XXYY` (XX = group, YY = AVF number).

### MAC Management & Failover in GLBP

Unlike HSRP/VRRP, GLBP uses *multiple* Virtual MACs. The AVG assigns a unique MAC to each AVF.

* **If an AVF fails:** Another active AVF will temporarily take over the failed router's Virtual MAC (in addition to its own) and send a **GARP**. This ensures client PCs pointing to the failed MAC do not drop their connections. Meanwhile, the AVG stops assigning the failed MAC to any new ARP requests.
* **If the AVG fails:** The Standby AVG takes over the process of answering ARPs and managing the MAC address assignments, without disrupting the AVFs.

### GLBP Configuration Examples

For these examples, we will use **GLBP Group 30** on **VLAN 30**.

**1. Basic Setup & AVG Election:**
*Preemption is OFF by default. Priority determines the AVG.*

```
SW(config)# interface Vlan 30

! Assign Virtual IP
SW(config-if)# glbp 30 ip 172.16.30.254

! Set priority to 110 (to win the AVG election)
SW(config-if)# glbp 30 priority 110

! Enable preemption for the AVG role with a 20-second delay
SW(config-if)# glbp 30 preempt delay minimum 20
```

**2. Load Balancing Configuration:**

```
! Set load balancing to round-robin, weighted, or host-dependent
SW(config-if)# glbp 30 load-balancing weighted
```

**3. AVF Weighting (Who actually forwards traffic):**
*Weighting determines if a router can act as an AVF.*

```
! Set max weight to 100. Stop forwarding if it drops below 80. Resume at 95.
SW(config-if)# glbp 30 weighting 100 lower 80 upper 95 

! Track an interface. If it goes down, decrease weight by 30
SW(config-if)# glbp 30 weighting track 2 decrement 30 
```

**4. Timers & Authentication:**
*Default Hellos are 3s, Hold time is 10s.*

```
! Set Hellos to 3s and Hold to 10s (or msec)
SW(config-if)# glbp 30 timers 3 10

! Authentication options
SW(config-if)# glbp 30 authentication md5 key-string Cisco123!
```

**5. Verification:**

```
SW# show glbp
SW# show glbp brief
```

## Crucial Design Rule: FHRP & STP Alignment

When utilizing Layer 2 infrastructure for FHRP gateways, **STP (Spanning Tree Protocol)** is running to prevent loops.

To prevent traffic from taking inefficient, sub-optimal paths through the Layer 2 network (tromboning), **the HSRP Active / VRRP Master / GLBP AVG router MUST align with the STP Root Bridge** for the corresponding VLAN. Always tune your STP priority so that the primary network/core switch acting as the gateway is exactly the same as the Root Bridge for that VLAN.
