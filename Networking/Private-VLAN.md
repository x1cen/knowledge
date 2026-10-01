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

## 4. Configuration Scenario

**The Scenario:** We are setting up a Data Center.

* Primary VLAN is 100.

* We want an Isolated VLAN (VLAN 101) for public web servers (they shouldn't talk to each other).

* We want a Community VLAN (VLAN 102) for database servers (they need to sync with each other).

* The router is connected to GigabitEthernet0/1.

### Step 1: VTP Transparent Mode (Crucial Prerequisite)

Historically, PVLANs require VTP to be set to Transparent mode (or VTP Version 3). If you don't do this, the switch might reject PVLAN commands.

```
Switch(config)# vtp mode transparent
```

### Step 2: Create the Secondary VLANs

First, we create the children and define their types.

```
! Create the Isolated VLAN
Switch(config)# vlan 101
Switch(config-vlan)# private-vlan isolated
Switch(config-vlan)# exit

! Create the Community VLAN
Switch(config)# vlan 102
Switch(config-vlan)# private-vlan community
Switch(config-vlan)# exit
```

### Step 3: Create the Primary VLAN and Map the Children

Now we create the parent (VLAN 100) and link the secondary VLANs to it.

```
Switch(config)# vlan 100
Switch(config-vlan)# private-vlan primary
Switch(config-vlan)# private-vlan association 101,102
Switch(config-vlan)# exit
```

### Step 4: Configure the Host Ports

Let's assign ports to our servers. Notice how we have to specify BOTH the primary and secondary VLANs for each port.

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

### Step 5: Configure the Promiscuous Port

This is the uplink to the Router. We must tell it to map to the Primary VLAN and all associated Secondary VLANs so traffic can flow in both directions.

```
Switch(config)# interface GigabitEthernet0/1
Switch(config-if)# switchport mode private-vlan promiscuous
Switch(config-if)# switchport private-vlan mapping 100 101,102
```

## 5. Layer 3 Configuration (SVI) for PVLAN

If the switch itself is doing the Layer 3 routing (it is a Core/Distribution switch acting as the gateway), you do not configure a physical Promiscuous port. Instead, you configure the SVI (Interface VLAN) of the **Primary VLAN** and map the secondary VLANs to it.

```
Switch(config)# interface Vlan 100
Switch(config-if)# ip address 10.0.0.254 255.255.255.0
Switch(config-if)# private-vlan mapping 101,102
```

*(Note: You NEVER create Layer 3 SVIs for secondary VLANs. The Primary VLAN SVI handles routing for all of them).*

## 6. Verification & Show Commands

Use these commands to verify your mappings and port assignments.

**See a clean table of your Primary, Secondary, and assigned ports:**

```
Switch# show vlan private-vlan
```

**Verify the exact switchport configuration for a specific interface:**

```
Switch# show interfaces GigabitEthernet0/10 switchport
```
