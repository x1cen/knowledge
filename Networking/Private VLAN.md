# Private VLAN (PVLAN)

## 1. What exactly is a Private VLAN?

Normally, if you put ten computers in the same VLAN (and the same IP subnet), they can all ping and talk to each other freely. But what if you want them in the same subnet to save IP addresses, but you DO NOT want them to talk to each other?

Think of a hotel network. All guests connect to "VLAN 10" and get an IP like `192.168.10.x`. They all need to reach the internet (the default gateway), but Guest A should absolutely NOT be able to scan or hack Guest B's laptop.

This is where **Private VLANs (PVLAN)** come in. PVLANs allow you to split a single normal VLAN into smaller, isolated sub-VLANs at Layer 2, preventing devices in the same IP subnet from communicating directly.

## 2. The Core Components: VLAN Types

To make this work, PVLAN uses a "Parent-Child" relationship.

**1. Primary VLAN (The Parent):**
This is the main VLAN that represents the entire IP subnet. It is responsible for forwarding traffic downstream from the router (the gateway) to all the isolated hosts.

**2. Secondary VLANs (The Children):**
These are the sub-VLANs where the actual computers are placed. There are two types of Secondary VLANs:

* **Isolated VLAN:** The ultimate lockdown. Devices in an Isolated VLAN cannot talk to ANYONE else in the VLAN, not even other devices in the exact same Isolated VLAN. They can only talk to the gateway. (Usually, you only need ONE Isolated VLAN per Primary VLAN).

* **Community VLAN:** A mini-group. Devices in a Community VLAN can talk to each other, and they can talk to the gateway. However, they CANNOT talk to devices in other Community VLANs or Isolated VLANs.

## 3. The Core Components: Port Roles

You also have to define the role of the physical switch ports.

**1. Promiscuous Port (P-Port):**
Think of this as the "God Port". It is allowed to talk to absolutely every port in the PVLAN (both Isolated and Community). You almost always connect this port to your Default Gateway (Router/Firewall).

**2. Host Port:**
These are the ports connected to your end devices (PCs, Servers). A Host Port is further divided into:

* **Isolated Port:** Belongs to an Isolated VLAN.

* **Community Port:** Belongs to a Community VLAN.

## 4. Configuration Scenario: Single Switch

**The Scenario:** We are setting up a Data Center on a single switch.
* Primary VLAN is 100.
* Isolated VLAN is 101 (for public web servers).
* Community VLAN is 102 (for database servers).

### Step 1: VTP Configuration

Historically (VTP v1 and v2), switches did not understand PVLANs and would drop the configuration unless the switch was in Transparent mode. However, **VTP Version 3 officially supports PVLANs**. If you run VTPv3, the PVLAN structure will be advertised and synchronized across your network!

```
! Legacy environments (VTP v1/v2):
Switch(config)# vtp mode transparent

! Modern environments (VTP v3):
Switch(config)# vtp version 3
```

### Step 2: Create the Secondary VLANs

First, we create the children and define their types.

```
Switch(config)# vlan 101
Switch(config-vlan)# private-vlan isolated
Switch(config-vlan)# exit

Switch(config)# vlan 102
Switch(config-vlan)# private-vlan community
Switch(config-vlan)# exit
```

### Step 3: Create the Primary VLAN and Map the Children

```
Switch(config)# vlan 100
Switch(config-vlan)# private-vlan primary
Switch(config-vlan)# private-vlan association 101,102
Switch(config-vlan)# exit
```

### Step 4: Configure the Host Ports

Notice how we have to specify BOTH the primary and secondary VLANs for each port.

```
! Assigning a port to the Web Server (Isolated)
Switch(config)# interface GigabitEthernet0/10
Switch(config-if)# switchport mode private-vlan host
Switch(config-if)# switchport private-vlan host-association 100 101

! Assigning ports to Database Servers (Community)
Switch(config)# interface range GigabitEthernet0/20 - 21
Switch(config-if)# switchport mode private-vlan host
Switch(config-if)# switchport private-vlan host-association 100 102
```

### Step 5: Configure the Promiscuous Port (Uplink to Router)

```
Switch(config)# interface GigabitEthernet0/1
Switch(config-if)# switchport mode private-vlan promiscuous
Switch(config-if)# switchport private-vlan mapping 100 101,102
```

## 5. Extending PVLANs Across Multiple Switches (Trunking)

What if you have Web Servers in Isolated VLAN 101 on Switch A, and more Web Servers in the same Isolated VLAN 101 on Switch B? How do you connect them?

**The Golden Rule:** PVLAN is an **INTERNAL (Locally Significant)** feature. When a switch sends a frame out of a trunk port, it does NOT add any special "PVLAN magic tags". It simply tags the frame with standard 802.1Q tags using the Secondary VLAN ID (e.g., VLAN 101).

Because of this, configuring a trunk between two PVLAN-enabled switches is incredibly simple. You just configure a normal 802.1Q trunk, but you MUST allow both the Primary and all Secondary VLANs across it.

```
! On both Switch A and Switch B
Switch(config)# interface GigabitEthernet0/24
Switch(config-if)# switchport trunk encapsulation dot1q
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk allowed vlan 100,101,102
```
*Note: Some newer Cisco switches support `switchport mode private-vlan trunk promiscuous`, but standard 802.1Q trunks work perfectly fine for extending PVLANs between switches.*

## 6. The Danger of Mismatched Trunks (The Security Edge Case)

Because PVLAN is an internal feature (and sends standard 802.1Q tags on the wire), a dangerous security breach can happen if your network is misconfigured.

**The Scenario:**
*   **Switch A:** Has PVLAN configured. Port 1 is in Isolated VLAN 101.
*   **Switch B:** Has NO PVLAN configured. The admin just created VLAN 101 as a standard, normal VLAN.
*   **The Link:** A standard trunk connects them.

**What happens?**
1.  A server on Switch A (Isolated) sends a broadcast or multicast frame.
2.  Switch A knows it's isolated, so it prevents it from going to other local isolated ports, but it **DOES** forward it out the trunk, tagged with VLAN 101.
3.  Switch B receives the frame tagged as VLAN 101.
4.  Switch B looks at its database. It sees VLAN 101 as a *normal* VLAN. It has no idea it's supposed to be isolated.
5.  **The Result:** Switch B forwards the frame to ALL ports in VLAN 101. The isolation boundary is completely destroyed! Devices on Switch B can now freely talk to each other, and traffic originating from Switch A just reached multiple hosts it wasn't supposed to.

**The Lesson:** PVLAN configuration MUST be identical end-to-end across all switches in the Layer 2 domain to maintain security.

## 7. Layer 3 Configuration (SVI) for PVLAN

If the switch itself is doing the Layer 3 routing (Core/Distribution switch), you configure the SVI of the **Primary VLAN** and map the secondary VLANs to it. You NEVER create Layer 3 SVIs for secondary VLANs.

```
Switch(config)# interface Vlan 100
Switch(config-if)# ip address 10.0.0.254 255.255.255.0
Switch(config-if)# private-vlan mapping 101,102
```

## 8. Verification & Show Commands

**See a clean table of your Primary, Secondary, and assigned ports:**
```
Switch# show vlan private-vlan
```

**Verify the exact switchport configuration for a specific interface:**
```
Switch# show interfaces GigabitEthernet0/10 switchport
```
