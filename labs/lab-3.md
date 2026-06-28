---
layout: lab
title: "Lab 3: BGP Traffic Engineering"
lab_number: 3
duration: "120 minutes"
objectives:
  - Use AS-prepend to influence inbound traffic path selection
  - Configure local preference to control outbound traffic
  - Inspect BGP communities for traffic engineering and route tagging
  - Troubleshoot and optimize traffic flow based on requirements
prev:
  url: /labs/lab-2
  title: Lab 2 - IPv6 Multi-homed BGP with Backup Path over OSPFv3
next:
  url: /
  title: Home
---

## Introduction

In [Lab 2](/routing-lab/labs/lab-2), you configured multi-homed IPv6 BGP peering with two ISPs and established OSPFv3 for internal redundancy. In a multi-homed environment, both ISPs advertise routes to the same destinations, and by default BGP selects paths based on shortest AS path. However, the shortest path is not always the best path — you may want to prefer one ISP for cost reasons, latency, or capacity.

This lab focuses on BGP traffic engineering techniques that allow you to influence path selection for both inbound and outbound traffic. You will work through realistic scenarios where specific prefixes require traffic optimization, similar to real-world network operations tickets.

<blockquote class="warning">
This lab involves modifying BGP policies that affect routes advertised to the global internet. Always follow these security guidelines:
1. Never share sensitive network and server details (including your ASN, internal addressing, username, password, etc.) in public
2. Ensure route filtering is properly implemented to avoid unintentionally advertising invalid routes to the global internet
3. Think carefully before applying changes that affect your advertisement of routes to the global internet
</blockquote>

## Prerequisites

Before starting this lab, ensure you have:

- Completed [Lab 1](/routing-lab/labs/lab-1) (IPv4 BGP Peering and Backup Path over OSPF) — in particular, the route filtering concepts in [Task 3](/routing-lab/labs/lab-1#task-3-configure-route-filtering)
- Completed [Lab 2](/routing-lab/labs/lab-2) (IPv6 Multi-homed BGP with Backup Path over OSPFv3) — in particular, the route filtering in [Task 3](/routing-lab/labs/lab-2#task-3-configure-route-filtering), iBGP configuration in [Task 4](/routing-lab/labs/lab-2#task-4-configure-ipv6-bgp-on-bgp1) and [Task 5](/routing-lab/labs/lab-2#task-5-configure-ipv6-bgp-on-bgp2)
- Working multi-homed BGP setup with both bgp1 and bgp2 peering to ISP1 and ISP2 respectively
- iBGP session between bgp1 and bgp2 is established (configured in Lab 2)
- Understanding of BGP path attributes (AS_PATH, LOCAL_PREF, COMMUNITIES and the rest of the attributes in BGP)
- Received from your instructor:
  - Your assigned **AS Number (ASN)**
  - Your allocated **IPv6 prefix**

## Lab Topology

This lab uses the same topology you built in Lab 2. Your multi-homed network has two ISP connections:

- **bgp1**: Primary BGP router (connects to ISP1)
- **bgp2**: Secondary BGP router (connects to ISP2)
- **ds1, ds2**: Distribution switches for internal routing

Both BGP routers peer with ISP routers on separate connections, providing redundancy. OSPFv3 provides routing between BGP routers and distribution switches.

<img src="/routing-lab/assets/image/IPv6_BGP.png" alt="IPv6 BGP Topology" style="width: 100%; max-width: 100%; height: auto;">

<blockquote class="tip">
This lab focuses on IPv6 traffic engineering, building on the IPv6 BGP and OSPFv3 configuration you completed in Lab 2. The same traffic engineering techniques also apply to IPv4 with the corresponding `address-family` and prefix-list commands.
</blockquote>

## Scenario: Traffic Optimization Tickets

You have received several tickets from the network operations team requiring traffic optimization for specific prefixes. Each ticket describes a traffic flow issue that needs to be resolved using BGP traffic engineering techniques.

---

## Ticket 1: Inbound Traffic Optimization Using AS-Prepend

### Problem Statement

One of ISP1's major IP transit providers is currently experiencing issues. Although ISP1 is the cheaper option, the increased latency through their upstream is causing performance problems for your inbound traffic. You want to influence the internet to slightly prefer ISP2 for traffic entering your network.

### Background

AS-prepend adds your AS number multiple times to the AS_PATH attribute, making the path appear longer and less desirable. This influences other ASes to prefer alternative paths. AS-prepend affects step 4 of the BGP path selection process (AS Path length — shortest wins).

In this case, you want inbound traffic to prefer ISP2 (bgp2). To achieve this, you make the path through ISP1 (bgp1) appear less desirable by prepending your AS number to routes advertised to ISP1.

<blockquote class="tip">
AS-prepend is a "hint" to other networks — it makes your path less desirable but does not guarantee that all networks will switch. Some networks may still prefer the prepended path due to their own local policies or if the alternative path is even longer. Typically, 1-2 prepends are sufficient for a slight preference shift. More than 5 prepends is rarely needed and can cause issues.
</blockquote>

### Task 1.1: Analyze Current Inbound Traffic Path

AS-prepend is a technique to influence **outside ASes** — you cannot verify its effect from inside your own routers. You need to check how external networks see your prefix. Before making any changes, establish a baseline by checking how your prefix is currently seen from the outside.

**Internal Looking Glass**

Use the looking glass at `151.158.219.14` to check how your prefix is seen from the ISP's perspective (as you did in [Lab 2, Task 10](/routing-lab/labs/lab-2#task-10-verify-using-looking-glass)):

1. Select rs1 in the left panel — this shows ISP1's view of your prefix
2. Search for your ASN in the right panel
3. Note the AS path — this is how ISP1 sees your network
4. Select rs2 in the left panel — this shows ISP2's view
5. Note the AS path from ISP2's perspective

Currently, both paths should have equal AS path length from your AS, meaning external networks have no strong reason to prefer one over the other.

<blockquote class="warning">
The looking glass at 151.158.219.14 is only accessible from within the university network. If you cannot access it, verify you are connected via the university network.
</blockquote>

**Public BGP Looking Glasses**

Also verify your prefix visibility from the global internet using public looking glasses (as you did in [Lab 2, Task 10](/routing-lab/labs/lab-2#task-10-verify-using-looking-glass)). Please pick up one of the following public looking glasses:

1. **Route Views** (https://lg.routeviews.org/lg)
2. **Hurricane Electric BGP Toolkit** (https://bgp.he.net/)
3. **BGPlay** (https://stat.ripe.net/widget/bgplay)
4. **bgp.tools** (https://bgp.tools/)
5. **NTT Data Looking Glass** (https://www.gin.ntt.net/looking-glass-landing/)

Search for your IPv6 prefix and note the AS path from different vantage points. This baseline will help you compare after applying AS-prepend. The geo location of the looking glass server will also affects the AS path you observe. For example, if you are in Japan, you may see a different AS path than if you are in the US.

<blockquote class="tip">
BGP propagation can take several minutes. If your prefix doesn't appear immediately, wait 5-10 minutes and try again. Please do not generate a lot of queries in a short time, since those services are shared resources. 
</blockquote>

### Task 1.2: Configure AS-Prepend on bgp1

Since you want inbound traffic to prefer ISP2, you need to make the path through ISP1 (bgp1) less desirable. Configure AS-prepend on bgp1 for your allocated prefix:

```
configure terminal

route-map RM_EXPORT_OUT6 permit 10
 match ipv6 address prefix-list PL_ALLOWED_PREFIX
 set as-path prepend 64632
exit

# BGP has configured to use RM_EXPORT_OUT6 for export filtering
end
```

<blockquote class="tip">
A single prepend (`<your-asn>` once) makes the path through ISP1 slightly less desirable — just enough to tip the balance toward ISP2 without making ISP1 completely undesirable. If you need a stronger shift, you can prepend multiple times (e.g., `<your-asn> <your-asn> <your-asn>` for 3 prepends). The second sequence (20) allows all other prefixes to be advertised normally without prepending.

In Lab 2, you already configured route-maps for route filtering (e.g., `RM_EXPORT_OUT6`). When applying a new route-map to a neighbor, it replaces the previous one. If you need to combine filtering with AS-prepend, merge the match conditions into a single route-map. For example, include both the prefix-list match from your export filter and the `set as-path prepend` statement in the same route-map entry.
</blockquote>

### Task 1.3: Verify AS-Prepend

Verify the AS path on the looking glass. You should see your AS number appearing one extra time in the path through ISP1, making it slightly longer than the path through ISP2.

---

## Ticket 2: Outbound Traffic Control Using Local Preference

### Problem Statement

The development team needs a preferred outbound path to `2001:df6:bf40::/48` via ISP1. They are uploading large reference files to a service in that network, and ISP1's uplink on your site has significantly better capacity for this traffic. 

### Background

Local preference is a BGP attribute that indicates the preferred path for outbound traffic within your AS. Higher local preference values are preferred. Local preference is only shared within an AS (via iBGP) — it is not advertised to external peers. 

Since you configured iBGP between bgp1 and bgp2 in [Lab 2, Task 4](/routing-lab/labs/lab-2#task-4-configure-ipv6-bgp-on-bgp1), local preference values set on one router will be propagated to the other via iBGP, allowing both routers to make consistent outbound path decisions.

### Task 2.1: Analyze Current Outbound Path

Check the current path selection for the destination prefix:

```
show ip bgp ipv6 2001:df6:bf40::/48
```

Look at the `localpref` and the path information. Note which ISP path is currently preferred and why. 

Test outbound traffic to observe which path is preferred from both distribution switches and think about why.

```
traceroute ipv6 2001:df6:bf40:1357::100
```

### Task 2.2: Configure Local Preference for the Destination

To ensure outbound traffic to `2001:df6:bf40::/48` uses ISP1 (bgp1), set a higher local preference on routes received from ISP1 that match this prefix.

In [Lab 2, Task 3](/routing-lab/labs/lab-2#task-3-configure-route-filtering), you configured the import route-map `RM_IMPORT_IN6` which matches `PL_IMPORT_GT48` to filter incoming routes. You need to add a new sequence to this existing route-map so that the local preference is set before the import filter allows the route through.

On bgp1:

```
configure terminal
ipv6 prefix-list DEV-DEST6 seq 5 permit 2001:df6:bf40::/48

route-map RM_IMPORT_IN6 permit 5
match ipv6 address prefix-list DEV-DEST6
set local-preference 150
exit

end
```

By inserting sequence number 5 before the existing sequence 10, the local preference is set for the dev team's destination prefix before the general import filter applies. Routes matching `DEV-DEST6` will get local preference 150 and still pass through the import filter (since `2001:df6:bf40::/48` is within the `/0 le 48` range allowed by `PL_IMPORT_GT48`). All other routes continue to be processed by sequence 10 with the default local preference of 100. Since local preference is propagated via iBGP, both bgp1 and bgp2 will see the higher local preference and consistently route traffic to `2001:df6:bf40::/48` via ISP1.


### Task 2.3: Verify Local Preference

Verify the local preference values on bgp1 and bgp2:

```
show ip bgp ipv6 2001:df6:bf40::/48
```

The route should now show `localpref: 150` on both BGP routers.

### Task 2.4: Verify Outbound Traffic Path

Test outbound traffic to verify it uses the preferred path from both distribution switches:

```
traceroute ipv6 2001:df6:bf40:1357::100
```

Traffic should now exit via bgp1/ISP1.

---

## Ticket 3: Traffic Engineering Using BGP Communities

### Problem Statement

ISP2 has announced a new discount program: traffic to their IXP peers is offered at a significantly reduced cost. Routes to IXP peers are tagged with the BGP community `20473:200`. You want to take advantage of this discount by giving higher local preference (120) to routes tagged with this community, so that traffic to those destinations prefers ISP2.

### Background

BGP communities are optional transitive attributes that allow you to tag routes with additional information. ISPs use communities to provide information about routes — such as where the route was learned, the geographic region, or the type of peering relationship.

In this case, ISP2 tags routes learned from their IXP peers with community `20473:200`. By matching on this community value in your import policy, you can give those routes a higher local preference, causing your outbound traffic to prefer ISP2 for those destinations — and taking advantage of the discounted pricing.

### Task 3.1: Inspect Communities on Received Routes

Before configuring a policy, verify that ISP2 is actually tagging routes with community `20473:200`. Check a few routes received from ISP2:

```
show ip bgp ipv6 2001:4860:4860::8888
show ip bgp ipv6 2606:4700:4700::1111
```

Look for the `Community` attribute in the output. You should see `20473:200` on routes that ISP2 learned from their upstream.


### Task 3.2: Configure Community-Based Import Policy

Create a community list to match the IXP peer community, then add a new sequence to the existing import route-map to set a higher local preference for matching routes.

In [Lab 2, Task 3](/routing-lab/labs/lab-2#task-3-configure-route-filtering), you configured the import route-map `RM_IMPORT_IN6` on bgp2 which matches `PL_IMPORT_GT48` to filter incoming routes. You need to add a new sequence before the existing one so that the local preference is set for IXP peer routes before the import filter processes them.

On bgp2:

```
configure terminal

bgp community-list standard CL-ISP2-IXP permit 20473:200

route-map RM_IMPORT_IN6 permit 5
match community CL-ISP2-IXP
set local-preference 120
exit

end
```

By inserting sequence number 5 before the existing sequence 10, routes tagged with community `20473:200` get local preference 120 before the general import filter applies. These routes still pass through the import filter (since IXP peer prefixes are within the `/0 le 48` range allowed by `PL_IMPORT_GT48`). All other routes continue to be processed by sequence 10 with the default local preference of 100. Since local preference is propagated via iBGP, bgp1 will also prefer ISP2 for IXP peer destinations.


### Task 3.3: Verify Community-Based Policy

Verify that routes with community `20473:200` now have local preference 120:

```
show ip bgp ipv6 community-list CL-ISP2-IXP
```

Check a specific IXP peer prefix:

```
show ip bgp ipv6 2001:4860:4860::8888
show ip bgp ipv6 2606:4700:4700::1111
```

The route via ISP2 should show `localpref: 120`, making it preferred in the local AS (at default `LocPrf: 100`).

### Task 3.4: Verify Outbound Traffic Path

Test outbound traffic to an IXP peer destination from the distribution switches:

```
traceroute ipv6 2001:4860:4860::8888
```

Traffic to IXP peer destinations should now exit via bgp2/ISP2.

---

## Open Question: Explore and Optimize

Now that you have learned three traffic engineering techniques (AS-prepend, local preference, and community-based policies), it's time to explore the real internet routing table and think critically about your network's performance.

### Task: Find an Interesting IPv6 Destination

1. Pick an IPv6 address or prefix that you find interesting — it could be a service you use regularly, a university network, or any prefix you can find in the BGP table.

2. Check the current routing for that prefix:

```
show ip bgp ipv6 <chosen-prefix>
```

3. Analyze the output:
   - How many paths are available?
   - Which ISP path is currently preferred? Why?
   - What is the AS path through each ISP?
   - Are there any communities attached to the route?

4. Think about whether the current path is optimal:
   - Is the latency acceptable? You can check with `traceroute ipv6 <address>`.
   - Is the traffic going through an unexpectedly long AS path?
   - Would a different ISP path be better for cost, latency, or capacity reasons?

5. If you think the current path is not optimal, consider:
   - Which traffic engineering technique would you use to improve it?
   - Would you use AS-prepend, local preference, or a community-based policy?
   - What are the potential side effects of your change?

There is no single correct answer — the goal is to practice analyzing real routing data and making informed decisions. Discuss your findings and proposed changes with your instructor or classmates.

---

## Extra Reading: BGP Communities in Depth

### Well-Known Communities (RFC 1997)

BGP defines several well-known community values that have standardized behavior:

| Community | Value | Purpose |
|-----------|-------|---------|
| `NO_EXPORT` | 65535:65281 | Route should not be advertised outside the local AS or local BGP confederation |
| `NO_ADVERTISE` | 65535:65282 | Route should not be advertised to any BGP peer |
| `NO_EXPORT_SUBCONFED` | 65535:65283 | Route should not be advertised outside the local AS |
| `Remote-Triggered Black Hole (RTBH)` | 65535:666 | Route should be black-holed (dropped) by the receiving AS. Providers that support this policy will null-route the traffic at their edge |

<blockquote class="tip">
When you attach `NO_EXPORT` to a route, the receiving AS will not advertise it to its eBGP peers. This is useful when you want your prefix to be visible only within the ISP's AS, for example when testing a new prefix before full deployment.
</blockquote>

### Communities That Encode Routing Information (RFC 8195)

RFC 8195 defines a convention for using BGP communities to encode routing information. These communities allow you to tag routes with structured data that other routers can use for policy decisions. The format is `<ASN>:<value>` where the value encodes specific information.

Common RFC 8195 community patterns include:

| Community Pattern | Example | Encoded Information |
|-------------------|---------|---------------------|
| `<ASN>:<region-code>` | `64500:100` | Route originates from region 100 (e.g., EU) |
| `<ASN>:<peer-type>` | `64500:2100` | Route learned from a transit provider |
| `<ASN>:<link-speed>` | `64500:31000` | Link speed in Mbps (useful for capacity-aware routing) |
| `<origin-ASN>:0` | `64500:0` | Route originated by AS 64500 |

ISP2's community `20473:200` follows this convention in a similar way — `20473` is ISP2's ASN and `200` indicates the route was learned via an IXP peering session.

**Example: Using communities to encode geographic origin**

Suppose your AS (64500) has two data centers — one in Europe and one in Asia. You can tag routes advertised from each data center with a community indicating the geographic region:

```
! Routes from the EU data center
set community 64500:100    ! 100 = Europe region

! Routes from the Asia data center
set community 64500:200    ! 200 = Asia-Pacific region
```

A peer receiving these routes can then apply policies based on the community value — for example, preferring the EU-origin route for European traffic and the Asia-origin route for Asia-Pacific traffic.

**Example: Using communities to signal route source type**

You can also tag routes based on where they were learned:

```
! Route learned from a transit provider
set community 64500:2100   ! 2100 = transit

! Route learned from a peering partner
set community 64500:2200   ! 2200 = peering

! Route learned from a customer
set community 64500:2300   ! 2300 = customer
```

This allows routers within your AS to apply different policies based on the route source — for example, preferring customer routes over peering routes over transit routes. 

Apart from those well-known communities, every AS might have their own BGP community list, which can be used to filter inbound routes or apply actions to upstream. Here is an example [ISP2's community list](https://docs.vultr.com/products/network/bgp/asn-information/as20473-bgp-communities-customer-guide).

---

## Verification Checklist

- [ ] AS-prepend configured on bgp1 to make ISP1 path slightly less desirable for inbound traffic
- [ ] AS-prepend verified on looking glass — your AS appears extra times in the ISP1 path
- [ ] Local preference set to 150 for `2001:df6:bf40::/48` via ISP1 on bgp1
- [ ] Local preference propagated via iBGP — both routers prefer ISP1 for the dev team's destination
- [ ] Community list matching `20473:200` configured on bgp2
- [ ] Routes with community `20473:200` have local preference 120, preferring ISP2
- [ ] Outbound traffic to IXP peer destinations exits via bgp2/ISP2
- [ ] Existing route filtering from Lab 2 still functional
- [ ] All configurations saved

## Common Issues

| Issue | Solution |
|-------|----------|
| AS-prepend not visible on looking glass | Clear BGP session; wait for propagation; verify route-map is applied |
| Local preference not affecting path | Verify route-map is applied inbound; clear BGP to re-receive routes; check iBGP is propagating the value |
| Community-based policy not matching | Verify community list matches the exact community value; check route-map is applied inbound |
| Traffic still using wrong path | Check all BGP attributes; verify no other policies override your changes; remember local preference overrides AS path |
| New route-map overwrites existing filtering | Merge prefix-list match conditions from your Lab 2 export filter into the new route-map |
| Changes not taking effect | Clear BGP sessions; verify configuration is saved |

## Troubleshooting Commands

| Command | Purpose |
|---------|---------|
| `show ip bgp ipv6` | View IPv6 BGP table with all attributes |
| `show ip bgp ipv6 <prefix>` | View detailed IPv6 BGP information for a prefix |
| `show ip bgp ipv6 neighbors <ip> advertised-routes` | View routes sent to neighbor |
| `show ip bgp ipv6 community-list <name>` | View routes matching a community list |
| `show route-map` | View configured route maps |
| `show ipv6 prefix-list` | View configured IPv6 prefix lists |
| `clear bgp ipv6 <ip> in` | Re-receive IPv6 routes from neighbor |
| `clear bgp ipv6 <ip> out` | Re-advertise IPv6 routes to neighbor |
| `debug bgp updates` | Debug BGP updates (use carefully) |

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

In this lab, you learned essential BGP traffic engineering techniques through realistic scenarios:

- **AS-prepend**: Influences inbound traffic by making paths appear less desirable (affects AS Path length). You used it to shift inbound traffic away from ISP1 when its upstream was experiencing issues.
- **Local preference**: Controls outbound traffic path selection within your AS (uses local preference). You used it to route the dev team's traffic to a specific destination via ISP1's higher-capacity uplink.
- **BGP communities**: Carry routing information that you can use for policy decisions. You matched ISP2's IXP peer community (`20473:200`) to give those routes a higher local preference, taking advantage of ISP2's discounted pricing for IXP traffic.

These techniques allow you to optimize traffic flow in multi-homed environments for performance, cost, and capacity reasons. In production networks, always coordinate with your ISP when implementing traffic engineering policies.
