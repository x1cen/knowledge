# Static Routing & Fundamentals

## 1. What is Static Routing?

Static routing is the process of manually typing a route into the router's routing table. Instead of letting the router automatically learn paths using protocols like OSPF or BGP, you explicitly tell the router: "To get to network X, send the traffic to router Y."

* **The Pros:** Zero CPU/RAM overhead, completely predictable, and highly secure (no routing updates to intercept).

* **The Cons:** Does not adapt to network changes. If a link goes down, the static route stays there (unless you use a Track Object) and traffic drops.

## 2. AD (Administrative Distance) vs. Metric

When a router needs to make a routing decision, it looks at two completely different numbers to choose the best path: Administrative Distance and Metric.

**Administrative Distance (AD): The Trustworthiness**
AD is the measure of how reliable a routing *source* is. If a router learns about the same destination from OSPF and also from a Static Route, it uses AD to decide which source to believe. Lower AD is always better.

**Metric: The Cost**
If the router learns about multiple paths to the exact same destination from the *exact same protocol*, it uses the Metric to break the tie. The path with the lower cost (Metric) wins.

### Standard Administrative Distance (AD) Table

| Routing Source | Default AD Value | 
 | ----- | ----- | 
| Connected Interface | 0 | 
| **Static Route** | **1** | 
| EIGRP Summary Route | 5 | 
| External BGP (eBGP) | 20 | 
| Internal EIGRP | 90 | 
| IGRP | 100 | 
| OSPF | 110 | 
| IS-IS | 115 | 
| RIP | 120 | 
| External EIGRP | 170 | 
| Internal BGP (iBGP) | 200 | 
| Unknown (Unreachable) | 255 | 

## 3. Configuration: IPv4 & IPv6 Static Routes

**Standard IPv4 Static Route:**

```
Router(config)# ip route 10.50.0.0 255.255.255.0 192.168.1.2
```

**Floating Static Route (Backup Route):**
Change the AD (e.g., to 130) so it hides in the background and only takes over if the primary route fails.

```
Router(config)# ip route 10.50.0.0 255.255.255.0 192.168.1.3 130
```

**IPv6 Static Route:**
*(Always remember to enable `ipv6 unicast-routing` globally first!)*

```
Router(config)# ipv6 route 2001:DB8:ACAD::/64 2001:DB8:CAFE::2
```

## 4. Next-Hop IP vs. Exit Interface (The P2P vs Ethernet Rule)

When configuring a static route, you can either point it to a Next-Hop IP Address, or an Exit Interface. The type of network connection dictates which one you should use.

**The Danger of Ethernet (Multi-Access Networks):**
If you point a static route to an Ethernet interface (e.g., `ip route 10.0.0.0 255.255.255.0 GigabitEthernet0/0`), the router assumes the destination is directly connected. It will send an ARP request for EVERY single destination IP. The next router will reply using **Proxy ARP**. Your router's ARP table will explode, killing the router's memory and CPU.
**Rule:** On Ethernet, ALWAYS use the Next-Hop IP Address.

**The Point-to-Point (P2P) Exception:**
Can we use the Exit Interface on Point-to-Point links like Serial interfaces or GRE Tunnels?
**YES!** In fact, it is often preferred.
**Why?** Because on a Point-to-Point link, there is only one possible device at the other end of the wire. There is no ARP process needed. The router simply shoves the packet out the interface, knowing the router on the other side is the only one who can receive it.

```
! Perfectly fine on a P2P link
Router(config)# ip route 10.0.0.0 255.255.255.0 Serial0/0/0
```

## 5. What exactly is Proxy ARP?

To understand the danger mentioned above, we must understand Proxy ARP.
Normally, ARP is used by a device to find the MAC address of another device on the **same local network**.

**The Concept:** Proxy ARP is when a router answers an ARP request on behalf of a device that is on a *completely different network*. The router acts like a proxy (a middleman).

**How it works (A Simple Scenario):**

1. PC-A wants to send data to Server-B.

2. PC-A has a misconfigured subnet mask, so it mistakenly thinks Server-B is on its own local network.

3. PC-A shouts an ARP Broadcast: "Hey, who has the MAC address for Server-B?"

4. The Router hears this. It checks its routing table and says: "I know how to get to Server-B. I will reply to PC-A and give it **MY OWN MAC address**."

5. PC-A sends the packet to the Router's MAC address, and the Router forwards it to Server-B.

**Why is Proxy ARP considered bad practice today?**
While it seems helpful, Proxy ARP hides network misconfigurations (like the wrong subnet mask on PC-A). More importantly, it creates a massive security risk (it makes ARP Spoofing and Man-in-the-Middle attacks much easier) and can exhaust the router's memory if it has to reply to thousands of incorrect ARP requests (as seen in the Ethernet Static Route issue).

**How to disable it:**
Proxy ARP is enabled by default on Cisco routers! It is highly recommended to disable it on your interfaces unless you have a specific legacy reason to keep it.

```
Router(config)# interface GigabitEthernet0/0
Router(config-if)# no ip proxy-arp
```

## 6. What is Recursive Routing (Recursive Lookup)?

When you type a static route, the router must know how to reach the Next-Hop IP.

* **Directly Connected:** If the Next-Hop IP is on a subnet physically attached to your router, it takes one quick look at the routing table and sends the packet.

* **Recursive Lookup:** What if the Next-Hop IP is 5 routers away? Your router must perform a "Double Lookup". First, it looks up the destination network and finds the Next-Hop IP. Second, it searches the routing table *again* to figure out how to reach that Next-Hop IP.

If the router cannot find a valid path to the Next-Hop IP during this recursive lookup, the static route is considered invalid and is **removed** from the active routing table.

## 7. Reliable Static Routing (The Track Object Scenario)

A normal static route is dumb; if the next-hop router goes offline but your physical interface stays up (e.g., connected through an unmanaged switch), the static route stays in the table, blackholing your traffic. We fix this by tying the static route to a Track Object.

**Scenario:** We want our default route to point to ISP 1 (8.8.8.8). If we cannot ping 8.8.8.8, the static route should automatically delete itself so the backup route can take over.

**Step 1: Set up the IP SLA (The Ping Test)**

```
Router(config)# ip sla 1
Router(config-ip-sla)# icmp-echo 8.8.8.8
Router(config-ip-sla-echo)# frequency 10
Router(config-ip-sla-echo)# exit
Router(config)# ip sla schedule 1 life forever start-time now
```

**Step 2: Create the Track Object (The Monitor)**

```
Router(config)# track 10 ip sla 1 reachability
```

**Step 3: Tie the Track Object to the Static Route**

```
! Primary route tied to Track 10
Router(config)# ip route 0.0.0.0 0.0.0.0 8.8.8.8 track 10

! Backup route (Floating Static Route with AD 10)
Router(config)# ip route 0.0.0.0 0.0.0.0 4.2.2.4 10
```

## 8. The Null0 Route (The Blackhole)

`Null0` is a virtual trash can inside the router. Any packet routed to Null0 is instantly dropped.

**Why use it?**

1. **Security:** Drop traffic from known malicious IP ranges without heavily taxing the CPU.

2. **Preventing Routing Loops:** When creating summary routes, you point the summary IP block to Null0. If a packet arrives for an IP within that block that does not actually exist in your network, it hits the Null0 route and dies, instead of bouncing back to the internet and causing a loop.

```
! Blackhole all traffic to this subnet
Router(config)# ip route 172.16.99.0 255.255.255.0 Null0
```

## 9. Load Balancing with Static Routes (ECMP)

If you have two internet links and want to use BOTH of them simultaneously (Active/Active), you use **Equal-Cost Multi-Path (ECMP)** load balancing.

**The Rule:** If you write multiple static routes to the *exact same destination* with the *exact same Administrative Distance*, the router will install all of them in the routing table and balance the traffic across them.

**Configuration Scenario:** Load balancing the default route across two ISPs.

```
! Both routes have the default AD of 1. Both will be active.
Router(config)# ip route 0.0.0.0 0.0.0.0 8.8.8.8
Router(config)# ip route 0.0.0.0 0.0.0.0 4.2.2.4
```

### How does CEF handle the Load Balancing?

Cisco Express Forwarding (CEF) manages how the traffic is actually distributed. It uses two main methods:

1. **Per-Destination (Per-Flow) - The Default Method:**
   The router looks at the Source IP and Destination IP. All packets belonging to a specific conversation (e.g., PC-A talking to Server-B) will ALWAYS take the exact same link.
   * *Benefit:* Guarantees that packets arrive in order, which is critical for TCP traffic and VoIP.

2. **Per-Packet (Round Robin):**
   The router alternates packet by packet. Packet 1 goes out Link A, Packet 2 goes out Link B, Packet 3 goes out Link A, etc.
   * *Drawback:* Highly likely to cause packets to arrive out of order, which can severely degrade TCP performance. This is rarely used in modern networks.

**Changing the Load Balancing Method:**
Load balancing behavior is configured directly on the outgoing interface.

```
! Change to Per-Packet (Round Robin) - Not recommended for normal traffic
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ip load-sharing per-packet

! Revert to Per-Destination (Default and Recommended)
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ip load-sharing per-destination
```

## 10. Verification & Show Commands

Use these essential commands to verify your static routing configuration.

**Show only the static routes in the routing table:**

```
Router# show ip route static
```

**View the detailed routing process for a specific IP (Shows exactly how the router performs the recursive lookup for this destination):**

```
Router# show ip route 10.50.0.5
```

**Check which static routes are actively using Track Objects, and see if their track status is currently UP or DOWN:**

```
Router# show ip route track-table
```

**Check the IPv6 routing table specifically for static routes:**

```
Router# show ipv6 route static
```
