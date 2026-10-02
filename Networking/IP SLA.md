# IP Service Level Agreement (IP SLA)

## 1. What exactly is IP SLA?

IP SLA is basically your router's way of actively checking if the network is healthy. Instead of waiting for the phone to ring because the internet is down, the router sends out its own test traffic (like pings or TCP connections) to see if a destination is actually reachable and performing well.

* **The Concept:** A router (the Source) sends synthetic traffic to a destination. It measures how long it takes to get a response, if any packets got lost along the way, or even variations in delay (known as jitter).

* **The Real Benefit:** It lets your network react on autopilot. For example, if your primary WAN link goes dead, IP SLA notices this instantly and tells the router to fail over to the backup link without any human intervention.

## 2. The Core Pieces of the Puzzle

To make IP SLA work, you generally need to set up three or four components:

1. **The Operation:** The actual test you want to run (e.g., an ICMP Ping, a DNS lookup, or a TCP connect).

2. **The Scheduler:** This tells the router when to start the test and how long to keep running it.

3. **The Track Object:** Think of this as the messenger. It monitors the result of the IP SLA operation (is it UP or DOWN?) and reports that status to your routing protocols or FHRP (like HSRP).

4. **The Responder (Optional):** A Cisco router on the other end that explicitly listens for and responds to IP SLA packets. You don't need this for a simple ping, but it is absolutely required for highly accurate tests like UDP Jitter.

## 3. Configuration Scenario A: Reliable Static Routing

**The Scenario:** You have a primary ISP and a backup ISP. You want your default route to point to the primary ISP, but if the internet drops (for instance, you can no longer ping 8.8.8.8), the router should automatically drop the primary route and switch to the backup ISP.

### Step 1: Create the IP SLA Operation

Here we define operation ID 1. We tell it to ping 8.8.8.8 every 10 seconds, and give up if it doesn't get a reply within 2000 milliseconds (2 seconds).

```
Router(config)# ip sla 1
Router(config-ip-sla)# icmp-echo 8.8.8.8 source-interface GigabitEthernet0/1
Router(config-ip-sla-echo)# frequency 10
Router(config-ip-sla-echo)# timeout 2000
Router(config-ip-sla-echo)# exit
```

### Step 2: Schedule the IP SLA

The operation we just created is sitting there doing nothing. We need to schedule it to start right now and run forever.

```
Router(config)# ip sla schedule 1 life forever start-time now
```

### Step 3: Tie the IP SLA to a Track Object

We create Track object 10 to keep an eye on IP SLA 1. If the ping succeeds, Track 10 is UP. If the ping fails, Track 10 goes DOWN.

```
Router(config)# track 10 ip sla 1 reachability
```

### Step 4: Apply the Track Object to the Routing Table

Now we write our default routes. The primary route uses the track object. The backup route has a higher administrative distance (10) so it stays hidden until the primary route fails.

```
Router(config)# ip route 0.0.0.0 0.0.0.0 198.51.100.1 track 10
Router(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1 10
```

> [Note] Integration with FHRP: You can use this exact same track object in HSRP, VRRP, or GLBP!
> Example: SW(config-if)# standby 1 track 10 decrement 20

## 4. Configuration Scenario B: UDP Jitter (VoIP Monitoring)

**The Scenario:** You are deploying VoIP phones and need to measure network jitter and packet loss between a Branch Router and the HQ Router. Simple pings are not accurate enough for voice traffic, so we must use the UDP Jitter operation. This requires a Responder at HQ.

### Step 1: Configure the Responder (HQ Router)

Before you do anything at the branch, you must tell the HQ router to act as a responder on a specific UDP port (let's use port 5000).

```
HQ-Router(config)# ip sla responder udp-echo port 5000
```

### Step 2: Configure the Source Operation (Branch Router)

Now, on the branch router, we create the UDP Jitter operation. We target the HQ router's IP and port 5000, and set it to run every 30 seconds.

```
Branch-Router(config)# ip sla 2
Branch-Router(config-ip-sla)# udp-jitter 10.0.0.1 5000 source-ip 192.168.1.1
Branch-Router(config-ip-sla-jitter)# frequency 30
Branch-Router(config-ip-sla-jitter)# exit
```

### Step 3: Schedule the Operation

Just like before, we start the operation immediately.

```
Branch-Router(config)# ip sla schedule 2 life forever start-time now
```

## 5. Monitoring IP SLA with NMS Tools (SolarWinds, Zabbix, PRTG)

You do not have to stay glued to the CLI to check your SLA statistics. You can easily pull this data into your favorite Network Management System (NMS) dashboard.

**How does the NMS read the data?**
Cisco routers store all IP SLA statistics (like round-trip time, packet loss, and jitter) in a specific SNMP database called the **CISCO-RTTMON-MIB**. Your NMS tool simply polls this MIB using standard SNMP requests to draw beautiful graphs and charts.

**Proactive Alerting (What is an SNMP Trap?):**
Normally, a monitoring server asks the router for an update every 5 minutes (this is called polling). But if a critical internet link goes down, you do not want to wait 5 minutes to find out! 

An **SNMP Trap** is an unsolicited, instant alert. It means the router yells at the monitoring server saying "Hey, something just broke!" without waiting to be asked. It is basically a push notification for network emergencies.

### Step 1: Basic SNMP Configuration

First, make sure your router is allowed to talk to your monitoring server (e.g., 10.10.10.50).

```
! Set a read-only community string for normal polling
Router(config)# snmp-server community MY-MONITOR RO

! Tell the router where to send the push notifications (Traps)
Router(config)# snmp-server host 10.10.10.50 version 2c MY-MONITOR

! Enable SNMP traps specifically for IP SLA (RTTMON)
Router(config)# snmp-server enable traps rtr
```

### Step 2: Triggering the Alert (Reaction Configuration)

Now, we tell the router to generate a trap if IP SLA 1 fails (like when the ping times out).

```
! If SLA 1 times out, trigger an instant SNMP trap
Router(config)# ip sla reaction-configuration 1 react timeout action-type trapOnly
```

Now, if your WAN drops, your router instantly fires off an SNMP Trap to your dashboard, triggering your alarms or email notifications immediately!

## 6. Verification & Troubleshooting

How do we know if it is actually working? Use these commands to check what is going on under the hood.

**Check a quick summary of all IP SLA operations and their current status (OK, Timeout, etc.):**

```
Router# show ip sla summary
```

**Dig into the detailed statistics (Successes, failures, and exact jitter/delay numbers):**

```
Router# show ip sla statistics
Router# show ip sla statistics 2

```

**Check the status of your Track objects (Is it UP or DOWN? When did it last change state?):**

```
Router# show track
Router# show track 10
```

**Verify the parameters you configured for your IP SLA:**

```
Router# show ip sla configuration
```
