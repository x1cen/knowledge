# Virtual Routing and Forwarding (VRF)

## 1. What is VRF?

**Virtual Routing and Forwarding (VRF)** is a technology that allows multiple instances of a routing table to co-exist within the same router at the same time.

* **The Concept:** Just as **VLANs** virtualize a physical switch into multiple independent Layer 2 broadcast domains, **VRFs** virtualize a physical router into multiple independent Layer 3 routing domains.

* **The Benefit:** Because the routing instances are completely isolated, you can use **overlapping IP addresses** across different VRFs without any conflict. (e.g., Customer A and Customer B can both use the `192.168.1.0/24` subnet on the same physical router).

## 2. Configuration Syntax: Legacy vs. Modern

Before jumping into configurations, it is crucial to know that Cisco has two different syntaxes for configuring VRFs, depending on the IOS version. This applies to both VRF Lite and Full VRF:

* **Legacy Syntax (`ip vrf`):** Used in older IOS versions. It supports **IPv4 ONLY**.

* **Modern Syntax (`vrf definition`):** Introduced in newer IOS (15.x+) and IOS-XE. It is multi-protocol, supporting **both IPv4 and IPv6** within the same VRF.

## 3. Quick Comparison: VRF Lite vs. Full VRF

| Feature | VRF Lite | Full VRF (MPLS L3VPN) | 
| :--- | :--- | :--- | 
| **Underlying Tech** | Standard IP Routing, 802.1Q (VLANs) | MPLS, MP-BGP | 
| **Typical Environment** | Enterprise / Campus | Service Provider / Large Datacenters | 
| **Route Exchange** | Standard protocols (OSPF, EIGRP) separated per VRF. | MP-BGP exchanges routes across the core using VPNv4. | 
| **RD / RT Usage** | RD is optional (mostly local). RT is NOT used. | Both RD and RT are **Mandatory**. | 
| **Core Network** | Every router in the path must be configured with the VRFs. | Core (P) routers don't know about VRFs, they just switch MPLS labels. | 

> 💡 **Crucial Concepts for Full VRF:**
>
> * **RD (Route Distinguisher):** Makes overlapping IP addresses mathematically unique (e.g., `65000:10`). It is purely local to the router.
>
> * **RT (Route Target):** An extended BGP community that controls which routes are imported into, and exported from, a specific VRF.

---

## 4. Configuration Scenario A: VRF Lite (Enterprise)

**Scenario:** You have an Enterprise Edge router connecting to a Core Switch. You need to completely separate the **HR** department traffic from the **GUEST** traffic. There is NO MPLS and NO BGP here.

### Step 1: Define the VRFs

**Option A: Modern Syntax (IPv4 & IPv6 - Recommended)**
Create the VRFs and explicitly activate the IPv4 address family for each.

```cisco
Router(config)# vrf definition HR_DEPT
Router(config-vrf)# address-family ipv4
Router(config-vrf-af)# exit-address-family

Router(config)# vrf definition GUEST_WIFI
Router(config-vrf)# address-family ipv4
Router(config-vrf-af)# exit-address-family
```

**Option B: Legacy Syntax (IPv4 Only)**
If you are on an older IOS, use the `ip vrf` command instead.

```cisco
Router(config)# ip vrf HR_DEPT
Router(config)# ip vrf GUEST_WIFI
```

### Step 2: Assign Interfaces

> ⚠️ **DANGER:** When you assign an interface to a VRF, **it instantly drops its current IP address**. Configure the VRF assignment *before* configuring the IP.

**Assigning the HR Interface (Modern Syntax):**

```cisco
Router(config)# interface GigabitEthernet0/1.10
Router(config-subif)# encapsulation dot1Q 10
Router(config-subif)# vrf forwarding HR_DEPT
Router(config-subif)# ip address 10.1.1.1 255.255.255.0
```

*(Note: If using Legacy syntax, the command would be `ip vrf forwarding HR_DEPT`)*

**Assigning the GUEST Interface:**

```cisco
Router(config)# interface GigabitEthernet0/1.20
Router(config-subif)# encapsulation dot1Q 20
Router(config-subif)# vrf forwarding GUEST_WIFI
Router(config-subif)# ip address 192.168.1.1 255.255.255.0
```

### Step 3: Routing (Independent OSPF Processes)

Because the tables are separate, you run separate routing processes tied to specific VRFs.

**Configure OSPF Process 10 dedicated to the HR VRF:**

```cisco
Router(config)# router ospf 10 vrf HR_DEPT
Router(config-router)# network 10.1.1.0 0.0.0.255 area 0
```

**Configure OSPF Process 20 dedicated to the GUEST VRF:**

```cisco
Router(config)# router ospf 20 vrf GUEST_WIFI
Router(config-router)# network 192.168.1.0 0.0.0.255 area 0
```

---

## 5. Configuration Scenario B: Full VRF / MPLS L3VPN (Service Provider)

**Scenario:** You are configuring a Provider Edge (PE) router in an ISP. You are connecting "Customer A". You must use an **RD** and **RTs**.

### Step 1: Define the VRF with RD and RT (Mandatory)

Create the VRF, define the Route Distinguisher (usually `ASN:CustomerNumber`), and set the Route Targets for importing and exporting BGP routes.

```cisco
Router(config)# vrf definition CUST_A
Router(config-vrf)# rd 65000:101
Router(config-vrf)# route-target export 65000:101
Router(config-vrf)# route-target import 65000:101
Router(config-vrf)# address-family ipv4
Router(config-vrf-af)# exit-address-family
```

### Step 2: Assign Interface connected to Customer A

Bind the interface facing Customer A to the VRF and assign the IP address.

```cisco
Router(config)# interface GigabitEthernet0/2
Router(config-if)# vrf forwarding CUST_A
Router(config-if)# ip address 172.16.1.1 255.255.255.0
Router(config-if)# no shutdown
```

### Step 3: MP-BGP Configuration (The SP Backbone)

Establish a standard BGP peering with another PE router in the core, activate the **VPNv4** address family to exchange VRF routes, and redistribute Customer A's connected networks into BGP.

```cisco
Router(config)# router bgp 65000
Router(config-router)# neighbor 10.255.255.2 remote-as 65000
Router(config-router)# neighbor 10.255.255.2 update-source Loopback0

Router(config-router)# address-family vpnv4
Router(config-router-af)# neighbor 10.255.255.2 activate
Router(config-router-af)# neighbor 10.255.255.2 send-community extended
Router(config-router-af)# exit-address-family

Router(config-router)# address-family ipv4 vrf CUST_A
Router(config-router-af)# redistribute connected
Router(config-router-af)# exit-address-family
```

---

## 6. Route Leaking via Route Replication (Modern Approach)

By default, VRFs are completely isolated. However, isolated environments often need access to **Shared Services** (like DNS, DHCP, or monitoring servers) located in a different VRF.

Allowing traffic between them is called **Route Leaking**. Historically, this required complex BGP configurations (using Route-Targets), even in non-MPLS Enterprise networks. Cisco introduced **Route Replication** to simplify this process without the need for BGP.

* **How it works:** It directly copies (leaks) routes from one VRF's routing table into another.
* **Routing Table Indicator:** Replicated routes show up with a `+` sign when viewing the routing table (e.g., `O+` for OSPF).

### Route Replication Configuration Example

**Scenario:** We want to leak OSPF routes from the `USERS` VRF into the `SERVICES` VRF.

Enter the target VRF (where you want the routes to go), and use the `route-replicate` command to pull the routes from the source VRF.

```cisco
Router(config)# vrf definition SERVICES
Router(config-vrf)# address-family ipv4
Router(config-vrf-af)# route-replicate from vrf USERS unicast ospf 1
```

> 📌 **Note:** You can specify `static`, `connected`, `all`, or attach a `route-map` for precise filtering instead of just blindly copying all routes from `ospf 1`.

---

## 7. Verification & Troubleshooting

Because the routing tables are now virtualized and separated, standard `show ip route` or `ping` commands will only look at the **Global Routing Table**. You must specify the VRF in your commands!

**View all configured VRFs and their assigned interfaces (Legacy: `show ip vrf`):**

```cisco
Router# show vrf
```

**View detail including RD and RTs (Crucial for Full VRF troubleshooting):**

```cisco
Router# show vrf detail CUST_A
```

**View the specific routing table for a VRF (Look for '+' for replicated routes):**

```cisco
Router# show ip route vrf HR_DEPT
```

**Pinging and Tracerouting from a specific VRF:**

```cisco
Router# ping vrf HR_DEPT 10.1.1.100
Router# traceroute vrf CUST_A 172.16.1.100
```

**Check the BGP VPNv4 routing table (Full VRF Only):**

```cisco
Router# show bgp vpnv4 unicast all
```
