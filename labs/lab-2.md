---
layout: lab
title: "Lab 2: IPv6 Multi-homed BGP with Backup Path over OSPFv3"
lab_number: 2
duration: "120 minutes"
objectives:
  - Configure multi-homed IPv6 BGP peering with two ISPs
  - Advertise your allocated IPv6 prefix to the internet
  - Configure OSPFv3 for IPv6 internal routing and backup paths
  - Implement route filtering for BGP security
  - Verify IPv6 BGP routes using looking glass and public BGP tools
prev:
  url: /labs/lab-1
  title: Lab 1 - IPv4 BGP Peering and Backup Path over OSPF
next:
  url: /labs/lab-3
  title: Lab 3 - BGP Traffic Engineering
---

## Introduction

Building on Lab 1, you will now configure multi-homed IPv6 BGP peering with two different ISPs. This provides redundancy and ensures your IPv6 network remains accessible even if one ISP connection fails. You will also configure OSPFv3 to provide backup paths between your BGP routers.

Multi-homing is a critical design pattern for enterprise networks that require high availability and optimal path selection.

<blockquote class="warning">
This lab involves modifying BGP policies that affect routes advertised to the global internet. Always follow these security guidelines:
1. Never share sensitive network and server details (including your ASN, internal addressing, username, password, etc.) in public
2. Ensure route filtering is properly implemented to avoid unintentionally advertising invalid routes to the global internet
</blockquote>

## Prerequisites

Before starting this lab, ensure you have:

- Completed Lab 1 (IPv4 BGP and OSPF configuration)
- Received from your instructor:
  - Your assigned **AS Number (ASN)**
  - Your allocated **IPv6 prefix**
  - **Peering IPs for both bgp1 and bgp2**
  - The **ISP AS numbers** and **peering IPs** for both connections
- Understanding of IPv6 addressing and routing concepts

## Lab Topology

Your enterprise network now has dual ISP connections:

- **bgp1**: Primary BGP router (connects to ISP1)
- **bgp2**: Secondary BGP router (connects to ISP2)
- **ds1, ds2**: Distribution switches for internal routing

Both BGP routers peer with ISP routers on separate connections, providing redundancy. OSPFv3 provides routing between BGP routers and distribution switches. 

<img src="/routing-lab/assets/image/IPv6_BGP.png" alt="IPv6 BGP Topology" style="width: 100%; max-width: 100%; height: auto;">

## Task 1: Configure IPv6 Addresses on BGP Routers

### Configure bgp1 Interfaces

Connect to bgp1:

```bash
ssh username@172.20.20.11
sudo vtysh
```

<blockquote class="tip">
The interfaces are already assigned to the correct VLANs. You only need to configure the IPv6 addresses on the interfaces.
</blockquote>

Configure the IPv6 addresses on bgp1:

```
configure terminal

interface eth1
ipv6 address <your-isp1-peering-ipv6>/<mask>
exit

interface eth2
ipv6 address <bgp1-to-bgp2-ipv6>/<mask>
exit

interface eth3
ipv6 address <bgp1-to-ds1-ipv6>/<mask>
exit

interface lo
ipv6 address <bgp1-loopback-ipv6>/128
exit

ipv6 forwarding

end
```

Verify the configuration:

```
show interface brief
show interface brief json
```

<blockquote class="tip">
"ipv6 forwarding" enables IPv6 forwarding on the interface. You will need to enable this function on all routers and switches to ensure IPv6 traffic can be forwarded between them. 
</blockquote>

### Configure bgp2 Interfaces

Connect to bgp2:

```bash
ssh username@172.20.20.12
sudo vtysh
```

Configure the IPv6 addresses on bgp2:

```
configure terminal

interface eth1
ipv6 address <your-isp2-peering-ipv6>/<mask>
exit

interface eth2
ipv6 address <bgp2-to-bgp1-ipv6>/<mask>
exit

interface eth3
ipv6 address <bgp2-to-ds2-ipv6>/<mask>
exit

interface lo
ipv6 address <bgp2-loopback-ipv6>/128
exit

end
```

Verify the configuration:

```
show interface brief
show interface brief json
```

### Please finish the interface configuration on ds1 and ds2 according to the lab topology. 

## Task 2: Configure Null Route for Advertised Prefix

Create a static null route for your allocated prefix. This ensures the prefix exists in the routing table for BGP to advertise:

On bgp1 and bgp2:

```
configure terminal
ipv6 route <your-allocated-ipv6-prefix> Null0
end
```

## Task 3: Configure Route Filtering

Route filtering is essential for BGP security. You need to control what routes you advertise and accept.

### Configure Prefix Lists

On bgp1 and bgp2:

```
configure terminal

ipv6 prefix-list PL_ALLOWED_PREFIX6 seq 10 permit <your-allocated-ipv6-prefix>
ipv6 prefix-list PL_ALLOWED_PREFIX6 seq 20 deny any

ipv6 prefix-list PL_IMPORT_GT48 seq 10 permit ::/0 le 48

end
```

<blockquote class="tip">
- `PL_ALLOWED_PREFIX6` ensures you only advertise your allocated prefix to ISPs
- `PL_IMPORT_GT48` accepts routes with prefix length up to /48, this is the common practice on the internet where the minimal prefix size is /48.
</blockquote>

### Configure Route Maps

On bgp1 and bgp2:

```
configure terminal

route-map RM_EXPORT_OUT6 permit 10
match ipv6 address prefix-list PL_ALLOWED_PREFIX6
exit

route-map RM_IMPORT_IN6 permit 10
match ipv6 address prefix-list PL_IMPORT_GT48
exit

end
```

## Task 4: Configure IPv6 BGP on bgp1

### Enable BGP Process and Configure Neighbors

On bgp1:

```
configure terminal
router bgp <your-asn>
bgp router-id <router-id>
no bgp default ipv4-unicast

neighbor <bgp2-loopback-ipv6> remote-as <your-asn>
neighbor <bgp2-loopback-ipv6> update-source lo

neighbor <isp1-ipv6-peering-ip> remote-as <isp1-asn>

address-family ipv6 unicast
network <your-allocated-ipv6-prefix>

neighbor <bgp2-loopback-ipv6> next-hop-self
neighbor <bgp2-loopback-ipv6> activate

neighbor <isp1-ipv6-peering-ip> route-map RM_IMPORT_IN6 in
neighbor <isp1-ipv6-peering-ip> route-map RM_EXPORT_OUT6 out
neighbor <isp1-ipv6-peering-ip> activate
exit-address-family

end
```

<blockquote class="tip">
The eBGP neighbor establishes a peering session with ISP1, while the iBGP peering session is configured with your internal bgp2 router. The iBGP neighbor uses the loopback address with `update-source lo` for stability and can utilize the alternative path from OSPFv3. The `next-hop-self` command ensures the iBGP neighbor advertises routes with the loopback address as the next hop, which is the default behavior.
</blockquote>

### Verify BGP Session

Check the IPv6 BGP neighbor status:

```
show bgp ipv6 summary
show bgp ipv6 neighbors
```

The neighbor state should show `Established`. Since your ISP is advertising the full table, which means it will give you all routes of the internet. Currently, the internet has about 250,000 routes. This means please try not use commands like `show ipv6 route` or `show ip bgp ipv6` to check the routing table. Instead, use commands like `show ipv6 route <query-ipv6-prefix>` to check the route to the prefix you want to know.

## Task 5: Configure IPv6 BGP on bgp2

### Enable BGP Process and Configure Neighbors

On bgp2:

```
configure terminal
router bgp <your-asn>
bgp router-id <router-id>
no bgp default ipv4-unicast

neighbor <bgp1-loopback-ipv6> remote-as <your-asn>
neighbor <bgp1-loopback-ipv6> update-source lo

neighbor <isp2-ipv6-peering-ip> remote-as <isp2-asn>

address-family ipv6 unicast
network <your-allocated-ipv6-prefix>

neighbor <bgp1-loopback-ipv6> next-hop-self
neighbor <bgp1-loopback-ipv6> activate

neighbor <isp2-ipv6-peering-ip> route-map RM_IMPORT_IN6 in
neighbor <isp2-ipv6-peering-ip> route-map RM_EXPORT_OUT6 out 
neighbor <isp2-ipv6-peering-ip> activate
exit-address-family

end
```

### Verify BGP Session

Check the IPv6 BGP neighbor status:

```
show bgp ipv6 summary
show bgp ipv6 neighbors
```

You should see both external (eBGP) and internal (iBGP) neighbors in `Established` state.

## Task 6: Configure OSPFv3 on bgp1

OSPFv3 provides routing between your BGP routers and distribution switches for IPv6, creating backup paths if one BGP router becomes unavailable.

### Enable OSPFv3 Process

On bgp1:

```
configure terminal
router ospf6
ospf6 router-id <router-id>
log-adjacency-changes
default-information originate always
end
```

<blockquote class="tip">
The `default-information originate always` command advertises a default route to OSPF neighbors, even if the router doesn't have a default route in its routing table. This ensures internal devices always have a path to the internet.
</blockquote>

### Enable OSPFv3 on Interfaces

On bgp1, enable OSPFv3 on internal interfaces and loopback:

```
configure terminal

interface eth2
ipv6 ospf6 area 0
exit

interface eth3
ipv6 ospf6 area 0
exit

interface lo
ipv6 ospf6 area 0
exit

end
```

## Task 7: Configure OSPFv3 on bgp2

### Enable OSPFv3 Process

On bgp2:

```
configure terminal
router ospf6
ospf6 router-id <router-id>
log-adjacency-changes
default-information originate always
end
```

### Enable OSPFv3 on Interfaces

On bgp2:

```
configure terminal

interface eth2
ipv6 ospf6 area 0
exit

interface eth3
ipv6 ospf6 area 0
exit

interface lo
ipv6 ospf6 area 0
exit

end
```

## Task 8: Configure OSPFv3 on Distribution Switches

Configure OSPFv3 on ds1:

```
configure terminal
router ospf6
ospf6 router-id <router-id>

interface eth1
ipv6 ospf6 area 0
exit

interface eth2
ipv6 ospf6 area 0
exit

end
```

Configure OSPFv3 on ds2:

```
configure terminal
router ospf6
ospf6 router-id <router-id>

interface eth1
ipv6 ospf6 area 0
exit

interface eth2
ipv6 ospf6 area 0
exit

end
```

### Verify OSPFv3 Neighbors

Check OSPFv3 neighbor relationships:

```
show ipv6 ospf6 neighbor
```

You should see neighbor relationships in `Full` state.

### Verify OSPFv3 Routes

Check the IPv6 routing table for OSPFv3-learned routes:

```
show ipv6 route ospf6
```

Routes learned via OSPFv3 will be marked with `O`.

## Task 9: Verify Multi-homed BGP Operation

### Check BGP Routes on bgp1

```
show ip bgp ipv6 summary
show ip bgp ipv6 neighbors <isp1-ipv6-peering-ip> advertised-routes
show ip bgp ipv6 statistics

```

### Check BGP Routes on bgp2

```
show ip bgp ipv6 summary
show ip bgp ipv6 neighbors <isp2-ipv6-peering-ip> advertised-routes
show ip bgp ipv6 statistics
```

### Verify Both Paths

Check that both ISP paths are available:

```
show ip bgp ipv6 2001:4860:4860::8888
```

2001:4860:4860::8888 is an example target network address. You should see multiple paths with different next-hops. If not, please try it on another BGP router. 

## Task 10: Verify Using Looking Glass

### Internal Looking Glass

The looking glass server at `151.158.219.14` (accessible within the university network) allows you to verify your BGP announcements from an external perspective.

Use the looking glass to search for your ASN and advertised routes:

1. Select rs1 or rs2 in the left panel, this is your isp1 or isp2's routing information
2. Search for your ASN in the right panel
3. Verify that:
   - Your prefix is visible in the ISP's routing table
   - The AS path shows your AS number
   - The next-hop (gateway) is correct

<blockquote class="warning">
The looking glass at 151.158.219.14 is only accessible from within the university network. If you cannot access it, verify you are connected via the university network.
</blockquote>

### Public BGP Looking Glasses

Pick up one of the public BGP looking glasses to verify your IPv6 prefix is announced globally, please do not generate a lot of query in a short time, since those services are shared resources: 

1. **Route Views** (https://lg.routeviews.org/lg)
2. **Hurricane Electric BGP Toolkit** (https://bgp.he.net/)
3. **BGPlay** (https://stat.ripe.net/widget/bgplay)
4. **bgp.tools** (https://bgp.tools/)
5. **NTT Data Looking Glass** (https://www.gin.ntt.net/looking-glass-landing/)

Query your allocated IPv6 prefix and verify:
- The prefix is visible from multiple locations
- Both ISP AS numbers appear in different paths

<blockquote class="tip">
BGP propagation can take several minutes. If your prefix doesn't appear immediately, wait 5-10 minutes and try again.
</blockquote>

## Task 11: Test Failover and Backup Path

### Simulate bgp1 Failure

We will simulate a failure of bgp1 to test failover and backup path.

Check the bgp route on bgp2:

```
show ip bgp ipv6 2001:4860:4860::8888
```

Test connectivity and path selection on ds1:
  
```
ping ipv6 2001:4860:4860::8888
ping ipv6 2606:4700:4700::1111
traceroute ipv6 2001:4860:4860::8888
traceroute ipv6 2606:4700:4700::1111
```

On bgp1, shut down the BGP session:

```
configure terminal
router bgp <your-asn>
address-family ipv6 unicast
neighbor <isp1-ipv6-peering-ip> shutdown
exit-address-family
end
```

### Verify Traffic Rerouting

Check that traffic now flows through bgp2:

```
show ip bgp ipv6 2001:4860:4860::8888
```

On ds1, verify connectivity and path selection:

```
ping ipv6 2001:4860:4860::8888
ping ipv6 2606:4700:4700::1111
traceroute ipv6 2001:4860:4860::8888
traceroute ipv6 2606:4700:4700::1111
```

### Check your connectivity on global routing table and internal looking glass

Verify that your IPv6 prefix is visible on the global routing table and internal looking glass:

### Verify prefix visibility on the looking glass
Confirm your IPv6 prefix appears in both the internal university looking glass and public BGP tools with these step-by-step checks:
1. On the internal looking glass, query your prefix and validate:
   - Your prefix in only visible on rs2's routing table
2. Use one of the public BGP looking glasses to confirm global propagation:
   - Search for your IPv6 prefix and verify the status of the two upstreams and see the change over time
   - It would take some time to see one of your upstream disapper in the global routing table, this might create blackhold routes, think about how to mitigate this problem?


### Restore bgp1

```
configure terminal
router bgp <your-asn>
address-family ipv6 unicast
no neighbor <isp1-ipv6-peering-ip> shutdown
exit-address-family
end
```

### Verify Reroute after restoration

Check that traffic now flows through bgp1:

After restoration, verify both paths are available:

```
show ip bgp ipv6 2001:4860:4860::8888
```

## Task 12: Verify OSPFv3 Backup Path

### Test Internal Routing

From bgp1, verify you can reach bgp2's loopback via OSPFv3:

```
ping ipv6 <bgp2-loopback-ipv6>
traceroute ipv6 <bgp2-loopback-ipv6>
```

### Test Distribution Switch Connectivity

Verify ds1 and ds2 can reach both BGP routers:

```
show ipv6 route ospf6
ping ipv6 <bgp1-loopback-ipv6>
ping ipv6 <bgp2-loopback-ipv6>
```

### Simulate Link Failure

Temporarily shut down the link on bgp1 to bgp2:

```
configure terminal
interface eth2
shutdown
end
```

### Verify Internal Connectivity

Check that bgp1 can still reach bgp2's loopback:

```
show ip route ospf
traceroute <bgp2-loopback-ip>
```

See the difference between the traceroute paths before and after the link failure.

### Restore Link Connection

After testing the backup path, re-enable the interface on bgp1:

```
configure terminal
interface eth2
no shutdown
end
```

## Verification Checklist

- [ ] IPv6 addresses configured on all interfaces (bgp1, bgp2, ds1, ds2)
- [ ] Null route configured for allocated prefix on bgp1 and bgp2
- [ ] Prefix lists and route maps configured for route filtering
- [ ] IPv6 BGP session with ISP1 on bgp1 is in `Established` state
- [ ] IPv6 BGP session with ISP2 on bgp2 is in `Established` state
- [ ] iBGP session between bgp1 and bgp2 is established using loopback addresses
- [ ] Your allocated IPv6 prefix is advertised to both ISPs
- [ ] BGP routes from both ISPs are received and installed
- [ ] OSPFv3 neighbors are in `Full` state on all routers
- [ ] OSPFv3 routes are present in the IPv6 routing table
- [ ] Default IPv6 route is propagated to internal routers via OSPFv3
- [ ] IPv6 internet connectivity is working
- [ ] Your IPv6 prefix is visible on the internal looking glass
- [ ] Your IPv6 prefix is visible on public BGP looking glasses
- [ ] Failover to backup path works correctly
- [ ] Configuration saved on all devices

## Common Issues

| Issue | Solution |
|-------|----------|
| IPv6 BGP neighbor stuck in `Idle` | Check IPv6 interface configuration |
| IPv6 BGP neighbor stuck in `Active` | Verify AS numbers match |
| iBGP session not establishing | Verify both routers use the same AS number; check `update-source lo` configuration |
| Routes not advertised | Ensure the network statement matches exactly; verify the null route exists |
| OSPFv3 neighbors not forming | Check that interfaces have `ipv6 ospf6 area 0` configured; verify link-local addresses |
| No IPv6 internet connectivity | Verify BGP routes are received; check IPv6 routing table for default route |
| Prefix not visible on internal looking glass | Verify both BGP sessions are established |
| Failover not working | Verify iBGP is configured correctly; check `next-hop-self` is configured |
| Route filtering not working | Verify prefix-lists and route-maps are correctly applied to neighbors |

## Troubleshooting Commands

| Command | Purpose |
|---------|---------|
| `show ip bgp ipv6 summary` | View IPv6 BGP neighbor status |
| `show ip bgp ipv6` | View full IPv6 BGP routing table (use carefully) |
| `show ip bgp ipv6 neighbors <ip> advertised-routes` | View IPv6 routes sent to neighbor |
| `show ip bgp ipv6 <ip>` | View IPv6 routes for specfic ip |
| `show ipv6 ospf6 neighbor` | View OSPFv3 neighbor relationships |
| `show ipv6 route` | View complete IPv6 routing table (use carefully) |
| `show ipv6 route ospf6` | View OSPFv3-learned routes |
| `show interface brief` | View interface status |
| `show ipv6 prefix-list` | View configured prefix lists |
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

In this lab, you successfully configured multi-homed IPv6 BGP peering with two ISPs and established OSPFv3 routing for internal redundancy. You learned how to:

- Configure IPv6 BGP peering with multiple ISPs
- Set up iBGP between BGP routers using loopback interfaces
- Configure OSPFv3 for IPv6 internal routing and backup paths
- Implement route filtering using prefix-lists and route-maps
- Advertise default routes to internal network via OSPFv3
- Verify IPv6 BGP announcements using looking glass tools
- Test failover and backup path functionality

In the next lab, you will learn how to use BGP traffic engineering techniques to control traffic flow across your multi-homed connections.
