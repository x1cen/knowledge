# VLAN Access Map (VACL)

## 1. What exactly is a VACL?

Normally, we use standard Access Control Lists (Router ACLs or RACLs) to filter traffic *between* different networks when it passes through a router. But what if you want to block two computers from talking to each other when they are in the **exact same VLAN**?

Since they are on the same subnet, their traffic never hits the router. This is where a **VLAN Access Map (VACL)** comes in. A VACL filters traffic purely at Layer 2, directly inside the switch, before it ever reaches a gateway.

* **The Logic:** VACLs work exactly like Route-Maps. You create a sequence of rules. Each rule has a **Match** condition (what traffic to look for) and an **Action** (usually Forward or Drop).

> ⚠️ **DANGER:** Just like normal ACLs and Route-Maps, every VACL has an **Implicit Deny** at the very end. If you do not explicitly tell the switch to forward the rest of the traffic, it will drop ALL communications inside that VLAN!

## 2. Configuration Scenario: IP-Based VACL

**The Scenario:** We have an Accounting Server (192.168.10.50) and an Infected PC (192.168.10.99) sitting in **VLAN 10**. We want to completely block the Infected PC from talking to the Server, but let all other devices in VLAN 10 communicate normally.

### Step 1: Define the traffic using an ACL

First, we create an ACL to identify the traffic. Note that the word `permit` here does not mean "allow the traffic to pass". It means "select this traffic so the VACL can take action on it".

```
Router(config)# ip access-list extended MATCH_INFECTED_PC
Router(config-ext-nacl)# permit ip host 192.168.10.99 host 192.168.10.50
Router(config-ext-nacl)# permit ip host 192.168.10.50 host 192.168.10.99
Router(config-ext-nacl)# exit

```

### Step 2: Create the VACL and apply the Drop action

Now we create the VLAN Access Map (Sequence 10), match our ACL, and tell the switch to drop it.

```
Router(config)# vlan access-map SECURE_VLAN10 10
Router(config-access-map)# match ip address MATCH_INFECTED_PC
Router(config-access-map)# action drop
Router(config-access-map)# exit

```

### Step 3: Prevent the Implicit Deny (Crucial)

We create a new sequence (Sequence 20). Because we do not provide a `match` statement, the switch assumes we mean "match everything else". We set the action to forward.

```
Router(config)# vlan access-map SECURE_VLAN10 20
Router(config-access-map)# action forward
Router(config-access-map)# exit

```

### Step 4: Apply the VACL to the VLAN

Finally, we apply this security map to VLAN 10 globally.

```
Router(config)# vlan filter SECURE_VLAN10 vlan-list 10

```

## 3. Configuration Scenario: MAC-Based VACL

Sometimes you need to filter non-IP traffic (like legacy IPX) or you simply want to block a specific device by its physical address regardless of its IP. You can use a MAC ACL for this.

### Step 1: Create the MAC ACL

```
Router(config)# mac access-list extended MATCH_MAC_ADDRESS
Router(config-ext-macl)# permit host 0000.1111.2222 any
Router(config-ext-macl)# exit

```

### Step 2: Apply it in the VACL

Notice that we use `match mac address` instead of `match ip address`.

```
Router(config)# vlan access-map BLOCK_MAC 10
Router(config-access-map)# match mac address MATCH_MAC_ADDRESS
Router(config-access-map)# action drop
Router(config-access-map)# exit

Router(config)# vlan access-map BLOCK_MAC 20
Router(config-access-map)# action forward
Router(config-access-map)# exit
```

### Step 3: Apply the filter

```
Router(config)# vlan filter BLOCK_MAC vlan-list 20
```

## 4. Advanced Logic: AND / OR Operations in VACLs

When writing complex security policies, you might need to combine multiple conditions. VACLs handle boolean logic (AND/OR) based on how you structure your `match` statements within a sequence.

### The OR Logic (Same Line)

If you apply multiple ACLs on the *same* match line, the switch treats it as an **OR** condition. If the traffic matches `ACL_A` **OR** `ACL_B`, the action is taken.

```
Router(config)# vlan access-map ADVANCED_VACL 10
Router(config-access-map)# match ip address ACL_A ACL_B
Router(config-access-map)# action drop
Router(config-access-map)# exit
```

*(Also remember: The sequences themselves act as a Top-Down OR. If Seq 10 doesn't match, it checks Seq 20, then Seq 30).*

### The AND Logic (Different Lines)

To create an **AND** condition within a single sequence block, you must match different protocol types (e.g., an IP ACL and a MAC ACL) on *separate* lines. The traffic must match BOTH conditions to trigger the action.

```
Router(config)# vlan access-map ADVANCED_VACL 20
Router(config-access-map)# match ip address MATCH_IP_ACL
Router(config-access-map)# match mac address MATCH_MAC_ACL
Router(config-access-map)# action forward
Router(config-access-map)# exit
```

*In this specific sequence, a packet is only forwarded if its IP matches the IP ACL **AND** its MAC address matches the MAC ACL.*

## 5. Verification & Troubleshooting

Use these commands to ensure your VACL is correctly configured and applied.

**View the contents of your Access Maps (Shows sequences, matches, and actions):**

```
Router# show vlan access-map
```

**Verify which VACL is applied to which VLAN:**

```
Router# show vlan filter
```

**Check statistics for matched packets (Available on newer switches and Nexus platforms):**

```
Router# show vlan access-map statistics
```
