---
layout: lab
title: "Lab 3: BGP Configuration"
lab_number: 3
duration: "120 minutes"
objectives:
  - Understand BGP fundamentals and use cases
  - Configure eBGP between autonomous systems
  - Advertise networks via BGP
  - Implement basic BGP policies
prev:
  url: /labs/lab-2
  title: Lab 2 - OSPF Configuration
next:
  url: /
  title: Home
---

## Introduction

Border Gateway Protocol (BGP) is the routing protocol of the Internet. This lab covers basic BGP configuration between different autonomous systems (AS).

## Prerequisites

- Completion of Labs 1 and 2
- Understanding of autonomous systems
- Two routers representing different ASes

## Lab Topology

```
    AS 65001              AS 65002
   ┌─────────┐          ┌─────────┐
   │ Router  │          │ Router  │
   │    A    │──────────│    B    │
   │         │  eBGP    │         │
   └─────────┘          └─────────┘
   192.168.1.1          192.168.1.2
```

## Task 1: Configure BGP Process

On Router A (AS 65001):

```bash
RouterA(config)# router bgp 65001
RouterA(config-router)# bgp router-id 1.1.1.1
RouterA(config-router)# neighbor 192.168.1.2 remote-as 65002
```

On Router B (AS 65002):

```bash
RouterB(config)# router bgp 65002
RouterB(config-router)# bgp router-id 2.2.2.2
RouterB(config-router)# neighbor 192.168.1.1 remote-as 65001
```

## Task 2: Advertise Networks

Advertise the networks you want to share:

```bash
RouterA(config-router)# network 10.0.0.0 mask 255.255.255.0
```

<blockquote class="tip">
The network must be in the routing table (via IGP or connected) for BGP to advertise it.
</blockquote>

## Task 3: Verify BGP Session

Check the BGP neighbor status:

```bash
RouterA# show ip bgp summary
```

Look for the neighbor state to be "Established".

## Task 4: View BGP Table

Examine the BGP routing table:

```bash
RouterA# show ip bgp
```

## Task 5: Implement Route Filtering

Create a prefix list to filter routes:

```bash
RouterA(config)# ip prefix-list ALLOWED_ROUTES permit 10.0.0.0/24
RouterA(config)# router bgp 65001
RouterA(config-router)# neighbor 192.168.1.2 prefix-list ALLOWED_ROUTES out
```

## BGP Path Attributes

| Attribute | Description |
|-----------|-------------|
| AS_PATH | List of ASes the route passed through |
| NEXT_HOP | IP address of next router |
| LOCAL_PREF | Preference within an AS |
| MED | Multi-Exit Discriminator |
| ORIGIN | How route was learned |

## Verification Checklist

- [ ] BGP session established between neighbors
- [ ] Routes being advertised correctly
- [ ] BGP routes in routing table
- [ ] Route filtering working as expected
- [ ] Configuration saved

## Common BGP States

| State | Meaning |
|-------|---------|
| Idle | Not trying to connect |
| Connect | TCP connection attempt |
| Active | Trying to establish TCP |
| OpenSent | BGP OPEN message sent |
| OpenConfirm | Waiting for KEEPALIVE |
| Established | BGP session active |

## Troubleshooting Commands

```bash
show ip bgp neighbors
show ip bgp rib-failure
debug ip bgp events
debug ip bgp updates
```

<blockquote class="warning">
Use debug commands carefully in production environments as they can impact performance.
</blockquote>

## Conclusion

You have configured basic eBGP between two autonomous systems. BGP is essential for inter-domain routing and provides powerful policy control capabilities.
