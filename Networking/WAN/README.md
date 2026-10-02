# Comprehensive Reference: Wide Area Networks (WAN)

This reference guide covers the fundamental concepts, layer-based categorizations, and underlying technologies of Wide Area Networks (WAN), based on standard telecommunications and networking principles.

## 1. What is a WAN?

A **Wide Area Network (WAN)** is a network designed to connect multiple Local Area Networks (LANs) across broad geographic distances.

**The Key Difference Between LAN and WAN (Ownership):**

* **LAN:** The organization or user typically owns all the infrastructure, cables, and equipment.

* **WAN:** The organization owns the end equipment (like the router), but the physical transmission media (cables, lines) are owned and maintained by a Service Provider or Telecommunications company (Telco).

### Key Physical Terminologies in WAN

To understand WAN connections, it is crucial to know these standard terms:

* **CPE (Customer Premises Equipment):** The devices physically located on the customer's site (e.g., the customer's router).

* **DTE (Data Terminal Equipment):** The endpoint of the user's network on the WAN link (usually the customer's router interface). It sends data to the DCE.

* **DCE (Data Communications Equipment):** The device that provides the physical connection to the network, forwards traffic, and **provides a clocking signal** used to synchronize data transmission (usually a modem or CSU/DSU provided by the Telco).

* **Demarcation Point (Demarc):** The physical point where the service provider's responsibility ends and the customer's responsibility begins.

## 2. WAN Categories by OSI Layer

WAN services can be categorized based on the OSI layer at which the service provider hands off the connection to the customer.

### Layer 1 WAN (Physical Layer)

* **Examples:** Dark Fiber, TDM (Time Division Multiplexing).

* **Characteristics:** Dedicated bandwidth, Private WAN. The customer has full control over the link and the protocols used.

### Layer 2 WAN (Data Link Layer)

* **Examples:** Metro Ethernet, MPLS L2 (Layer 2 VPNs).

* **Characteristics:** Shared bandwidth (within the provider's core network), Private WAN. The provider handles MAC addressing or Label switching, but IP routing is entirely up to the customer.

### Layer 3 WAN (Network Layer)

* **Examples:** MPLS L3 (Layer 3 VPNs), Internet, Intranet.

* **Characteristics:** Shared bandwidth. Can be Public (Internet) or Private (MPLS L3 VPN).

* **Crucial Rule:** The service provider actively participates in the customer's IP routing. To use a Layer 3 WAN, the customer's edge router must run a routing protocol (like OSPF, EIGRP, or BGP) with the provider's router to exchange LAN networks.

## 3. Layer 1 WAN Technologies in Detail

### A. Dark Fiber

* **Definition:** "Dark Fiber" refers to physical, underground fiber optic cables that are installed but currently unlit (no active light/signal is passing through them).

* **How it Works:** It is a passive service. The customer leases the physical fiber strand from the provider.

* **Hardware:** The customer is completely responsible for providing the active equipment (Transceivers/SFPs) at both ends to light the fiber. The telecom company does not provide routing or switching, only the physical path.

### B. TDM (Time Division Multiplexing) and PCM

TDM is a method of putting multiple data streams in a single signal by separating the signal into many segments, each having a very short duration. TDM operates as Point-to-Point (P2P) connections using serial links (physical or virtual).

**PCM (Pulse Code Modulation):**
PCM is the standard method used to convert analog signals (continuous waves) into digital signals (discrete binary bits) for transport over digital WAN links.

* **The Process:** It involves 3 steps: Sampling, Quantization, and Encoding.

* **Sampling Rate (Nyquist Theorem):** To accurately reproduce human voice (which maxes out around 4000 Hz), the Nyquist theorem dictates we must sample at twice the highest frequency. Therefore, the standard telephone voice system takes **8000 samples per second**.

* **Sample Size:** Each sample is encoded into **8 bits**.

* **The DS0 Standard:** $8000 \text{ samples/sec} \times 8 \text{ bits/sample} = 64,000 \text{ bits per second}$ (**64 Kbps**).

* This 64 Kbps digital channel is called a **DS0** (Digital Signal 0), which is the fundamental building block of all digital TDM circuits.

### C. Link Capacities and Port Types

By multiplexing multiple DS0 channels together, providers create higher-capacity links.

**Copper Links (T-Carrier & E-Carrier):**
* **E1 (Europe/Global):** 32 x DS0 = **2.048 Mbps**
* **E3 (Europe/Global):** 512 x DS0 = **34.064 Mbps**
* **T1 (US/Japan):** 24 x DS0 = **1.544 Mbps**
* **T3 (US/Japan):** 672 x DS0 = **44.736 Mbps**

**Fiber Optic Links (SDH/SONET):**
* **STM-1:** 64 x E1 = **155 Mbps**
* **STM-4:** 256 x E1 = **622 Mbps**
* **STM-16:** 1024 x E1 = **2.4 Gbps**
* **STM-64:** 4096 x E1 = **9.6 Gbps**
* **STM-256:** 16384 x E1 = **40 Gbps**

### D. TDM Encapsulation Protocols and Configuration

To transmit data over point-to-point TDM serial links, Layer 2 encapsulation protocols are required.

* **Options:** HDLC, PPP, Frame-Relay, ATM.

* **HDLC vs. PPP:** For simple point-to-point serial links, HDLC (the Cisco default) or PPP are used. **PPP is generally preferred** because it has far more features than HDLC, including Link Quality Monitoring (LQM), authentication (PAP/CHAP), and multi-link support (MLPPP).

**Basic Cisco Router Configuration for Serial Links (PPP):**

```
Router(config)# interface serial 0/0/0
! Set the encapsulation type to PPP
Router(config-if)# encapsulation ppp 

! Assign the IP address for routing
Router(config-if)# ip address X.X.X.X Y.Y.Y.Y

! (Optional) Enable PPP Authentication using CHAP
Router(config-if)# ppp authentication chap

! Note: If this router is simulating the provider side (DCE) in a lab, you must provide clocking
Router(config-if)# clock rate 64000

! Turn on the interface
Router(config-if)# no shutdown
```

**Verification Commands:**

```
Router# show interfaces serial 0/0/0
! Useful to see if the line protocol is up and which encapsulation is active.

Router# debug ppp negotiation
! Essential for troubleshooting PPP connection or authentication issues.
```

## 4. Layer 2 WAN: Metro Ethernet & MPLS L2

**Metro Ethernet** is the delivery of Ethernet services at a metropolitan or wide-area scale by a service provider, usually over a fiber-optic infrastructure. It allows companies to use standard Ethernet protocols over long distances.

It comes in several flavors based on the communication topology:

1. **VPWS (Virtual Private Wire Service) / E-Line:**
   * **Topology:** Point-to-Point (P2P).
   * **Use Case:** Acts exactly like a very long Ethernet cable connecting two sites. Both sites are in the same broadcast domain. It is heavily used for VLAN trunking between branches.

2. **VPLS (Virtual Private LAN Service) / E-LAN:**
   * **Topology:** Full-Mesh / Multipoint-to-Multipoint.
   * **Use Case:** Connects multiple sites to a single logical switch provided by the ISP. All sites can communicate directly with each other in the same broadcast domain.

3. **EoMPLS (Ethernet over MPLS) / E-Tree:**
   * **Topology:** Hub-and-Spoke (Point-to-Multipoint).
   * **Use Case:** Branches (spokes) can communicate with the main HQ (hub), but branches cannot communicate directly with each other at Layer 2.

## 5. Introduction to MPLS Concepts

Multiprotocol Label Switching (MPLS) is a highly scalable data-carrying mechanism used by service providers to speed up and shape traffic flows.

* **The 32-bit Label ("Layer 2.5"):** MPLS works by inserting a 32-bit header (the Label) between the Layer 2 header (Data Link, like Ethernet MAC) and the Layer 3 header (Network, like IP).

* **Label Switching vs. IP Routing:** Instead of making complex routing decisions based on large IP routing tables at every hop, MPLS routers inside the provider's network (LSRs) simply forward packets based on these short, fixed-length labels. This is much faster.

* **Versatility:** MPLS is completely protocol-agnostic. It can carry Layer 2 traffic (as seen in VPLS/VPWS) or Layer 3 IP traffic across its infrastructure.

### MPLS Router Roles (The Architecture)

To understand MPLS, you must know the three distinct router roles involved in the connection:

* **CE (Customer Edge):** The router located at the customer's site. It knows nothing about MPLS labels. It just routes standard IP packets to the provider.

* **PE (Provider Edge):** The provider's router that connects to the CE. The PE is the most intelligent device; it takes the customer's IP packets, slaps an MPLS label on them (Push), and sends them into the provider's core network.

* **P (Provider):** The core routers inside the provider's MPLS network. They do not know about the customer's IP routes. They only look at the MPLS labels and switch the packets extremely fast based on those labels (Swap).

## 6. Modern WAN Supplements: SD-WAN & VPNs

While TDM and MPLS are foundational, modern networks often rely on internet-based overlays to reduce costs and increase flexibility.

### A. SD-WAN (Software-Defined WAN)
SD-WAN abstracts the underlying WAN infrastructure (MPLS, Broadband Internet, LTE/5G). It uses a centralized controller to intelligently route traffic across the best available path based on real-time network conditions and application policies, rather than relying solely on traditional routing protocols.

### B. VPN Tunnels (Over Layer 3 WAN)
When using a public Layer 3 WAN (like the Internet), traffic is not inherently secure. Companies use tunnels to create a Private WAN overlay:
* **GRE (Generic Routing Encapsulation):** Creates a point-to-point tunnel that can carry routing protocols (like OSPF or EIGRP) over the internet, but it provides **no encryption**.
* **IPsec (Internet Protocol Security):** Provides strong encryption and data integrity, ensuring data sent over the public internet remains secure. Often, GRE and IPsec are combined to get both routing support and security.
