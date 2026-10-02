# Comprehensive Reference: NetFlow and Flexible NetFlow (FNF)

Here is a comprehensive reference guide to NetFlow and Flexible NetFlow (FNF), covering their architectures, components, operational mechanics, and key differences.

## 1. What is NetFlow?

Developed by Cisco, **NetFlow** is a network protocol used to collect IP traffic information and monitor network flow. By analyzing NetFlow data, network administrators can determine the source and destination of traffic, class of service, and the causes of congestion.

### Understanding "The Flow"

At the heart of NetFlow is the concept of a **"Flow"**. But what exactly is it?

A flow is a **unidirectional** sequence of packets passing through a network device that share a common set of characteristics. You can think of a flow as a specific "conversation stream" happening on the network.

**When do we call it a "Flow"?**
A router begins tracking a flow the moment it receives the *first* packet of a new communication that doesn't match any currently tracked flow in its cache. As subsequent packets arrive, if they match the exact same characteristics, they are added to that existing flow record (incrementing the packet and byte counters) rather than creating a new one.

**Important Characteristics of a Flow:**

* **Unidirectional:** If User A downloads a file from Server B, the packets from Server B to User A constitute *one* flow. The acknowledgment packets from User A back to Server B constitute a *second, separate* flow.

* **Stateful Tracking (in cache):** The router keeps a table in its memory (the Flow Cache) of all active flows.

### How a Flow is Identified: The 7-Tuple

In traditional NetFlow (specifically version 5 and version 9), a unique flow is identified if the following seven packet attributes (the 7-tuple) are identical:

1. **Source IP address:** (e.g., 192.168.1.10)

2. **Destination IP address:** (e.g., 10.0.0.50)

3. **Source port number:** (e.g., 54321)

4. **Destination port number:** (e.g., 443 for HTTPS)

5. **Layer 3 protocol type:** (e.g., TCP=6, UDP=17, ICMP=1)

6. **Type of Service (ToS)** / Differentiated Services Code Point (DSCP): (e.g., 0x00)

7. **Logical ingress interface:** (e.g., GigabitEthernet0/1)

**Example:**
If 100 packets travel from IP A to IP B on port 443, they are counted as **1 flow** with 100 packets.
If the 101st packet comes from IP A to IP B but targets port 80 instead of 443, the router sees the port difference and creates a **2nd, completely new flow**.

### When Does a Flow End? (Timeouts)

A flow doesn't live in the router's memory forever. A flow is considered "finished" and its data is exported to the collector under three main conditions:

1. **TCP Termination (Natural End):** The router sees a TCP FIN or RST flag, indicating the connection is closed.

2. **Active Timeout:** The flow is long-running (like a large file transfer or a VPN tunnel). To prevent the router from holding onto the data forever, it forces an export at regular intervals (default is usually 30 minutes). After export, a new flow starts for the ongoing connection.

3. **Inactive Timeout:** The flow hasn't seen any new packets for a certain period (e.g., a user stopped browsing a website). The router assumes the conversation is over (default is usually 15 seconds) and exports the data to free up memory.

## 2. Core NetFlow Architecture

The traditional NetFlow ecosystem consists of three main components:

* **Flow Exporter:** Usually a router, switch, or firewall. It observes network traffic, aggregates packets into flows, and exports flow records to one or more collectors.

* **Flow Collector:** A dedicated server or appliance that receives, stores, and pre-processes the exported flow data from multiple exporters.

* **Flow Analyzer:** An application or visualization tool that analyzes the collected data to provide insights, generate reports, detect anomalies (like DDoS attacks), and assist in capacity planning.

## 3. What is Flexible NetFlow (FNF)?

**Flexible NetFlow (FNF)** is the next generation of flow technology, primarily built around NetFlow Version 9 and IPFIX (IP Flow Information Export). Traditional NetFlow is rigid - you are forced to use the predefined 7-tuple to define a flow, which can consume significant memory and CPU, and might capture data you don't actually need.

Flexible NetFlow allows administrators to customize exactly what to track and how to track it. You can define custom flow records, inspecting fields from Layer 2 up to Layer 4 (and beyond, with technologies like NBAR - Network Based Application Recognition).

## 4. Flexible NetFlow Components

FNF achieves its modularity by breaking the configuration into distinct, reusable components:

### 1. Flow Records

The most critical part of FNF. A flow record defines what data the router should look at to define a flow (**Key fields**) and what data to collect about that flow (**Non-key fields**).

* **Key Fields:** Used to differentiate one flow from another (e.g., Source IP, Destination IP, Application ID).

* **Non-Key Fields:** Information collected about the flow but not used to uniquely identify it (e.g., byte count, packet count, timestamps).

### 2. Flow Monitors

A flow monitor ties everything together. It is the component applied to an interface. A flow monitor must contain a **Flow Record** and can optionally contain a **Flow Exporter**. It essentially tells the interface *what* to collect and *where* to send it.

### 3. Flow Exporters

Defines the destination for the NetFlow data. It contains parameters such as:

* Destination IP address of the NetFlow Collector.

* Transport protocol and destination port (e.g., UDP 2055).

* Source interface for the exported packets.

* The export protocol version (NetFlow v9 or IPFIX).

### 4. Flow Samplers (Optional)

On high-speed networks, analyzing every single packet can overwhelm the router's CPU. A flow sampler tells the router to analyze only a subset of packets (e.g., 1 out of every 100 packets). This reduces system overhead while still providing statistically accurate traffic profiling.

## 5. Traditional NetFlow vs. Flexible NetFlow

| Feature | Traditional NetFlow (v5) | Flexible NetFlow (v9 / IPFIX) | 
 | ----- | ----- | ----- | 
| **Flow Definition** | Fixed (Always the 7-Tuple). | Customizable (Admin defines Key fields). | 
| **Data Structure** | Static fields. | Template-based and modular. | 
| **Layer Visibility** | Layer 3 and Layer 4 only. | Layer 2, Layer 3, Layer 4, and Application (Layer 7 via NBAR). | 
| **IPv6 Support** | No (IPv4 only in v5). | Yes (Full IPv6, Multicast, MPLS, MAC address support). | 
| **Resource Efficiency** | High overhead (collects unnecessary data). | Highly efficient (collects only what is configured). | 
| **Configuration** | Flat, monolithic configuration. | Object-oriented (Records, Exporters, Monitors). | 

## 6. Primary Use Cases

* **Bandwidth Monitoring & Capacity Planning:** Identifying "top talkers" (users or applications consuming the most bandwidth) to plan WAN or LAN upgrades.

* **Security & Anomaly Detection:** Recognizing sudden spikes in specific traffic types (e.g., TCP SYN floods) to mitigate DDoS attacks or malware propagation.

* **Billing & Accounting:** ISPs use NetFlow to track data usage for customer billing based on volume and protocol.

* **Application Performance Monitoring:** Using FNF coupled with Application Visibility and Control (AVC) to measure latency, jitter, and drop rates for specific applications like VoIP or video streaming.

## 7. Configuration Process (Cisco IOS-XE)

Here is a step-by-step guide to configuring both Traditional NetFlow and Flexible NetFlow on Cisco routers.

### A. Configuring Traditional NetFlow

Traditional NetFlow configuration is relatively flat. You define the export destination globally and enable it on the desired interfaces.

**1. Configure the NetFlow Exporter (Global Configuration)**

```
! Specify the Collector's IP address and the UDP port it listens on
Router(config)# ip flow-export destination 192.168.1.100 2055

! Specify the source interface for exported NetFlow packets (usually a loopback or management interface)
Router(config)# ip flow-export source GigabitEthernet0/1

! Define the NetFlow version to use (version 9 is recommended over version 5 if FNF isn't used)
Router(config)# ip flow-export version 9
```

**2. Enable NetFlow on Interfaces**

You must enable NetFlow on each interface where you want to monitor traffic. You can monitor inbound (`ingress`), outbound (`egress`), or both.

```
! Enter the interface configuration mode
Router(config)# interface GigabitEthernet0/0

! Monitor traffic entering the interface
Router(config-if)# ip flow ingress

! Monitor traffic leaving the interface (Note: requires appropriate hardware/IOS support)
Router(config-if)# ip flow egress
```

### B. Configuring Flexible NetFlow (FNF)

FNF follows a modular, object-oriented approach. You build the components (Record, Exporter) first, combine them into a Monitor, and then apply the Monitor to an interface.

**Step 1: Create a Flow Record**

Define what to match (keys) and what to collect (non-keys).

```
! Name the flow record
Router(config)# flow record MY_CUSTOM_RECORD

! Define the KEY fields (used to uniquely identify a flow)
Router(config-flow-record)# match ipv4 source address
Router(config-flow-record)# match ipv4 destination address
Router(config-flow-record)# match transport source-port
Router(config-flow-record)# match transport destination-port
Router(config-flow-record)# match ipv4 protocol

! Define the NON-KEY fields (data to collect about the matched flows)
Router(config-flow-record)# collect counter bytes long
Router(config-flow-record)# collect counter packets long
Router(config-flow-record)# collect timestamp sys-uptime first
Router(config-flow-record)# collect timestamp sys-uptime last
```

**Step 2: Create a Flow Exporter**

Define where to send the flow data.

```
! Name the flow exporter
Router(config)# flow exporter MY_COLLECTOR

! Specify the Collector's IP address
Router(config-flow-exporter)# destination 192.168.1.100

! Specify the transport protocol and port
Router(config-flow-exporter)# transport udp 2055

! Specify the source interface for the exported packets
Router(config-flow-exporter)# source GigabitEthernet0/1

! Define the export protocol version (NetFlow v9 or IPFIX)
Router(config-flow-exporter)# export-protocol netflow-v9
```

**Step 3: Create a Flow Monitor**

Combine the Record and Exporter. The monitor is what actively tracks the flows in the router's cache.

```
! Name the flow monitor
Router(config)# flow monitor MY_MONITOR

! Bind the previously created record to this monitor
Router(config-flow-monitor)# record MY_CUSTOM_RECORD

! Bind the previously created exporter to this monitor
Router(config-flow-monitor)# exporter MY_COLLECTOR

! (Optional) Set the active flow timeout - export flows that are active for 60 seconds
Router(config-flow-monitor)# cache timeout active 60
```

**Step 4: Apply the Flow Monitor to an Interface**

Finally, attach the monitor to the interface(s) you want to observe.

```
! Enter the interface configuration mode
Router(config)# interface GigabitEthernet0/0

! Apply the monitor to incoming traffic
Router(config-if)# ip flow monitor MY_MONITOR input
Router(config-if)# exit
```

## 8. Configuring Top Talkers (On-Box Monitoring)

Sometimes you need to quickly see which IP addresses or applications are consuming the most bandwidth directly on the router's CLI, without relying on an external collector. This is where the **Top Talkers** feature comes in.

### A. Top Talkers for Traditional NetFlow

This aggregates flows in the router's memory based on specific criteria and presents the top results.

```
! Enter top-talkers configuration mode
Router(config)# ip flow-top-talkers

! Define how many top flows to display (e.g., top 10)
Router(config-flow-top-talkers)# top 10

! Define how to sort the flows (by bytes or packets)
Router(config-flow-top-talkers)# sort-by bytes

! (Optional) Set criteria to match specific traffic, e.g., only show HTTP traffic
! Router(config-flow-top-talkers)# match destination port 80
```

### B. Top Talkers equivalent in Flexible NetFlow

In FNF, there isn't a direct `ip flow-top-talkers` command. Instead, you use the powerful `show flow monitor` command with sorting and filtering options directly from the EXEC prompt. You don't need a separate configuration block for top talkers in FNF.

```
! The sorting and top N logic is done at the execution time in FNF:
! (This will be shown in the Verification section below)
```

## 9. Verification and Troubleshooting (`show` and `debug` Commands)

To verify the configuration, check the status of the caches, and troubleshoot issues, use the following commands.

### Verifying Traditional NetFlow

```
! View the global export configuration and statistics (packets exported, dropped, etc.)
Router# show ip flow export

! View the current NetFlow cache (shows the actual flows being tracked)
Router# show ip cache flow

! View interface-specific NetFlow configuration
Router# show ip interface

! View the Top Talkers output (based on the config in Section 8)
Router# show ip flow top-talkers
```

### Verifying Flexible NetFlow (FNF)

```
! View the configuration of the specific flow monitor
Router# show flow monitor MY_MONITOR

! View the contents of the flow cache (the actual traffic flows being tracked by the monitor)
Router# show flow monitor MY_MONITOR cache format table

! VIEW FNF TOP TALKERS (Sorting the cache directly):
! Sort the cache by the 'bytes' counter in descending order to see who is using the most bandwidth
Router# show flow monitor MY_MONITOR cache sort highest counter bytes

! View statistics for the flow exporter (check for export errors or drops)
Router# show flow exporter MY_COLLECTOR statistics

! View the configuration of a specific flow record
Router# show flow record MY_CUSTOM_RECORD

! View a summary of all FNF components applied to interfaces
Router# show flow interface
```

### Troubleshooting (Debugging)

**Caution:** Debug commands can impact router performance, especially on busy networks. Use them sparingly and during maintenance windows if possible.

```
! (Traditional) Debug the export process of NetFlow v9 packets
Router# debug ip flow export v9

! (FNF) Debug the flow exporter to see if packets are being built and sent to the collector
Router# debug flow exporter events
Router# debug flow exporter packets
```
