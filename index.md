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
| [Lab 0](/routing-lab/labs/lab-0) | Environment Setup | 30 min | SSH access, lab topology, and FRR terminal basics |
| [Lab 1](/routing-lab/labs/lab-1) | IPv4 BGP and OSPF | 90 min | Configure IPv4 BGP peering with ISP and OSPF backup path |
| [Lab 2](/routing-lab/labs/lab-2) | IPv6 Multi-homed BGP | 120 min | Set up multi-homed IPv6 BGP with OSPFv3 redundancy |
| [Lab 3](/routing-lab/labs/lab-3) | BGP Traffic Engineering | 90 min | Control traffic using AS-prepend, local-pref, and communities |

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

Begin with [Lab 0: Environment Setup](/routing-lab/labs/lab-0) and progress through each lab sequentially. Each lab builds upon concepts from previous exercises.

## Resources

- [FRR Documentation](https://docs.frrouting.org/)
- [FRR OSPF](https://docs.frrouting.org/en/latest/ospfd.html)
- [FRR Basic BGP Concepts](https://docs.frrouting.org/en/latest/bgp.html#basic-concepts)
- [RFC 4271: A Border Gateway Protocol 4 (BGP-4)](https://datatracker.ietf.org/doc/html/rfc4271)
- [RFC 8212: Default External BGP Route Policies for Routers](https://datatracker.ietf.org/doc/html/rfc8212)
- [Looking Glass](http://151.158.219.14) (accessible within university network)
- [Hurricane Electric BGP Toolkit](https://bgp.he.net/)
- [bgp.tools](https://bgp.tools/)

## RFCs and RIR policies related to the resource allocation

- [RFC 1930: Guidelines for creation, selection, and registration of an Autonomous System (AS)](https://datatracker.ietf.org/doc/html/rfc1930)
- [RFC 2050: INTERNET REGISTRY IP ALLOCATION GUIDELINES](https://datatracker.ietf.org/doc/html/rfc2050)
- [RFC 7020: The Internet Numbers Registry System](https://datatracker.ietf.org/doc/html/rfc7020)
- [APNIC Resource Allocation Policy](https://www.apnic.net/community/policy/resources)
- [RFC 6177: IPv6 Address Assignment to End Sites](https://datatracker.ietf.org/doc/html/rfc6177)

## Need Help?

If you encounter issues during the labs:

1. Review the troubleshooting sections in each lab
2. Check the common issues tables
3. Consult with your instructor
4. Use the `show` commands to verify configurations
