---
layout: lab
title: "Lab 7-1: Configuring BGP with Default Routing"
lab_number: 7
duration: "60 minutes"
objectives:
  - Configure BGP peering between an enterprise boundary router and two ISP routers
  - Advertise enterprise networks to ISPs while filtering transit routes
  - Implement floating static routes for primary/backup default gateway redundancy
  - Propagate a default route into the BGP domain using the default-originate command
prev:
  url: /
  title: Home
next:
  url: /labs/lab-7-2
  title: Lab 7-2: Advanced BGP Path Control
---

## Introduction

The International Travel Agency (R2-ITA) relies extensively on the Internet for sales and requires a multihomed ISP connectivity solution with fault tolerance. In this lab, you will configure BGP between the R2-ITA boundary router (AS 100) and two ISP routers (R1-ISP1 in AS 200, R3-ISP2 in AS 300). You will learn to exchange routing information, prevent unwanted transit routing using route filters, implement floating static routes for default path redundancy, and propagate a default route into BGP.

## Prerequisites

Before starting this lab, ensure you have:

- Access to an IOU/IOS simulator with three routers configured
- Basic understanding of IP addressing and subnetting
- Familiarity with Cisco IOS interface configuration and BGP fundamentals
- Knowledge of administrative distance and static routing
- The provided network addressing scheme

## Lab Topology

```
[R1-ISP1 (AS 200)] ---- [R2-ITA (AS 100)] ---- [R3-ISP2 (AS 300)]
  10.1.1.1/24               10.0.0.2/30            172.16.1.1/24
  10.0.0.1/30               172.16.0.2/30          172.16.0.1/30
                          / 192.168.0.1/24 (Lo0)
                          / 192.168.1.1/24 (Lo1)
```

| Router | Interface | IP Address/Subnet | Description |
|--------|-----------|-------------------|-------------|
| R1-ISP1 | Lo0 | 10.1.1.1/24 | Internet Network |
| R1-ISP1 | S1/0 | 10.0.0.1/30 | Link to R2-ITA |
| R2-ITA | Lo0 | 192.168.0.1/24 | Core link 1 |
| R2-ITA | Lo1 | 192.168.1.1/24 | Core link 2 |
| R2-ITA | S1/0 | 10.0.0.2/30 | Link to R1-ISP1 |
| R2-ITA | S1/1 | 172.16.0.2/30 | Link to R3-ISP2 |
| R3-ISP2 | Lo0 | 172.16.1.1/24 | Internet Network |
| R3-ISP2 | S1/1 | 172.16.0.1/30 | Link to R2-ITA |

## Task 1: Configure Interface Addresses

Create the loopback interfaces and apply IPv4 addresses to the loopback and serial interfaces on R1-ISP1, R2-ITA, and R3-ISP2. Set a clock rate on the DCE serial interfaces to enable the link.

```bash
! R1-ISP1
R1-ISP1(config)# interface Lo0
R1-ISP1(config-if)# description R1-ISP1 Internet Network
R1-ISP1(config-if)# ip address 10.1.1.1 255.255.255.0
R1-ISP1(config-if)# exit
R1-ISP1(config)# interface Serial1/0
R1-ISP1(config-if)# description R1-ISP1 -> R2-ITA
R1-ISP1(config-if)# ip address 10.0.0.1 255.255.255.252
R1-ISP1(config-if)# clock rate 128000
R1-ISP1(config-if)# no shutdown
R1-ISP1(config-if)# end

! R2-ITA
R2-ITA(config)# interface Lo0
R2-ITA(config-if)# description Core router network link 1
R2-ITA(config-if)# ip address 192.168.0.1 255.255.255.0
R2-ITA(config-if)# exit
R2-ITA(config)# interface Lo1
R2-ITA(config-if)# description Core router network link 2
R2-ITA(config-if)# ip address 192.168.1.1 255.255.255.0
R2-ITA(config-if)# exit
R2-ITA(config)# interface Serial1/0
R2-ITA(config-if)# description R2-ITA -> R1-ISP1
R2-ITA(config-if)# ip address 10.0.0.2 255.255.255.252
R2-ITA(config-if)# no shutdown
R2-ITA(config-if)# exit
R2-ITA(config)# interface Serial1/1
R2-ITA(config-if)# description R2-ITA -> R3-ISP2
R2-ITA(config-if)# ip address 172.16.0.2 255.255.255.252
R2-ITA(config-if)# clock rate 128000
R2-ITA(config-if)# no shutdown
R2-ITA(config-if)# end

! R3-ISP2
R3-ISP2(config)# interface Lo0
R3-ISP2(config-if)# description R3-ISP2 Internet Network
R3-ISP2(config-if)# ip address 172.16.1.1 255.255.255.0
R3-ISP2(config-if)# exit
R3-ISP2(config)# interface Serial1/1
R3-ISP2(config-if)# description R3-ISP2 -> R2-ITA
R3-ISP2(config-if)# ip address 172.16.0.1 255.255.255.252
R3-ISP2(config-if)# no shutdown
R3-ISP2(config-if)# end
```

Test connectivity between directly connected routers using `ping`. Note that R1-ISP1 and R3-ISP2 cannot reach each other yet, as they are not directly connected.

## Task 2: Configure BGP Peering and Advertise Networks

Configure BGP on the ISP routers to peer with R2-ITA and advertise their loopback networks. Then configure BGP on R2-ITA to peer with both ISPs and advertise its core networks.

```bash
! R1-ISP1
R1-ISP1(config)# router bgp 200
R1-ISP1(config-router)# neighbor 10.0.0.2 remote-as 100
R1-ISP1(config-router)# network 10.1.1.0 mask 255.255.255.0

! R3-ISP2
R3-ISP2(config)# router bgp 300
R3-ISP2(config-router)# neighbor 172.16.0.2 remote-as 100
R3-ISP2(config-router)# network 172.16.1.0 mask 255.255.255.0

! R2-ITA
R2-ITA(config)# router bgp 100
R2-ITA(config-router)# neighbor 10.0.0.1 remote-as 200
R2-ITA(config-router)# neighbor 172.16.0.1 remote-as 300
R2-ITA(config-router)# network 192.168.0.0
R2-ITA(config-router)# network 192.168.1.0
```

### Explanation
- ISP routers (AS 200 & 300) peer with R2-ITA (AS 100) and advertise their Internet loopbacks.
- R2-ITA peers with both ISPs and advertises its internal core networks (Lo0 & Lo1).
- BGP adjacency messages (`%BGP-5-ADJCHANGE`) should appear on the console as neighbors reach the `Established` state.

## Task 3: Verify BGP Operation and Connectivity

Verify BGP peering and routing table entries on R2-ITA, then run a connectivity test across the topology.

```bash
R2-ITA# show ip route
R2-ITA# show ip bgp
```

**Expected Output Notes:**
- `B` entries indicate BGP-learned routes.
- An asterisk (`*`) marks a valid route, and an angle bracket (`>`) marks the best route installed in the routing table.
- The local router ID is derived from the highest IP address on an active interface (typically 192.168.1.1).

Test end-to-end connectivity using the Tcl script or individual pings:
```bash
R2-ITA# ping 10.1.1.1
R2-ITA# ping 172.16.1.1
R2-ITA# ping 192.168.0.1
```
*Note: WAN serial subnets are not advertised in BGP, so ISPs cannot ping each other's serial interfaces directly.*

## Task 4: Configure Route Filters to Prevent Transit Routing

By default, R2-ITA advertises routes learned from one ISP to the other, making it a transit router. Configure an access list and apply it as an outbound route filter to only advertise R2-ITA's own networks.

```bash
R2-ITA(config)# access-list 1 permit 192.168.0.0 0.0.1.255
R2-ITA(config)# router bgp 100
R2-ITA(config-router)# neighbor 10.0.0.1 distribute-list 1 out
R2-ITA(config-router)# neighbor 172.16.0.1 distribute-list 1 out
```

### Explanation
- ACL 1 permits only the 192.168.0.0/22 range (covering Lo0 and Lo1).
- The `distribute-list` applies the ACL to BGP updates sent to neighbors.
- After applying the filter, clear BGP adjacencies to refresh the routing tables:
```bash
R2-ITA# clear ip bgp *
```
- Verify on R3-ISP2 and R1-ISP1 that the route to the other ISP's loopback no longer appears in their routing tables.

## Task 5: Configure Floating Static Routes for Redundancy

Configure primary and backup default routes using floating statics. R1-ISP1 will be primary (lower AD), and R3-ISP2 will be backup (higher AD).

```bash
R2-ITA(config)# ip route 0.0.0.0 0.0.0.0 10.0.0.1 210
R2-ITA(config)# ip route 0.0.0.0 0.0.0.0 172.16.0.1 220
```

**Verification:**
```bash
R2-ITA# show ip route
```
You should see `Gateway of last resort is 10.0.0.1 to network 0.0.0.0` and an entry `S* 0.0.0.0/0 [210/0] via 10.0.0.1`.

Test the default route by creating an unadvertised loopback on R1-ISP1 and pinging it from R2-ITA:
```bash
R1-ISP1(config)# interface Loopback100
R1-ISP1(config-if)# ip address 192.168.100.1 255.255.255.0
R2-ITA# ping 192.168.100.1 source 192.168.1.1
```
Success confirms the default route is functioning. Test fails if the primary link goes down and the backup route is used (due to missing reverse path on R1-ISP1).

## Task 6: Propagate Default Route via BGP

Remove the static default routes and configure R1-ISP1 to inject a default route into BGP.

```bash
! Remove static defaults on R2-ITA
R2-ITA(config)# no ip route 0.0.0.0 0.0.0.0 10.0.0.1 210
R2-ITA(config)# no ip route 0.0.0.0 0.0.0.0 172.16.0.1 220

! Configure default-originate on R1-ISP1
R1-ISP1(config)# router bgp 200
R1-ISP1(config-router)# neighbor 10.0.0.2 default-originate
```

**Verification:**
```bash
R2-ITA# show ip route
```
You should now see `B* 0.0.0.0/0 [20/0] via 10.0.0.1`, indicating the default route is learned via BGP instead of static routing.

## Verification Checklist

- [ ] BGP neighbors on R1-ISP1, R2-ITA, and R3-ISP2 reach `Established` state
- [ ] ISP loopbacks (10.1.1.0/24 & 172.16.1.0/24) are visible in R2-ITA's routing table
- [ ] Route filtering successfully prevents R3-ISP2 from learning 10.1.1.0/24 via R2-ITA
- [ ] Floating static route installs as gateway of last resort (Administrative Distance 210)
- [ ] Backup static route appears in the table with Administrative Distance 220
- [ ] Extended ping to unadvertised network succeeds using the primary path
- [ ] Default route is successfully propagated and learned via BGP (`B* 0.0.0.0/0`)

## Common Issues

| Issue | Solution |
|-------|----------|
| BGP neighbors stuck in `Idle` or `Active` | Verify IP addressing, ensure `clock rate` is set on DCE ends, and confirm AS numbers match neighbor configurations |
| Route filter not taking effect immediately | BGP does not automatically refresh outbound updates. Use `clear ip bgp *` or `clear ip bgp * out` to force a new advertisement |
| Extended ping to unadvertised network fails | Ensure the default route points toward the ISP hosting the test network. Reverse path routing must exist for ICMP replies |
| BGP table version mismatch between routers | Normal behavior; version increments when routes are updated, neighbors reset, or policies change |
| Transit routing still occurs | Verify ACL is correctly applied with `distribute-list 1 out` and that `clear ip bgp *` was executed |

## Cleanup

Restore the routers to their initial state by removing all configurations applied during this lab:

```bash
! R2-ITA
R2-ITA# configure terminal
R2-ITA(config)# router bgp 100
R2-ITA(config-router)# no neighbor 10.0.0.1 distribute-list 1 out
R2-ITA(config-router)# no neighbor 172.16.0.1 distribute-list 1 out
R2-ITA(config-router)# exit
R2-ITA(config)# no access-list 1
R2-ITA(config)# end
R2-ITA# write erase
R2-ITA# reload

! R1-ISP1 & R3-ISP2
R1-ISP1# configure terminal
R1-ISP1(config)# router bgp 200
R1-ISP1(config-router)# no neighbor 10.0.0.2 default-originate
R1-ISP1(config-router)# no neighbor 10.0.0.2 remote-as 100
R1-ISP1(config-router)# no network 10.1.1.0 mask 255.255.255.0
R1-ISP1(config-router)# end
R1-ISP1# write erase
R1-ISP1# reload

R3-ISP2# configure terminal
R3-ISP2(config)# router bgp 300
R3-ISP2(config-router)# no neighbor 172.16.0.2 remote-as 100
R3-ISP2(config-router)# no network 172.16.1.0 mask 255.255.255.0
R3-ISP2(config-router)# end
R3-ISP2# write erase
R3-ISP2# reload
```

## Conclusion

In this lab, you configured BGP to establish multihomed ISP connectivity, learned how to advertise and filter routes to prevent unwanted transit routing, implemented floating static routes for default path redundancy, and propagated a default route into the BGP domain. These techniques are fundamental for designing fault-tolerant, scalable enterprise edge networks that interact with multiple service providers.