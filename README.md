# Cisco Network Lab (Hands-on Practice)

A hands-on lab built on real hardware that covers inter-VLAN routing (Router-on-a-Stick), NAT overload, DHCP, and ACL-based access control between departments.

**Designed and implemented by Chanathip Artwichian**

![Network diagram](01-diagrams/01-network-diagram.png)

---

## Table of Contents

1. [Objectives](#objectives)
2. [Equipment](#equipment)
3. [Topology](#topology)
4. [Addressing Plan](#addressing-plan)
5. [Configuration Summary](#configuration-summary)
6. [Verification and Test Results](#verification-and-test-results)
7. [Repository Structure](#repository-structure)
8. [Key Takeaways](#key-takeaways)

---

## Objectives

- Segment a small office network into three VLANs (IT, HR, BD)
- Route between VLANs using a router with 802.1Q sub-interfaces (Router-on-a-Stick)
- Provide DHCP services for each VLAN from the router
- Give all VLANs internet access through NAT overload (PAT)
- Enforce a department access policy with extended ACLs

## Equipment

| Role | Device | Notes |
|---|---|---|
| Internet gateway | TP-Link Archer MR200 | WAN side of the lab |
| Router (Layer 3) | Cisco 1841 | Fa0/0 = WAN, Fa0/1 = trunk with sub-interfaces |
| Switch (Layer 2) | Cisco Catalyst 2960 | Gi0/24 = trunk, Gi0/1-3 = access ports |
| Clients | Lenovo ThinkCentre x 3 | One per VLAN |

## Addressing Plan

### WAN

| Device | Interface | Address |
|---|---|---|
| TP-Link Archer MR200 | WAN1 | 203.0.113.1/30 |
| Cisco 1841 | Fa0/0 | 203.0.113.2/30 |

### VLANs

| VLAN | Name | Subnet | Default Gateway (Router sub-interface) | DHCP Pool | Switch Port |
|---|---|---|---|---|---|
| 10 | IT | 192.168.10.0/24 | 192.168.10.1 | 192.168.10.101 - .200 | Gi0/1 |
| 20 | HR | 192.168.20.0/24 | 192.168.20.1 | 192.168.20.101 - .200 | Gi0/2 |
| 30 | BD | 192.168.30.0/24 | 192.168.30.1 | 192.168.30.101 - .200 | Gi0/3 |

### Trunk

| Device | Port | Mode | Allowed VLANs |
|---|---|---|---|
| Catalyst 2960 | Gi0/24 | 802.1Q trunk | 10, 20, 30 |
| Cisco 1841 | Fa0/1 | 802.1Q sub-interfaces | 10, 20, 30 |

## Configuration Summary

Full device configurations are in the [`03-configs/`](03-configs/) folder (sensitive values have been redacted).

| Feature | Implementation |
|---|---|
| VLANs | VLAN 10 (IT), VLAN 20 (HR), VLAN 30 (BD) on the 2960 |
| Inter-VLAN routing | Router-on-a-Stick: `Fa0/1.10`, `Fa0/1.20`, `Fa0/1.30` using `encapsulation dot1Q` |
| DHCP | One pool per VLAN on the router (range .101 - .200) |
| NAT | NAT overload (PAT) for 192.168.10.0/24, 192.168.20.0/24, 192.168.30.0/24 out through Fa0/0 |
| Default route | Static default route to 203.0.113.1 |
| ACL | Department access policy (see below) |

### Access Policy (ACL)

| Source | Destination | Result |
|---|---|---|
| VLAN 10 (IT) | VLAN 20 (HR), VLAN 30 (BD) | Permit |
| VLAN 20 (HR) | VLAN 10 (IT) | Deny |
| VLAN 30 (BD) | VLAN 10 (IT) | Deny |
| VLAN 20 (HR) | VLAN 30 (BD) | Permit |
| VLAN 30 (BD) | VLAN 20 (HR) | Permit |
| All VLANs | Internet | Permit (via NAT) |

In short, IT can reach every department, HR and BD can communicate with each other, and neither HR nor BD can initiate connections to IT.

## Verification and Test Results

### Switch (Catalyst 2960)

**VLANs and port assignments** (`show vlan brief`)

![show vlan brief](02-screenshots/01-switch-vlan-brief.png)

**Trunk status** (`show interfaces trunk`)

![show interfaces trunk](02-screenshots/02-switch-interfaces-trunk.png)

### Router (Cisco 1841)

**Interface status** (`show ip interface brief`)

![show ip interface brief](02-screenshots/03-router-ip-interface-brief.png)

**Routing table** (`show ip route`)

![show ip route](02-screenshots/04-router-ip-route.png)

**DHCP pools** (`show ip dhcp pool`)

![show ip dhcp pool](02-screenshots/05-router-dhcp-pool.png)

**DHCP leases** (`show ip dhcp binding`)

![show ip dhcp binding](02-screenshots/06-router-dhcp-binding.png)

### Clients

| VLAN | IP configuration |
|---|---|
| VLAN 10 (IT) | ![ipconfig VLAN10](02-screenshots/08-pc-vlan10-ipconfig.png) |
| VLAN 20 (HR) | ![ipconfig VLAN20](02-screenshots/09-pc-vlan20-ipconfig.png) |
| VLAN 30 (BD) | ![ipconfig VLAN30](02-screenshots/10-pc-vlan30-ipconfig.png) |

### Internet Access and NAT

**Internet connectivity from each VLAN** (`ping 8.8.8.8` and `ping google.com`)

| VLAN | Result |
|---|---|
| VLAN 10 (IT) | ![Internet ping VLAN10](02-screenshots/11-pc-vlan10-internet-ping.png) |
| VLAN 20 (HR) | ![Internet ping VLAN20](02-screenshots/12-pc-vlan20-internet-ping.png) |
| VLAN 30 (BD) | ![Internet ping VLAN30](02-screenshots/13-pc-vlan30-internet-ping.png) |

**NAT translations** (`show ip nat translations`)

![NAT translations](02-screenshots/07-router-nat-translations.png)

### ACL Tests

| Test | Expected | Result |
|---|---|---|
| VLAN 10 to VLAN 20 / VLAN 30 | Success | ![](02-screenshots/14-acl-vlan10-to-vlan20-30-ok.png) |
| VLAN 20 to VLAN 30 | Success | ![](02-screenshots/15-acl-vlan20-to-vlan30-ok.png) |
| VLAN 20 to VLAN 10 | Blocked | ![](02-screenshots/16-acl-vlan20-to-vlan10-blocked.png) |
| VLAN 30 to VLAN 10 | Blocked | ![](02-screenshots/17-acl-vlan30-to-vlan10-blocked.png) |

**ACL hit counters** (`show access-lists`)

![show access-lists](02-screenshots/18-router-access-lists.png)

## Repository Structure

```
cisco-network-lab/
├── README.md
├── 01-diagrams/        # Network diagram
├── 02-screenshots/     # Command output and test results
└── 03-configs/         # Device configurations (sensitive data redacted)
```


## Author

**Chanathip Artwichian**
GitHub: [@chanathip-awc](https://github.com/chanathip-awc)
