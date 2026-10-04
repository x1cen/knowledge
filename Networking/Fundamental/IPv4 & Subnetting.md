# Networking and IPv4 Addressing Notes

## 1. IPv4 Address Structure & Classes

An IPv4 address is a 32-bit address represented in four octets (`X.X.X.X` where `X => 0-255`). Each octet consists of 8 bits.

**Note:** The total number of possible IPv4 addresses is $2^{32}$.

### Legacy IPv4 Classes

* **Class A:** `1 - 126` $\rightarrow$ Default Subnet Mask: `255.0.0.0` (`/8`)
* **Class B:** `128 - 191` $\rightarrow$ Default Subnet Mask: `255.255.0.0` (`/16`)
* **Class C:** `192 - 223` $\rightarrow$ Default Subnet Mask: `255.255.255.0` (`/24`)
* **Class D:** `224 - 239` $\rightarrow$ Used for **Multicast**
* **Class E:** `240 - 255` $\rightarrow$ **Experimental / Reserved**

*Note: Classful addressing has largely been replaced by CIDR (Classless Inter-Domain Routing), which uses prefix lengths such as `/24`, `/16`, and `/8` instead of fixed classes.*

---

## 2. Private IP Ranges & Subnetting Concepts

### Private IP Ranges

These IPs are not routable on the public internet and are used for local area networks.

* **Class A:** `10.0.0.0 - 10.255.255.255`
* **Class B:** `172.16.0.0 - 172.31.255.255`
* **Class C:** `192.168.0.0 - 192.168.255.255`

### Subnetting Methods

* **FLSM (Fixed Length Subnet Mask):** All subnets use the exact same Subnet Mask. Therefore, all subnets will have an equal number of hosts and IPs available.
* **VLSM (Variable Length Subnet Mask):** Each subnet can have a different Subnet Mask. This allows the size of each subnet to be customized based on specific network requirements, minimizing wasted IP addresses.

---

## 3. Important Network Boundaries & Definitions

Based on your notes, here is a detailed explanation of the most important concepts in a subnet:

* **Subnet ID (First Address):** The very first IP address in a subnet range. It represents the network itself. **Note:** *The first address cannot be assigned to a client (it cannot be set on a Network Interface Card / NIC).* However, routers typically use the *first usable address* (the IP right after the Subnet ID) as the default gateway for that subnet.
* **Direct Broadcast (Last Address):** The very last IP address in a subnet range. **Note:** *Packets sent to this last address are received by all IP addresses that are within that specific subnet range.* Like the Subnet ID, the last address cannot be assigned to a single client.
* **Communication Rule:** Two computers can only communicate directly with each other (without needing a router) if their IP addresses belong to the exact same network (subnet).
* **Routing:** Routers are responsible for routing traffic *between* different networks.

---

## 4. Subnetting Design Examples

### Scenario A: Subnetting based on Host Requirement
**Goal:** Subnet the `10.0.0.0/8` network to accommodate **1500 hosts** per subnet.

1. **Calculate required Host bits:** $2^{11} = 2048 \ge 1500$ (We need 11 host bits).
2. **Binary Breakdown:** 
   IP Address: `11111111 . 00000000 . 00000000 . 00000000`
   Subnet Mask (`/21`): `11111111 . 11111111 . 11111 | 000 . 00000000`
   Decimal Mask: `255 . 255 . 248 . 0`
3. **Subnet Step Calculation:**
   `11111111 . 11111111 . 11111 | 000 . 00000000`
   `10       . 0        . 00000 | 000 . 00000000`

**Resulting Subnets (Groups):**
* `H01:` Network: `10.0.0.0/21` $\rightarrow$ Broadcast: `10.0.7.255`
* `H02:` Network: `10.0.8.0/21` $\rightarrow$ Broadcast: `10.0.15.255`
* `H03:` Network: `10.0.16.0/21` $\rightarrow$ Broadcast: `10.0.23.255`
* `H04:` Network: `10.0.24.0/21` $\rightarrow$ Broadcast: `10.0.31.255`
* `H05:` Network: `10.0.32.0/21` $\rightarrow$ Broadcast: `10.0.39.255`
* `H06:` Network: `10.0.40.0/21` $\rightarrow$ Broadcast: `10.0.47.255`

### Scenario B: Subnetting based on Subnet Requirement
**Goal:** Subnet the `10.0.0.0/8` network to create **800 subnets**.

1. **Calculate required Subnet bits:** $2^{10} = 1024 \ge 800$ (Borrow 10 bits).
2. **Binary Breakdown:** 
   IP Address: `11111111 . 00000000 . 00000000 . 00000000`
   Subnet Mask (`/18`): `11111111 . 11111111 . 11 | 000000 . 00000000`
   Decimal Mask: `255 . 255 . 192 . 0`
3. **Subnet Step Calculation:**
   `11111111 . 11111111 . 11 | 000000 . 00000000`
   `10       . 0        . 00 | 000000 . 00000000`

**Resulting Subnets (Groups):**
* `S01:` Subnet ID: `10.0.0.0/18` $\rightarrow$ Direct Broadcast: `10.0.63.255`
* `S02:` Subnet ID: `10.0.64.0/18` $\rightarrow$ Direct Broadcast: `10.0.127.255`
* `S03:` Subnet ID: `10.0.128.0/18` $\rightarrow$ Direct Broadcast: `10.0.191.255`
* `S04:` Subnet ID: `10.0.192.0/18`
* `S05:` Subnet ID: `10.1.0.0/18`
* `S06:` Subnet ID: `10.1.64.0/18`
* `S07:` Subnet ID: `10.1.128.0/18`

---

## 5. IP Address Analysis Examples (Binary Breakdowns)

**Example 1:**
* **IP Address:** `172.16.100.100` 
* **Subnet Mask:** `255.255.224.0`
* **Binary:**
  `11111111 . 11111111 . 1110 | 0000 . 00000000`
  `172      . 16       . 011  | 00100 . 01100100`
* **Subnet ID:** `172.16.96.0/19`
* **Direct Broadcast:** `172.16.127.255`
* **Usable IP Addresses:** $2^{13} - 2 = 8,190$

**Example 2:**
* **IP Address:** `10.140.140.140`
* **Subnet Mask:** `255.224.0.0`
* **Binary:**
  `11111111 . 111 | 00000 . 00000000 . 00000000`
  `10       . 100 | 01100 . 10001100 . 10001100`
* **Subnet ID:** `10.128.0.0/11`
* **Direct Broadcast:** `10.159.255.255`
* **Usable IP Addresses:** $2^{21} - 2 = 2,097,150$

**Example 3:**
* **IP Address:** `192.168.10.20`
* **Subnet Mask:** `255.255.255.248`
* **Binary:**
  `11111111 . 11111111 . 11111111 . 11111 | 000`
  `192      . 168      . 10       . 00010 | 100`
* **Subnet ID:** `192.168.10.16/29`
* **Direct Broadcast:** `192.168.10.23`
* **Usable IP Addresses:** $2^3 - 2 = 6$

**Example 4:**
* **IP Address:** `192.168.10.25`
* **Subnet Mask:** `255.255.255.252`
* **Binary:**
  `11111111 . 11111111 . 11111111 . 111111 | 00`
  `192      . 168      . 10       . 000110 | 01`
* **Subnet ID:** `192.168.10.24/30`
* **Direct Broadcast:** `192.168.10.27`
* **Usable IP Addresses:** $2^2 - 2 = 2$

**Example 5:**
* **IP Address:** `192.168.10.200`
* **Subnet Mask:** `255.255.255.240`
* **Binary:**
  `11111111 . 11111111 . 11111111 . 1111 | 0000`
  `192      . 168      . 10       . 1100 | 1000`
* **Subnet ID:** `192.168.10.192/28`
* **Direct Broadcast:** `192.168.10.207`
* **Usable IP Addresses:** $2^4 - 2 = 14$
