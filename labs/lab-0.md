---
layout: lab
title: "Lab 0: Environment Setup and FRR Access"
lab_number: 0
duration: "30 minutes"
objectives:
  - Connect to the lab server using SSH key authentication
  - Understand the lab topology and node management IPs
  - Access FRR routing daemon terminal on network nodes
prev:
  url: /
  title: Home
next:
  url: /labs/lab-1
  title: Lab 1 - IPv4 BGP Peering and Backup Path over OSPF
---

## Introduction

You are a network engineer tasked with setting up your enterprise's core network with a multi-homed BGP configuration. This architecture provides redundancy by connecting to two different ISP peers (via bgp1 and bgp2), ensuring your network remains accessible even if one ISP connection fails.

Before you can begin configuring BGP peering and routing policies, you need to verify that you can access all network devices and have received the necessary information from your ISP and the local internet registry.

## Prerequisites

Before starting, ensure you have received from your instructor:

- The server IP address
- Your username and password for BGP routers and distribution switches
- Your SSH private key file for server access
- Your assigned **AS Number (ASN)**
- Your allocated **IPv4 prefix**
- Your allocated **IPv6 prefix**

You will also need a terminal application (Linux/macOS Terminal, Windows PowerShell, PuTTY, etc.).

## Task 1: SSH Key Authentication

### Connecting to Your VM

Connect using the provided credentials:

```bash
ssh -i /path/to/your_key username@server_ip
```

Replace:
- `/path/to/your_key` - Path to the SSH private key file provided to you
- `username` - Your assigned username
- `server_ip` - The IP address of your VM

<blockquote class="tip">
If you get a "Permission denied" error on the key file, check that your key file has correct permissions: `chmod 600 /path/to/your_key`
</blockquote>

### Verifying Connection

Once connected, verify you are on the correct server:

```bash
hostname
whoami
```

## Task 2: Lab Topology and Node Access

### Lab Topology

The lab environment consists of multiple network nodes running FRR. Each node has a management IP address that you can use to connect directly.

![Management Network Topology](/assets/image/mgmt_topo.png)

### Node Information

| Node | Interface | Management IP | Description |
|------|------|---------------|-------------|
| bgp1 | Eth0 | 172.20.20.11 | BGP router 1 |
| bgp2 | Eth0 | 172.20.20.12 | BGP router 2 |
| ds1 | Eth0 | 172.20.20.13 | Distribution switch 1 |
| ds2 | Eth0 | 172.20.20.14 | Distribution switch 2 |

### Connecting to a Node

To access a specific node, SSH directly to its management IP:

```bash
ssh username@172.20.20.11
```

Replace `172.20.20.11` with the management IP of the node you want to access.

## Task 3: Accessing FRR Terminal

FRR (Free Range Routing) is a routing software suite used in the labs. You need to access its command-line interface to configure routing protocols. Don't worry, you can easily use your existing experience with Cisco CLI to configure FRR.

### Entering FRR CLI

Once connected to a node, enter the FRR terminal:

```bash
sudo vtysh
```

This opens the FRR command-line interface where you can configure routing.

### Basic Navigation

Enter configuration mode:

```
enable
configure terminal
```

Exit to previous mode:

```
exit
```

Exit configure mode:

```
end
```

### Exiting the Node

To exit FRR and return to the node shell:

```bash
hostname
exit
```

To exit the node and return to your VM:

```bash
hostname
exit
```

## Verification Checklist

- [ ] Successfully connected to VM via SSH
- [ ] Able to SSH to individual nodes using management IPs
- [ ] Able to access FRR terminal
- [ ] Understand basic FRR CLI navigation
- [ ] Confirmed receipt of ASN, IPv4 prefix, and IPv6 prefix

## Common Commands Reference

| Action | Command |
|--------|---------|
| SSH to VM | `ssh -i key_file user@host` |
| SSH to node | `ssh user@node_mgmt_ip` |
| Enter FRR | `sudo vtysh` |
| Enable mode | `enable` |
| Config mode | `configure terminal` |
| Exit | `exit` |

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Permission denied (SSH) | Check key permissions: `chmod 600 key_file` |
| Cannot connect to server | Verify the server IP, username and password are correct, if still cannot access the server, contact your instructor |
| Cannot connect to node | Verify the management IP is correct |
| Cannot enter FRR | Verify FRR service is running on the node |

## Conclusion

You now have verified access to your enterprise core network devices and confirmed you have received the necessary BGP configuration parameters (ASN, IPv4 prefix, IPv6 prefix). In the upcoming labs, you will configure BGP peering with your ISP peers and implement routing policies for your multi-homed network.
