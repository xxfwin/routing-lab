---
layout: lab
title: "Lab 3: BGP Traffic Engineering"
lab_number: 3
duration: "120 minutes"
objectives:
  - Use AS-prepend to influence inbound traffic path selection
  - Configure local preference to control outbound traffic
  - Apply BGP communities for traffic engineering
  - Troubleshoot and optimize traffic flow based on requirements
prev:
  url: /labs/lab-2
  title: Lab 2 - IPv6 Multi-homed BGP
next:
  url: /
  title: Home
---

## Introduction

In a multi-homed BGP environment, you often need to control how traffic enters and exits your network. This lab focuses on BGP traffic engineering techniques that allow you to influence path selection for both inbound and outbound traffic.

You will work through realistic scenarios where specific prefixes require traffic optimization, similar to real-world network operations tickets.

## Prerequisites

Before starting this lab, ensure you have:

- Completed Labs 1 and 2 (IPv4/IPv6 BGP and OSPF configuration)
- Working multi-homed BGP setup with both bgp1 and bgp2
- Understanding of BGP path attributes (AS_PATH, LOCAL_PREF, COMMUNITIES)
- Received from your instructor:
  - Your assigned **AS Number (ASN)**
  - Your allocated **IPv4 and IPv6 prefixes**
  - **ISP community values** for traffic engineering

## Lab Topology

Your multi-homed network has two ISP connections:

- **bgp1**: Primary BGP router (connects to ISP1)
- **bgp2**: Secondary BGP router (connects to ISP2)
- **ds1, ds2**: Distribution switches for internal routing

Both BGP routers peer with ISP routers on separate connections, providing redundancy. OSPFv3 provides routing between BGP routers and distribution switches.

![IPv6 BGP Topology](/routing-lab/assets/image/IPv6_BGP.png)

## Scenario: Traffic Optimization Tickets

You have received several tickets from the network operations team requiring traffic optimization for specific prefixes. Each ticket describes a traffic flow issue that needs to be resolved using BGP traffic engineering techniques.

---

## Ticket 1: Inbound Traffic Optimization Using AS-Prepend

### Problem Statement

A critical service hosted on one of your prefixes is experiencing high latency. Investigation shows that inbound traffic is arriving through a suboptimal path. You need to influence the inbound traffic to use the preferred ISP connection.

### Background

Your allocated prefix includes a subnet hosting latency-sensitive applications. Currently, traffic from certain regions is entering through the backup ISP path, causing increased latency.

### Task 1.1: Analyze Current Traffic Path

First, examine the current BGP path for your prefix:

On bgp1:

```
show bgp ipv4 <your-prefix>
show bgp ipv6 <your-ipv6-prefix>
```

On bgp2:

```
show bgp ipv4 <your-prefix>
show bgp ipv6 <your-ipv6-prefix>
```

Use the looking glass at `151.158.219.14` to check how your prefix is seen from external perspectives:

```bash
curl http://151.158.219.14/lg
```

Query your prefix and note the AS path from different viewpoints.

### Task 1.2: Configure AS-Prepend

AS-prepend adds your AS number multiple times to the AS_PATH attribute, making the path appear longer and less desirable. This influences other ASes to prefer alternative paths.

On the router you want to make **less preferred** for inbound traffic (e.g., bgp2), configure AS-prepend:

```
configure terminal
router bgp <your-asn>
address-family ipv4 unicast
neighbor <isp2-peering-ip> route-map PREPEND-OUT out
exit
exit

route-map PREPEND-OUT permit 10
set as-path prepend <your-asn> <your-asn> <your-asn>
end
```

For IPv6:

```
configure terminal
router bgp <your-asn>
address-family ipv6 unicast
neighbor <isp2-ipv6-peering-ip> route-map PREPEND-OUT-V6 out
exit
exit

route-map PREPEND-OUT-V6 permit 10
set as-path prepend <your-asn> <your-asn> <your-asn>
end
```

<blockquote class="tip">
The number of times you prepend your AS determines how much less desirable the path becomes. Typically, 2-3 prepends are sufficient. More than 5 prepends is rarely needed and can cause issues.
</blockquote>

### Task 1.3: Apply to Specific Prefixes

To apply AS-prepend only to specific prefixes, use a prefix list:

```
configure terminal
ip prefix-list CRITICAL-PREFIX seq 5 permit <specific-prefix>/<mask>

route-map PREPEND-OUT permit 10
match ip address prefix-list CRITICAL-PREFIX
set as-path prepend <your-asn> <your-asn> <your-asn>
exit

route-map PREPEND-OUT permit 20
end
```

The second sequence (20) allows all other prefixes to be advertised normally.

### Task 1.4: Verify AS-Prepend

Clear the BGP session to force re-advertisement:

```
clear bgp ipv4 <isp2-peering-ip> out
clear bgp ipv6 <isp2-ipv6-peering-ip> out
```

Verify the AS path on the looking glass. You should see your AS number repeated in the path through the prepended connection.

### Task 1.5: Verify Traffic Flow

Monitor traffic flow to confirm inbound traffic now prefers the desired path:

```
show interface <interface-to-isp1> | include rate
show interface <interface-to-isp2> | include rate
```

---

## Ticket 2: Outbound Traffic Control Using Local Preference

### Problem Statement

Outbound traffic to certain destinations is using a suboptimal path, resulting in higher costs and latency. You need to configure local preference to ensure outbound traffic uses the preferred ISP connection.

### Background

Local preference is a BGP attribute that indicates the preferred path for outbound traffic within your AS. Higher local preference values are preferred.

### Task 2.1: Analyze Current Outbound Path

Check the current path selection for outbound traffic:

```
show bgp ipv4
show bgp ipv6
```

Look at the `LocPrf` column to see current local preference values.

### Task 2.2: Configure Default Local Preference

To make one ISP the preferred path for all outbound traffic, set a higher local preference on routes received from that ISP.

On bgp1 (preferred ISP):

```
configure terminal
router bgp <your-asn>
address-family ipv4 unicast
neighbor <isp1-peering-ip> route-map SET-LOCALPREF-IN in
exit
exit

route-map SET-LOCALPREF-IN permit 10
set local-preference 200
end
```

On bgp2 (backup ISP):

```
configure terminal
router bgp <your-asn>
address-family ipv4 unicast
neighbor <isp2-peering-ip> route-map SET-LOCALPREF-IN in
exit
exit

route-map SET-LOCALPREF-IN permit 10
set local-preference 100
end
```

<blockquote class="tip">
Default local preference is 100. Setting a value higher than 100 makes that path preferred. Setting lower than 100 makes it less preferred.
</blockquote>

### Task 2.3: Apply Local Preference to Specific Prefixes

To set local preference for specific destination prefixes:

```
configure terminal
ip prefix-list DEST-PREFIXES seq 5 permit <destination-prefix>/<mask>

route-map SET-LOCALPREF-IN permit 10
match ip address prefix-list DEST-PREFIXES
set local-preference 250
exit

route-map SET-LOCALPREF-IN permit 20
set local-preference 100
end
```

### Task 2.4: Verify Local Preference

After applying the route map, clear and re-receive routes:

```
clear bgp ipv4 <isp-peering-ip> in
```

Verify the local preference values:

```
show bgp ipv4
show bgp ipv4 neighbors <isp-peering-ip> received-routes
```

Routes should now show the configured local preference values.

### Task 2.5: Verify Outbound Traffic Path

Test outbound traffic to verify it uses the preferred path:

```
traceroute <destination-ip>
show ip route <destination-ip>
```

---

## Ticket 3: Traffic Engineering Using BGP Communities

### Problem Statement

Your ISP supports BGP communities for traffic engineering. You need to use communities to request specific handling of your routes, such as limiting propagation or setting local preference on the ISP side.

### Background

BGP communities are optional transitive attributes that allow you to tag routes with additional information. ISPs use communities to provide traffic engineering services to customers.

### Task 3.1: Identify ISP Community Values

Your ISP has assigned specific community values for traffic engineering. These will be provided by your instructor and typically include:

| Community | Purpose |
|-----------|---------|
| `<ISP-ASN>:100` | Set local preference 100 (backup) |
| `<ISP-ASN>:200` | Set local preference 200 (primary) |
| `<ISP-ASN>:300` | Do not advertise to peers |
| `<ISP-ASN>:400` | Advertise only to specific peers |

<blockquote class="tip">
Community values are ISP-specific. Always verify the correct community values with your ISP documentation or instructor.
</blockquote>

### Task 3.2: Configure BGP Communities

To attach communities to your advertised routes:

```
configure terminal
router bgp <your-asn>
address-family ipv4 unicast
neighbor <isp-peering-ip> send-community
neighbor <isp-peering-ip> route-map COMMUNITY-OUT out
exit
exit

ip prefix-list ANNOUNCE-PREFIX seq 5 permit <your-prefix>/<mask>

route-map COMMUNITY-OUT permit 10
match ip address prefix-list ANNOUNCE-PREFIX
set community <ISP-ASN>:200
exit

route-map COMMUNITY-OUT permit 20
end
```

### Task 3.3: Configure Multiple Communities

You can attach multiple communities to a route:

```
route-map COMMUNITY-OUT permit 10
match ip address prefix-list ANNOUNCE-PREFIX
set community <ISP-ASN>:200 <ISP-ASN>:300 additive
end
```

The `additive` keyword preserves existing communities while adding new ones.

### Task 3.4: Verify Communities

Verify that communities are being sent:

```
show bgp ipv4 <your-prefix>
show bgp ipv4 neighbors <isp-peering-ip> advertised-routes
```

Look for the `Community` attribute in the output.

### Task 3.5: Verify ISP Applied Changes

Use the looking glass to verify the ISP has applied the community-based changes:

```bash
curl http://151.158.219.14/lg
```

Query your prefix and check:
- Local preference values (if visible)
- Propagation to different peers
- AS path changes

---

## Ticket 4: Combined Traffic Engineering Scenario

### Problem Statement

A new service deployment requires specific traffic engineering:
1. Inbound traffic for the service subnet should prefer bgp1
2. Outbound traffic to specific destinations should use bgp2
3. The service subnet should have limited propagation (no-advertise to certain peers)

### Task 4.1: Plan the Configuration

Before implementing, plan your approach:

1. **Inbound traffic**: Use AS-prepend on bgp2 for the service subnet
2. **Outbound traffic**: Set higher local preference on bgp2 for specific destinations
3. **Limited propagation**: Use BGP communities to control advertisement

### Task 4.2: Implement AS-Prepend for Service Subnet

On bgp2:

```
configure terminal
ip prefix-list SERVICE-SUBNET seq 5 permit <service-subnet>/<mask>

route-map PREPEND-SERVICE permit 10
match ip address prefix-list SERVICE-SUBNET
set as-path prepend <your-asn> <your-asn> <your-asn>
exit

route-map PREPEND-SERVICE permit 20
end

router bgp <your-asn>
address-family ipv4 unicast
neighbor <isp2-peering-ip> route-map PREPEND-SERVICE out
end
```

### Task 4.3: Implement Local Preference for Destinations

On bgp2:

```
configure terminal
ip prefix-list DEST-NETWORKS seq 5 permit <destination-network>/<mask>

route-map LOCALPREF-DEST permit 10
match ip address prefix-list DEST-NETWORKS
set local-preference 250
exit

route-map LOCALPREF-DEST permit 20
set local-preference 100
end

router bgp <your-asn>
address-family ipv4 unicast
neighbor <isp2-peering-ip> route-map LOCALPREF-DEST in
end
```

### Task 4.4: Implement Communities for Limited Propagation

On bgp1:

```
configure terminal
ip prefix-list SERVICE-SUBNET seq 5 permit <service-subnet>/<mask>

route-map COMMUNITY-SERVICE permit 10
match ip address prefix-list SERVICE-SUBNET
set community <ISP-ASN>:300
exit

route-map COMMUNITY-SERVICE permit 20
end

router bgp <your-asn>
address-family ipv4 unicast
neighbor <isp1-peering-ip> send-community
neighbor <isp1-peering-ip> route-map COMMUNITY-SERVICE out
end
```

### Task 4.5: Verify Complete Configuration

Verify all components are working:

```
show bgp ipv4 <service-subnet>
show bgp ipv4 <destination-network>
show bgp ipv4 neighbors <isp-peering-ip> advertised-routes
```

Use the looking glass to verify external visibility:

```bash
curl http://151.158.219.14/lg
```

---

## Verification Checklist

- [ ] AS-prepend configured and verified on looking glass
- [ ] Local preference configured and affecting outbound path selection
- [ ] BGP communities sent and acknowledged by ISP
- [ ] Traffic flow matches requirements for each ticket
- [ ] All configurations saved

## BGP Path Selection Order

Understanding BGP path selection is crucial for traffic engineering:

1. **Weight** (Cisco proprietary, highest wins)
2. **Local Preference** (highest wins)
3. **Locally originated** (preferred)
4. **AS Path length** (shortest wins)
5. **Origin code** (IGP < EGP < Incomplete)
6. **MED** (lowest wins)
7. **eBGP over iBGP** (preferred)
8. **Oldest path** (for iBGP)
9. **Router ID** (lowest wins)

<blockquote class="tip">
AS-prepend affects step 4 (AS Path length). Local preference affects step 2. Communities can influence ISP's local preference or MED.
</blockquote>

## Common Issues

| Issue | Solution |
|-------|----------|
| AS-prepend not visible on looking glass | Clear BGP session; wait for propagation; verify route-map is applied |
| Local preference not affecting path | Verify route-map is applied inbound; clear BGP to re-receive routes |
| Communities not being sent | Add `send-community` to neighbor configuration; verify route-map |
| Traffic still using wrong path | Check all BGP attributes; verify no other policies override your changes |
| Changes not taking effect | Clear BGP sessions; verify configuration is saved |

## Troubleshooting Commands

| Command | Purpose |
|---------|---------|
| `show bgp ipv4` | View BGP table with all attributes |
| `show bgp ipv4 <prefix>` | View detailed BGP information for a prefix |
| `show bgp ipv4 neighbors <ip> advertised-routes` | View routes sent to neighbor |
| `show bgp ipv4 neighbors <ip> received-routes` | View routes received from neighbor |
| `show route-map` | View configured route maps |
| `show ip prefix-list` | View configured prefix lists |
| `clear bgp ipv4 <ip> in` | Re-receive routes from neighbor |
| `clear bgp ipv4 <ip> out` | Re-advertise routes to neighbor |
| `debug bgp updates` | Debug BGP updates (use carefully) |

## Configuration Save

Save your configurations on all devices:

```
write memory
```

## Conclusion

In this lab, you learned essential BGP traffic engineering techniques:

- **AS-prepend**: Influences inbound traffic by making paths appear less desirable
- **Local preference**: Controls outbound traffic path selection within your AS
- **BGP communities**: Communicates traffic engineering requests to ISP

These techniques allow you to optimize traffic flow in multi-homed environments, ensuring optimal performance and cost efficiency. In production networks, always coordinate with your ISP when implementing traffic engineering policies.
