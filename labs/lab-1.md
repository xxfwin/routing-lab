---
layout: lab
title: "Lab 1: Basic Static Routing"
lab_number: 1
duration: "60 minutes"
objectives:
  - Understand the basics of routing tables
  - Configure static routes on a router
  - Verify connectivity using ping and traceroute
prev:
  url: /
  title: Home
next:
  url: /labs/lab-2
  title: Lab 2 - Dynamic Routing
---

## Introduction

In this lab, you will learn the fundamentals of static routing. Static routes are manually configured routes that tell the router exactly which path to use to reach a specific destination network.

## Prerequisites

Before starting this lab, ensure you have:

- A router (physical or virtual)
- Console access to the router
- Basic understanding of IP addressing
- Network topology diagram provided by your instructor

## Lab Topology

```
[Router A] ---- [Router B] ---- [Router C]
   10.0.1.0/24    10.0.2.0/24    10.0.3.0/24
```

## Task 1: Examine the Current Routing Table

Connect to Router A and examine its current routing table:

```bash
RouterA# show ip route
```

Take note of the directly connected networks and any existing routes.

## Task 2: Configure Static Routes

Configure a static route on Router A to reach the 10.0.3.0/24 network:

```bash
RouterA(config)# ip route 10.0.3.0 255.255.255.0 10.0.2.2
```

### Explanation

- `10.0.3.0` - Destination network
- `255.255.255.0` - Subnet mask
- `10.0.2.2` - Next-hop IP address (Router B's interface)

## Task 3: Verify the Configuration

Verify that the static route has been added:

```bash
RouterA# show ip route
```

You should see a new entry marked with `S` indicating a static route.

## Task 4: Test Connectivity

Test connectivity to the remote network:

```bash
RouterA# ping 10.0.3.1
RouterA# traceroute 10.0.3.1
```

## Verification Checklist

- [ ] Static route appears in routing table
- [ ] Ping to remote network succeeds
- [ ] Traceroute shows correct path
- [ ] Configuration saved to NVRAM

## Common Issues

| Issue | Solution |
|-------|----------|
| Ping fails | Check next-hop IP address |
| Route not in table | Verify interface is up |
| Wrong path taken | Check administrative distance |

## Cleanup

Remove the static route:

```bash
RouterA(config)# no ip route 10.0.3.0 255.255.255.0 10.0.2.2
```

## Conclusion

In this lab, you learned how to configure and verify static routes. Static routing is useful for small networks or when you need precise control over routing paths.
