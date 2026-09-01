---
layout: lab
title: "Lab 1: IPv4 BGP Peering and Backup Path over OSPF"
lab_number: 1
duration: "90 minutes"
objectives:
  - Configure IPv4 BGP peering with an ISP router (IX-like environment)
  - Advertise your allocated IPv4 prefix to the internet
  - Configure OSPF for internal routing and backup path
  - Implement route filtering for BGP security
  - Verify BGP routes using looking glass tools
prev:
  url: /labs/lab-0
  title: Lab 0 - Environment Setup and FRR Access
next:
  url: /labs/lab-2
  title: Lab 2 - IPv6 Multi-homed BGP with Backup Path over OSPFv3
---

## Introduction

You are a network engineer configuring your enterprise's first BGP peering with an ISP. In this lab, you will establish IPv4 BGP peering with one ISP router, advertise your allocated IPv4 prefix, and configure OSPF to provide a backup path to your main BGP router through internal routers.

This lab focuses on a single-homed IPv4 BGP setup with internal redundancy through OSPF routing.

<blockquote class="tip">
The ISP in this lab behaves like an Internet Exchange (IX) - it does not provide IP transit service. You will receive routes from other peers at the IX, to increase the reliability of the network you will use OSPF to provide an alternate path to your main BGP router. Also you will use OSPF to distribute a default route to your internal network so that the traffic to other peers can be correctly routed to the main BGP router.
</blockquote>

## Prerequisites

Before starting this lab, ensure you have:

- Completed Lab 0 and verified access to all network nodes
- Received from your instructor:
  - Your assigned **AS Number (ASN)**
  - Your allocated **IPv4 prefix**
  - Your **BGP router peering IP**
  - The **ISP's AS Number** and **peering IP**
- Basic understanding of BGP and OSPF concepts by attending or watch this weeks' lecture.

---

## Lab Topology

Your enterprise network consists of:

- **bgp1**: Primary BGP router (connects to ISP1)
- **bgp2**: Secondary BGP router (for internal routing, will be used in Lab 2)
- **ds1, ds2**: Distribution switches for internal routing

The ISP peers with your bgp1 router. Your internal network uses OSPF for routing between BGP routers and distribution switches.

<img src="/routing-lab/assets/image/IPv4_BGP.png" alt="IPv4 BGP Topology" style="width: 100%; max-width: 100%; height: auto;">

## Task 1: Configure IPv4 Addresses on BGP Routers

### Configure bgp1 Interfaces

Connect to bgp1:

```bash
ssh username@172.20.20.11
sudo vtysh
```

<blockquote class="tip">
The interfaces are already assigned to the correct VLANs. You only need to configure the IP addresses on the interfaces.
</blockquote>

Configure the IPv4 addresses on bgp1:

```
configure terminal

interface eth1
ip address <your-isp1-peering-ip>/<mask>
exit

interface eth2
ip address <bgp1-to-bgp2-ip>/<mask>
exit

interface eth3
ip address <bgp1-to-ds1-ip>/<mask>
exit

interface lo
ip address <bgp1-loopback-ip>/24
exit

end
```

Verify the configuration:

```
show interface brief
```

<blockquote class="tip">
There might be minor difference between FRR and Cisco IOS CLI commands. The "show interface brief" command is an example, as FRR does not have "show ip interface brief".
</blockquote>

### Configure bgp2 Interfaces

Connect to bgp2:

```bash
ssh username@172.20.20.12
sudo vtysh
```

Configure the IPv4 addresses on bgp2:

```
configure terminal

interface eth2
ip address <bgp2-to-bgp1-ip>/<mask>
exit

interface eth3
ip address <bgp2-to-ds2-ip>/<mask>
exit

end
```

Verify the configuration:

```
show interface brief
```

### Please finish the interface configuration on ds1 and ds2 according to the lab topology. 

## Task 3: Configure Route Filtering

Route filtering is essential for BGP security, this is also a current RFC: RFC-8212 Default External BGP (EBGP) Route Propagation Behavior without Policies. This RFC states that, without the incoming filter, no routes will be accepted. Without the outgoing filter, no routes will be announced. You need to define route filters to control what routes you advertise and accept. 

### Configure Prefix Lists

On bgp1:

```
configure terminal

ip prefix-list PL_ALLOWED_PREFIX4 seq 10 permit <your-allocated-ipv4-prefix>
ip prefix-list PL_ALLOWED_PREFIX4 seq 20 deny any

ip prefix-list PL_IMPORT_LE24 seq 10 permit 0.0.0.0/0 le 24

end
```

<blockquote class="tip">
- `PL_ALLOWED_PREFIX4` ensures you only advertise your allocated prefix to the ISP
- `PL_IMPORT_LE24` accepts routes with prefix length up to /24, rejecting more specific routes that could be used for attacks and reduce the size the overall internet routing table which is already more than 1M routes there. This is a common practise on the internet where the minimal advertisement prefix length is /24. 
</blockquote>

### Configure Route Maps

On bgp1:

```
configure terminal

route-map RM_EXPORT_OUT4 permit 10
match ip address prefix-list PL_ALLOWED_PREFIX4
exit

route-map RM_IMPORT_IN4 permit 10
match ip address prefix-list PL_IMPORT_LE24
exit

end

```

## Task 4: Configure IPv4 BGP on bgp1

### Enable BGP Process and Configure Neighbor

On bgp1:

```
configure terminal
router bgp <your-asn>
bgp router-id <router-id>

no bgp default ipv4-unicast

neighbor <isp-peering-ip> remote-as <isp-asn>

network <your-allocated-ipv4-prefix>
neighbor <isp-peering-ip> route-map RM_IMPORT_IN4 in
neighbor <isp-peering-ip> route-map RM_EXPORT_OUT4 out

neighbor <isp-peering-ip> activate

end
```

<blockquote class="tip">
The route maps ensure you only advertise your allocated prefix and only accept routes with prefix length up to /24 from the ISP. This is the common practice on the internet where you are only authorized to advertise the prefix you are allocated.
Since the ISP1 behaves like an Internet Exchange (IX), it does not provide IP transit, i.e., internet access. You will not get a full table of the internet or a default route from ISP1.
The `no bgp default ipv4-unicast` command disables the default route advertisement. This will give you the explicit configuration on the BGP, otherwise FRR will automatically add any neighbour to address-family IPv4, including the ones with an IPv6 address. 
</blockquote>

### Verify BGP Session

Check the BGP neighbor status:

```
show ip bgp ipv4 summary
show ip bgp ipv4 neighbors <isp-peering-ip>
```

The neighbor state should show `Established`. If it shows `Active` or `Idle`, check:
- Interface IP configuration
- AS numbers match on both sides
- Network connectivity to the ISP1 peering IP

## Task 5: Configure OSPF on bgp1

OSPF provides routing between your BGP routers and distribution switches, creating a backup path if one of the link on the primary BGP router becomes unavailable.

### Enable OSPF Process

On bgp1:

```
configure terminal
router ospf
ospf router-id <router-id>
log-adjacency-changes
default-information originate always
end
```

<blockquote class="tip">
The `default-information originate always` command advertises a default route to OSPF neighbors, even if the router doesn't have a default route in its routing table. This ensures internal devices always send traffic not in their routing table to bgp1.
</blockquote>

### Enable OSPF on Interfaces

On bgp1, enable OSPF on internal interfaces and loopback:

```
configure terminal

interface eth2
ip ospf area 0
exit

interface eth3
ip ospf area 0
exit

interface lo
ip ospf area 0
exit

end
```

## Task 6: Configure OSPF on bgp2

Configure OSPF on bgp2 to participate in internal routing:

### Enable OSPF Process

On bgp2:

```
configure terminal
router ospf
ospf router-id <router-id>
log-adjacency-changes
end
```

### Enable OSPF on Interfaces

On bgp2:

```
configure terminal

interface eth2
ip ospf area 0
exit

interface eth3
ip ospf area 0
exit

end
```

## Task 7: Configure OSPF on Distribution Switches

Configure OSPF on ds1:

```
configure terminal
router ospf
ospf router-id <router-id>

interface eth1
ip ospf area 0
exit

interface eth2
ip ospf area 0
exit

end
```

Configure OSPF on ds2:

```
configure terminal
router ospf
ospf router-id <router-id>

interface eth1
ip ospf area 0
exit

interface eth2
ip ospf area 0
exit

end
```

### Verify OSPF Neighbors

Check OSPF neighbor relationships:

```
show ip ospf neighbor
```

You should see neighbor relationships in `Full` state.

### Verify OSPF Routes

Check the routing table for OSPF-learned routes:

```
show ip route ospf
```

Routes learned via OSPF will be marked with `O`.

## Task 8: Verify BGP Operation

### Test BGP Route Advertisement

From bgp1, verify your prefix is being advertised:

```
show ip bgp ipv4 summary
show ip bgp ipv4 neighbors <isp1-peering-ip> advertised-routes
```

## Task 9: Verify Route Reception Using Looking Glass

The looking glass server at `151.158.219.14` (accessible within the university network) allows you to verify your BGP announcements from an external perspective.

### Query Your Prefix

Use the looking glass to search for your ASN and advertised routes:

1. Select rs1 in the left panel, this is your isp1's routing information
2. Search for your ASN in the right panel
3. Verify that:
   - Your prefix is visible in the ISP's routing table
   - The AS path shows your AS number
   - The next-hop (gateway) is correct

<blockquote class="warning">
The looking glass at 151.158.219.14 is only accessible from within the university network. If you cannot access it, verify you are connected via the university network.
</blockquote>

## Task 10: Test OSPF Backup Path

To verify the OSPF backup path works correctly:

### Verify Internal Routing

From ds1, verify you can reach bgp1's loopback via OSPF:

```
ping <bgp1-loopback-ip>
traceroute <bgp1-loopback-ip>
```

### Simulate Link Failure

Temporarily shut down the link on bgp1 to ds1:

```
configure terminal
interface eth3
shutdown
end
```

### Verify Internal Connectivity

Check that ds1 can still reach bgp1's loopback:

```
show ip route ospf
traceroute <bgp1-loopback-ip>
```

See the difference between the traceroute paths before and after the link failure.

### Restore Link Connection

After testing the backup path, re-enable the interface on bgp1:

```
configure terminal
interface eth3
no shutdown
end
```

## Task 11: Verify Learned BGP Routes and Connectivity to Peers

### Check Received BGP Routes
First, verify that you have successfully received IPv4 prefixes from other peers connected to the ISP/IX. On bgp1, view all learned BGP routes:

```
show ip bgp ipv4
```

### Test Connectivity to Peer Networks
From any internal router (e.g., ds1), test connectivity to an IPv4 prefix learned from another peer at the IX to confirm end-to-end routing works:

```
ping <ip-from-learned-ipv4-prefix>
traceroute <ip-from-learned-ipv4-prefix>
```


## Verification Checklist

- [ ] IPv4 addresses configured on all interfaces (bgp1, bgp2, ds1, ds2)
- [ ] Prefix lists and route maps configured for route filtering
- [ ] BGP session with ISP is in `Established` state
- [ ] Your allocated IPv4 prefix is advertised to the ISP
- [ ] Routes from IX peers are received
- [ ] OSPF neighbors are in `Full` state on all routers
- [ ] OSPF routes are present in the routing table
- [ ] Default IPv4 route is propagated to internal routers via OSPF
- [ ] Your prefix is visible on the looking glass
- [ ] Connectivity to other peers is working
- [ ] Configuration saved on all devices

## Common Issues

| Issue | Solution |
|-------|----------|
| BGP neighbor stuck in `Idle` | Check interface IP configuration and verify network connectivity to ISP peering IP |
| BGP neighbor stuck in `Active` | Verify AS numbers match on both sides; check for firewall blocking TCP port 179 |
| Prefix not advertised | Ensure the network statement matches exactly; verify the route exists |
| OSPF neighbors not forming | Check that interfaces have `ip ospf area 0` configured |
| Looking glass shows no route | Wait a few minutes for propagation; verify BGP session is established |
| Route filtering not working | Verify prefix-lists and route-maps are correctly applied to neighbors |

## Troubleshooting Commands

| Command | Purpose |
|---------|---------|
| `show ip bgp ipv4 summary` | View BGP neighbor status |
| `show ip bgp ipv4` | View BGP routing table |
| `show ip bgp ipv4 neighbors <ip> advertised-routes` | View routes sent to neighbor |
| `show ip ospf neighbor` | View OSPF neighbor relationships |
| `show ip route` | View complete routing table |
| `show ip route ospf` | View OSPF-learned routes |
| `show interface brief` | View IP interface status |
| `show ip prefix-list` | View configured prefix lists |
| `show route-map` | View configured route maps |

## Configuration Save

Save your configurations on all devices:

```
write memory
```

Or:

```
copy running-config startup-config
```

## Conclusion

In this lab, you successfully configured IPv4 BGP peering with an ISP router and established OSPF routing for internal redundancy. You learned how to:

- Configure IPv4 BGP peering with an ISP (IX-like environment)
- Implement route filtering using prefix-lists and route-maps
- Configure OSPF for IPv4 internal routing and backup paths
- Advertise default routes to internal network via OSPF
- Verify BGP announcements using a looking glass
- Test and troubleshoot BGP and OSPF configurations

<blockquote class="tip">
Since the ISP behaves like an Internet Exchange (IX), it does not provide transit or a default route. You receive routes from other peers at the IX, and use `default-information originate always` in OSPF to provide a default route to internal devices.
</blockquote>

In the next lab, you will extend this configuration to support IPv6 multi-homed BGP with OSPFv3 for IPv6 infrastructure.
