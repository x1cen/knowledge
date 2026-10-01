It’s a networking mechanism used to provide **default-gateway redundancy**. The idea is simple: if a router acting as the default gateway fails, another router can take over **without the hosts needing to change their default gateway**.

And this is why FHRP is especially important in a VLAN environment: having two redundant routers is useless if hosts still depend on one physical router as their only gateway.

| Protocol                               | Abbreviation | Vendor | Versions / Standards                 |
| -------------------------------------- | ------------ | ------ | ------------------------------------ |
| **Hot Standby Router Protocol**        | **HSRP**     | Cisco  | v1, v2                               |
| **Virtual Router Redundancy Protocol** | **VRRP**     | IETF   | v2 (**RFC 3768**), v3 (**RFC 5798**) |
| **Gateway Load Balancing Protocol**    | **GLBP**     | Cisco  | GLBP                                 |

>If you can stack the devices that are going to act as the gateway, you can skip implementing this protocol, because stacking makes the devices operate as a single logical device.

## How does FHRP work?

The entire FHRP process is based on the use of a **Virtual MAC address**. In practice, when you configure a virtual IP address as the default gateway, a host sends an **ARP request** for that virtual IP. Within the FHRP group, the virtual IP is associated with a specific router, which is the **Active** router, and that router is responsible for replying to the ARP request with the corresponding Virtual MAC address. The host then uses this Virtual MAC as the destination MAC when sending traffic to the default gateway. **GLBP** works slightly differently because it provides load balancing by design: each participating router can have its own Virtual MAC address, and the Active Virtual Gateway (AVG) distributes these Virtual MAC addresses among the hosts. In contrast, **HSRP and VRRP** normally use a single Virtual MAC address for the FHRP group, which is actively owned and used by the current Active/Master router.

## HSRP - Host Standby Router Protocol

**HSRP (Hot Standby Router Protocol)** is a Cisco proprietary FHRP that provides **default gateway redundancy** by using a virtual IP and MAC address shared between routers. One router is **Active**, while another acts as **Standby** and takes over if the Active router fails.

HSRP has three roles: **Active, Standby, and Listen**. Traffic is handled by the **Active** router, while the **Standby** router takes over if the Active router fails.

#### HSRP Setup

Under the interface configuration, you can enable HSRP using the following command. Make sure that the virtual IP address is unique within the network, and that both the **group number** and the **virtual IP address** are the same on all routers participating in the HSRP group.

```cisco
SW(config-if)# standby <group-number> ip <ip-address> [secondary]
```

In HSRP, you can configure two virtual IP addresses for the same HSRP group, as shown below:

```cisco
SW(config-if)# standby 1 ip 192.168.0.3
```

```cisco
SW(config-if)# standby 1 ip 192.168.0.4 secondary
```

**On a single interface, you can configure multiple HSRP groups, and the same group number can be used across different interfaces.**

#### HSRP Preemption

This command is used to enable the **preemption** feature, which is disabled by default. The **preemption delay** values can also be configured, as explained above.

```cisco
SW(config-if)# standby <group-number> preempt 
```

```cisco
SW(config-if)# standby <group-number> preempt delay minimum <value> reload <value>
```

When you configure a preemption delay, **preemption is enabled automatically by default**, so you don't need to enter a separate command to enable it.

If a switch does not have preemption enabled, it cannot take over the Active role even if it has a higher priority. It can become Active only after the current Active router fails.

#### HSRP Priority

This command is used to change the HSRP priority. The router with the highest priority becomes the Active router, provided that **preemption** is enabled. The default priority value is **100**.

```cisco
SW(config-if)# standby <group-number> priority <priority-value>
```

You may be wondering how the Active router is selected when the priority values are equal. In that case, the router with the **highest IP address** is elected as the Active router.

#### HSRP Authentication

```cisco
SW(config-if)# standby group-number authentication text <text-string>
```

```cisco
SW(config-if)# standby group-number authentication md5 key-string <key-string> 
```

```cisco
SW(config-if)# standby group-number authentication md5 key-chain <key-chain-name>
```

#### HSRP Timers

By default, HSRP hello packets are sent every 3 seconds using UDP port 1985 to the multicast address `224.0.0.2`, and the hold time is 10 seconds. You can modify these timers as needed.

```cisco
SW(config-if)# standby <group-number> timers <hello-time> <hold-time>
```

```cisco
 SW(config-if)# standby <group-number> timers msec <hello-time> msec <hold-time>
```

#### HSRP Verify

```cisco
SW# show standby
```

```cisco
SW# show standby [type mode/num]
```

```cisco
SW# show standby brief [all]
```

```cisco
SW# show standby neighbors [type mode/num]
```

```cisco
SW# show standby delay [type mode/num]
```

The **Minimum** delay applies when the interface comes up, while the **Reload** delay applies after the entire router reboots. For example, `Minimum 1` means HSRP waits at least 1 second after the interface becomes active, while `Reload 5` means HSRP waits 5 seconds after the router reloads before starting the HSRP process.

#### HSRP Track Object 

You can configure multiple tracking objects for HSRP without a specific limit. When a tracked object goes down, the configured decrement value is subtracted from the HSRP priority.

```cisco
SW(config-if)# standby <group-number> track <object-number> decrement <value>
```

```cisco
SW(config-if)# standby <group-number> track <object-number> shutdown
```

#### HSRP Virtual MAC

```cisco
SW(config-if)# standby <group-number> mac-address <value>
```

#### HSRP Configuration Based on STP

When we use HSRP, we typically have two gateways connected through a Layer 2 infrastructure. Wherever a Layer 2 network exists, **STP** is also used to prevent loops and manage the Layer 2 topology. Therefore, the HSRP Active router should ideally align with the **STP Root Bridge** for the corresponding VLAN. Otherwise, traffic may take inefficient paths through the Layer 2 network. So, the first step is to check your **STP tuning** and determine which switch is the Root Bridge for each VLAN. Ideally, that same switch should also be the HSRP Active router for that VLAN. Here, we are specifically talking about the **network/core switches** acting as the gateways.

#### Differences between HSRPv1 and HSRPv2

The main differences between HSRPv1 and HSRPv2 are the group ID range, multicast address, virtual MAC address format, and IPv6 support.

|Feature|HSRPv1|HSRPv2|
|---|---|---|
|Group IDs|`0–255`|`0–4095`|
|Multicast Address|`224.0.0.2`|`224.0.0.102`|
|Virtual MAC|`0000.0C07.ACXX`|`0000.0C9F.FXXX`|
|IPv6 Support|No|Yes|
|Scalability|Limited|Improved|
|Compatibility|Legacy|Newer implementations|

```cisco
SW(config-if)# standby <group-number> version 2
```

## VRRP - Virtual Router Redundancy Protocol

**VRRP (Virtual Router Redundancy Protocol)** is an open-standard FHRP that provides default gateway redundancy. Multiple routers share a virtual IP address, with one router acting as the Master and the others as Backup routers. If the Master fails, a Backup router takes over the Master role.

VRRP has two main roles: the **Master** router, which handles traffic for the virtual IP address, and the **Backup** routers, which monitor the Master and take over its role if it fails.

#### VRRP Setup

Under the interface configuration, you can enable VRRP using the following command. Make sure that the virtual IP address is unique within the network, and that both the **group number** and the **virtual IP address** are the same on all routers participating in the VRRP group.

```cisco
SW(config-if)# vrrp <group-number> ip <ip-address> [secondary]
```

In VRRP, you can configure two virtual IP addresses for the same VRRP group, as shown below:

```cisco
SW(config-if)# vrrp 1 ip 192.168.0.3
```

```v
SW(config-if)# vrrp 1 ip 192.168.0.4 secondary
```

**On a single interface, you can configure multiple VRRP groups, and the same group number can be used across different interfaces.**

#### VRRP Preemption

Preemption is enabled by default in VRRP, so you do not need to enter the following command.

```cisco
SW(config-if)# vrrp <group-number> preempt 
```

Using the following command, you can specify how long a router should wait before becoming the Master when it has the required priority to take over. This delay gives the router enough time to fully initialize its interfaces and routing protocols, helping prevent unnecessary state changes and instability.

```cisco
SW(config-if)# vrrp <group-number> preempt delay minimum <value>
```

When you configure a preemption delay, **preemption is enabled automatically by default**, so you don't need to enter a separate command to enable it.
#### VRRP Priority

This command is used to change the VRRP priority. The router with the highest priority becomes the Master router, provided that **preemption** is enabled. The default priority value is **100**.

```cisco
SW(config-if)# vrrp <group-number> priority <priority-value>
```

You may be wondering how the Master router is selected when the priority values are equal. In that case, the router with the **highest IP address** is elected as the Master router.

#### VRRP Authentication

**Authentication is only supported in VRRPv2.**

```cisco
SW(config-if)# vrrp group-number authentication text <text-string>
```

#### VRRP Timers

By default, VRRP advertisements are sent every 1 second to the multicast address `224.0.0.18` using IP protocol 112. The default Master Down Interval is ~3 seconds.

```cisco
SW(config-if)# vrrp group-number timers advertise <value>
```

```cisco
SW(config-if)# vrrp group-number timers advertise msec <value>
```

With the commands above, you can change the Advertisement Interval. By default, it is set to 1 second. The Hold Time is calculated automatically using the following formula:

$3 * (hello - interval) + (256 - priority) / 256$

```cisco
SW(config-if)# vrrp group-number timers learn
```

The `timers learn` command allows a Backup router to learn the Advertisement Interval from the Master and use the same timer value. For example, if the Master sends advertisements every 5 seconds, the Backup router will also use a 5-second interval.

#### VRRP Track Object 

You can configure multiple tracking objects for VRRP without a specific limit. When a tracked object goes down, the configured decrement value is subtracted from the VRRP priority.

```cisco
SW(config-if)# vrrp <group-number> track <object-number> decrement <value>
```

#### VRRP Verify

```cisco
SW# show vrrp
```

```cisco
SW# show vrrp brief [all]
```

```cisco
SW# show vrrp interface type mode/num [all | brief | group group-number]
```

#### VRRP Configuration Based on STP

When we use VRRP, we typically have two gateways connected through a Layer 2 infrastructure. Wherever a Layer 2 network exists, **STP** is also used to prevent loops and manage the Layer 2 topology. Therefore, the VRRP Master router should ideally align with the **STP Root Bridge** for the corresponding VLAN. Otherwise, traffic may take inefficient paths through the Layer 2 network. So, the first step is to check your **STP tuning** and determine which switch is the Root Bridge for each VLAN. Ideally, that same switch should also be the VRRP Master router for that VLAN. Here, we are specifically talking about the **network/core switches** acting as the gateways.
#### Differences between VRRPv2 and VRRPv3

VRRP has two main versions. VRRPv2 was designed for IPv4 and is defined in RFC 3768, while VRRPv3 supports both IPv4 and IPv6 and is defined in RFC 5798. VRRPv3 also provides improvements such as support for larger address configurations and more flexible operation compared to VRRPv2.

| Feature          | VRRPv2                  | VRRPv3                                  |
| ---------------- | ----------------------- | --------------------------------------- |
| Standard         | RFC 3768                | RFC 5798                                |
| IPv4             | Yes                     | Yes                                     |
| IPv6             | No                      | Yes                                     |
| Address Support  | IPv4 only               | IPv4 and IPv6                           |
| VRID Range       | `1–255`                 | `1–255`                                 |
| Multicast (IPv4) | `224.0.0.18`            | `224.0.0.18`                            |
| Authentication   | Supported               | Removed from the protocol specification |
| Main Purpose     | IPv4 gateway redundancy | IPv4/IPv6 gateway redundancy            |
#### VRRPv3 Setup

```cisco
SW(config)# fhrp version vrrp v3
```

```cisco
SW(config-if)# vrrp <group-number> address-family <ipv4 | ipv6>
SW(config-if)# address <ip> 
SW(config-if)# priority <value>
SW(config-if)# track <object-number> decrement <value>
```

```cisco
SW# show fhrp version
```

## GLBP - Gateway Load Balancing Protocol 

GLBP (Gateway Load Balancing Protocol) is a Cisco proprietary FHRP that provides both gateway redundancy and load balancing. Unlike HSRP and VRRP, which normally use a single virtual MAC address for the group, GLBP allows multiple routers to actively forward traffic at the same time by assigning a different virtual MAC address to each forwarding router.

GLBP uses two main roles: the Active Virtual Gateway (AVG) and Active Virtual Forwarders (AVFs). The AVG is responsible for managing the GLBP group, responding to ARP requests for the virtual IP address, and assigning virtual MAC addresses to the AVFs. The AVFs then forward the actual traffic from hosts to the destination. This allows GLBP to distribute traffic across multiple gateways instead of keeping one router active while the others remain idle.

When a host sends an ARP request for the virtual gateway IP, the AVG responds with a virtual MAC address belonging to one of the AVFs. Different hosts can receive different virtual MAC addresses, allowing their traffic to be distributed across multiple routers.

In GLBP, the priority value is used only to determine the AVG for the network. The weight is used to determine whether a switch can become an AVF.


#### GLBP Setup

Under the interface configuration, you can enable GLBP using the following command. Make sure that the virtual IP address is unique within the network, and that both the **group number** and the **virtual IP address** are the same on all routers participating in the GLBP group.

```cisco
SW(config-if)# glbp <group-number> ip <ip-address> [secondary]
```

In GLBP, you can configure two virtual IP addresses for the same GLBP group, as shown below:

```cisco
SW(config-if)# glbp 1 ip 192.168.0.3
```

```cisco
SW(config-if)# glbp 1 ip 192.168.0.4 secondary
```

**On a single interface, you can configure multiple GLBP groups, and the same group number can be used across different interfaces.**

#### GLBP Preemption

Preemption is disabled by default in GLBP.

```cisco
SW(config-if)# glbp <group-number> preempt 
```

Using the following command, you can specify how long a router should wait before becoming the AVG when it has the required priority to take over. This delay gives the router enough time to fully initialize its interfaces and routing protocols, helping prevent unnecessary state changes and instability.

```cisco
SW(config-if)# glbp <group-number> preempt delay minimum <value>
```

When you configure a preemption delay, **preemption is enabled automatically by default**, so you don't need to enter a separate command to enable it.

#### GLBP Priority

This command is used to change the GLBP priority. The router with the highest priority becomes the AVG router, provided that **preemption** is enabled. The default priority value is **100**.

```cisco
SW(config-if)# glbp <group-number> priority <priority-value>
```

You may be wondering how the AVG router is selected when the priority values are equal. In that case, the router with the **highest IP address** is elected as the AVG router.

#### GLBP Authentication

```cisco
SW(config-if)# glbp group-number authentication text <text-string>
```

```cisco
SW(config-if)# glbp group-number authentication md5 key-string <key-string> 
```

```cisco
SW(config-if)# glbp group-number authentication md5 key-chain <key-chain-name>
```

#### GLBP Timers

By default, GLBP Hello packets are sent every 3 seconds using UDP port 3222 to the multicast address `224.0.0.102`, and the default Hold Time is 10 seconds.

```cisco
SW(config-if)# glbp <group-number> timers <hello-time> <hold-time>
```

```cisco
 SW(config-if)# glbp <group-number> timers msec <hello-time> msec <hold-time>
```

```cisco
 SW(config-if)# glbp <group-number> timers redirect <redirect-time> <timeout>
```

The `redirect-time` specifies how long the AVG waits before redirecting hosts to a different virtual MAC address when a forwarding path changes. The `timeout` specifies how long the old virtual MAC address remains valid after the redirect. During this period, another AVF can continue forwarding traffic sent to the old virtual MAC address, allowing hosts that still have the old MAC address in their ARP cache to continue sending traffic without disruption. For example, with `glbp 1 timers redirect 600 7200`, the AVG can redirect hosts to a new virtual MAC after 600 seconds, while the old virtual MAC remains valid for up to 7200 seconds (2 hours).

#### GLBP weighting

The switches responsible for forwarding traffic are the AVFs. Therefore, to determine which AVFs can be active, we use the weighting mechanism by configuring the appropriate thresholds.

```cisco
SW(config-if)# glbp <g-number> weighting <value> lower <value> upper <value> 
```

```
SW(config-if)# glbp <g-number> weighting track <object-number> decrement <value> 
```

```cisco
SW(config-if)# glbp <g-number> forwarder preempt [delay minimum <seconds>]
```

#### GLBP Load Balancing Method

**Round-Robin:** The AVG distributes virtual MAC addresses sequentially among the available AVFs. As new hosts send ARP requests, they are assigned to different AVFs in a rotating order, providing a relatively even distribution of traffic.

**Weighted:** The AVG distributes traffic based on the weight of each AVF. AVFs with higher weights receive a larger share of the traffic, while AVFs with lower weights receive a smaller share.

**Host-Dependent:** The AVG assigns a specific virtual MAC address to each host and keeps that assignment consistent. This ensures that a host continues to use the same AVF as long as that AVF remains available.

```cisco
SW(config-if)# glbp <g-number> load-balancing [host-dependent | round-robin | weighted]
```

#### GLBP Verify

```cisco
SW# show glbp [type mode/num]
```

```cisco
SW# show glbp brief [all]
```

