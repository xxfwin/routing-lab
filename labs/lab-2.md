---
layout: lab
title: "Lab 2: Dynamic Routing with OSPF"
lab_number: 2
duration: "90 minutes"
objectives:
  - Understand OSPF routing protocol concepts
  - Configure OSPF on multiple routers
  - Verify OSPF neighbor relationships
  - Analyze OSPF database
prev:
  url: /labs/lab-1
  title: Lab 1 - Static Routing
next:
  url: /labs/lab-3
  title: Lab 3 - BGP Configuration
---

## Introduction

Open Shortest Path First (OSPF) is a link-state routing protocol that automatically discovers and maintains routes. In this lab, you will configure OSPF in a multi-router environment.

## Prerequisites

- Completion of Lab 1
- Three routers connected as shown in topology
- IP addresses configured on all interfaces

## Lab Topology

```
        Area 0
    ┌──────────────┐
    │              │
[Router A] ---- [Router B] ---- [Router C]
 10.0.1.1       10.0.2.1        10.0.3.1
```

## Task 1: Enable OSPF Process

On each router, enable the OSPF routing process:

```bash
RouterA(config)# router ospf 1
RouterA(config-router)# router-id 1.1.1.1
```

Repeat for Router B and Router C with appropriate router-ids (2.2.2.2 and 3.3.3.3).

## Task 2: Configure Network Statements

Add the directly connected networks to OSPF:

```bash
RouterA(config-router)# network 10.0.1.0 0.0.0.255 area 0
RouterA(config-router)# network 10.0.2.0 0.0.0.255 area 0
```

### Understanding the Command

- `network` - The network to advertise
- `0.0.0.255` - Wildcard mask (inverse of subnet mask)
- `area 0` - OSPF area (backbone area)

## Task 3: Verify OSPF Neighbors

Check that OSPF neighbors have formed:

```bash
RouterA# show ip ospf neighbor
```

Expected output:

```
Neighbor ID     State       Interface
2.2.2.2         FULL        GigabitEthernet0/0
```

## Task 4: Examine OSPF Database

View the OSPF link-state database:

```bash
RouterA# show ip ospf database
```

## Task 5: Verify Routing Table

Check that OSPF routes are being learned:

```bash
RouterA# show ip route ospf
```

Routes learned via OSPF will be marked with `O`.

## Verification Checklist

- [ ] OSPF process running on all routers
- [ ] Neighbor relationships established (FULL state)
- [ ] OSPF routes in routing table
- [ ] Full connectivity between all networks
- [ ] Configuration saved

## OSPF States Reference

| State | Description |
|-------|-------------|
| Down | No hello received |
| Init | Hello received |
| 2-Way | Bidirectional communication |
| ExStart | Database exchange starting |
| Exchange | Database exchange in progress |
| Loading | Loading database entries |
| Full | Fully adjacent |

## Troubleshooting Tips

<blockquote class="warning">
If neighbors don't form, check:
- Interface IP addresses and masks
- OSPF area configuration
- Hello and Dead timers match
- Authentication settings
</blockquote>

## Conclusion

You have successfully configured OSPF in a multi-router environment. OSPF automatically maintains routing information and adapts to network changes.
