# Cisco Object Tracking (Track Object)

## 1. What exactly is a Track Object?

Think of a Track Object as a dedicated network sensor or a "middleman". Its entire job is to watch the condition of a specific network component and report its status (either UP or DOWN) to other services that need to make decisions.

* **The Concept:** A feature like HSRP or Static Routing (the client) wants to know if the network is healthy, but it cannot always check everything itself. So, it asks a Track Object to do the monitoring.
* **The Flow:** You tell the Track Object what to monitor (e.g., an interface, a route, or an IP SLA). The Track Object watches it. Then, you tell HSRP or a static route to act based on the state of that Track Object.

## 2. What can we track?

Cisco routers allow you to track several different components. The most common ones are:

1. **Interface Line Protocol:** Is the physical cable plugged in, and is the Layer 2 protocol up?
2. **Interface IP Routing:** Is the interface up AND does it have a valid, functioning IP address?
3. **IP Route:** Does a specific route currently exist in the routing table?
4. **IP SLA:** Is our active test (like a ping to Google) succeeding?
5. **List (Boolean Logic):** A combination of multiple Track Objects using AND/OR logic.

## 3. Configuration Scenario A: Interface Tracking (The Classic FHRP Setup)

**The Scenario:** You have a core switch running HSRP for your local network (VLAN 10). This switch connects to the internet via GigabitEthernet0/0. If the internet link goes down, you want this switch to give up its HSRP Active role so the backup switch can take over.

### Step 1: Create the Track Object

We will create Track 10 to monitor the line protocol (Layer 2 state) of the uplink interface.

```
Router(config)# track 10 interface GigabitEthernet0/0 line-protocol
Router(config-track)# exit
```

### Step 2: Apply it to HSRP

Now we tell HSRP to monitor Track 10. If Track 10 goes DOWN, HSRP will automatically decrement (reduce) its priority by 20.

```
Router(config)# interface Vlan 10
Router(config-if)# standby 1 track 10 decrement 20
```

## 4. Configuration Scenario B: Route Tracking

**The Scenario:** Sometimes an interface stays physically UP (because it is connected to a local switch), but the actual route to your destination is lost. Here, we track the routing table directly. We want to know if the route to the HQ subnet (10.50.0.0/16) exists.

### Step 1: Create the Route Track Object

We create Track 20 to monitor the IP routing table for the specific HQ subnet.

```
Router(config)# track 20 ip route 10.50.0.0 255.255.0.0 reachability
Router(config-track)# exit
```

### Step 2: Tie it to a Static Route

We can use this track object to inject a backup default route only if the route to HQ is healthy.

```
Router(config)# ip route 0.0.0.0 0.0.0.0 192.168.1.254 track 20
```

## 5. Configuration Scenario C: Boolean Tracking (Advanced AND/OR Logic)

**The Scenario:** You have a highly redundant router with TWO internet links (ISP 1 and ISP 2). You only want your router to failover to the 4G LTE Backup link if BOTH ISP 1 and ISP 2 go down. 

To do this, we track both ISPs individually, and then create a "Master" track object that combines them.

### Step 1: Create the individual Track Objects

First, we track the interface for ISP 1 (Track 1) and the interface for ISP 2 (Track 2).

```
Router(config)# track 1 interface GigabitEthernet0/1 line-protocol
Router(config-track)# exit

Router(config)# track 2 interface GigabitEthernet0/2 line-protocol
Router(config-track)# exit
```

### Step 2: Create the Boolean List (OR Logic)

We create Track 100 as a list using Boolean OR. This means Track 100 will be UP if either Track 1 OR Track 2 is UP. It will only go DOWN if both of them fail.

```
Router(config)# track 100 list boolean or
Router(config-track)# object 1
Router(config-track)# object 2
Router(config-track)# exit
```

### Step 3: Apply the Master Track to your backup route

Now we tie the 4G LTE backup route to this master track object (Track 100). If both ISPs fail, Track 100 goes DOWN, and we can trigger our backup mechanism.

```
! Primary default route relying on the master track
Router(config)# ip route 0.0.0.0 0.0.0.0 10.10.10.1 track 100

! Floating backup route (Administrative Distance of 10) that takes over if Track 100 fails
Router(config)# ip route 0.0.0.0 0.0.0.0 172.16.1.1 10
```

## 6. Adding Delays to Track Objects (Preventing Flapping)

If a cable is loose, an interface might flap UP and DOWN every second. You do not want your entire network routing to flap with it. You can tell a Track Object to delay its reporting.

### Step 1: Configure Up/Down Delays

We tell Track 10 to wait 10 seconds before declaring the interface is DOWN, and wait 20 seconds after the interface comes back before declaring it is UP.

```
Router(config)# track 10 interface GigabitEthernet0/0 line-protocol
Router(config-track)# delay down 10 up 20
```

## 7. Verification & Troubleshooting

Use these commands to check the exact state of your tracking objects.

**Check the status of all track objects (Shows UP/DOWN, delays, and what it is tracking):**

```
Router# show track
```

**Check a specific track object (e.g., Track 100):**

```
Router# show track 100
```

**Check a quick, summarized list of all track objects:**

```
Router# show track brief
```
