---
layout: default
title: "Home"
description: "Lab instructions for the routing unit"
---

## Welcome to Routing Lab Instructions

This site contains comprehensive lab instructions for the routing unit. You will learn about various routing concepts and protocols through hands-on exercises.

## Lab Overview

| Lab | Topic | Duration | Description |
|-----|-------|----------|-------------|
| [Lab 0](/labs/lab-0) | Environment Setup | 15 min | SSH access, lab topology, and FRR terminal basics |
| [Lab 1](/labs/lab-1) | IPv4 BGP and OSPF | 90 min | Configure IPv4 BGP peering with ISP and OSPF backup path |
| [Lab 2](/labs/lab-2) | IPv6 Multi-homed BGP | 120 min | Set up multi-homed IPv6 BGP with OSPFv3 redundancy |
| [Lab 3](/labs/lab-3) | BGP Traffic Engineering | 90 min | Control traffic using AS-prepend, local-pref, and communities |

## Learning Objectives

By completing these labs, you will be able to:

- Configure IPv4 BGP peering with ISP routers
- Implement OSPF for internal routing and backup paths
- Set up multi-homed IPv6 BGP for redundancy
- Configure OSPFv3 for IPv6 infrastructure
- Apply BGP traffic engineering techniques (AS-prepend, local-pref, communities)
- Verify and troubleshoot routing configurations using looking glass tools

## Prerequisites

Before starting the labs, ensure you have:

1. Basic understanding of TCP/IP networking (IPv4 and IPv6)
2. Familiarity with IP addressing and subnetting
3. Understanding of routing concepts (static routing, dynamic routing)
4. SSH client and terminal application
5. Credentials provided by your instructor (SSH key, AS number, allocated prefixes)

## Getting Started

Begin with [Lab 0: Environment Setup](/labs/lab-0) and progress through each lab sequentially. Each lab builds upon concepts from previous exercises.

## Resources

- [FRR Documentation](https://docs.frrouting.org/)
- [OSPF Design Guide](https://www.cisco.com/c/en/us/support/docs/ip/open-shortest-path-first-ospf/7039-1.html)
- [BGP Configuration Guide](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/26634-bgp-toc.html)
- [Looking Glass](http://151.158.219.14) (accessible within university network)

## Need Help?

If you encounter issues during the labs:

1. Review the troubleshooting sections in each lab
2. Check the common issues tables
3. Consult with your instructor
4. Use the `show` commands to verify configurations
