# Aurelian Financial Group — Network Addressing Table

> Reference document for the enterprise network redesign.
> All addresses are simulated in Cisco Packet Tracer 8.x.

---

## Device Inventory

| Device | Model | Role | Management IP |
|---|---|---|---|
| Aurelian-R1 | Cisco 2911 | Gateway Router / Inter-VLAN Routing | 192.168.40.254 |
| SW-CORE | Cisco 3560-24PS | Core Switch / Trunk Hub | 192.168.40.253 |
| SW-FINANCE | Cisco 2960-24TT | Access Switch — Finance | — |
| SW-HR | Cisco 2960-24TT | Access Switch — HR | — |
| SW-IT | Cisco 2960-24TT | Access Switch — IT | — |
| SW-MGMT | Cisco 2960-24TT | Access Switch — Management | — |
| DHCP-Server | Server-PT | Centralized DHCP Server | 192.168.40.2 |

---

## VLAN Addressing

| VLAN | Name | Subnet | Gateway | DHCP Pool | Scope |
|---|---|---|---|---|---|
| 10 | FINANCE | 192.168.10.0/24 | 192.168.10.1 | FINANCE_POOL | .10 – .200 |
| 20 | HR | 192.168.20.0/24 | 192.168.20.1 | HR_POOL | .10 – .200 |
| 30 | IT | 192.168.30.0/24 | 192.168.30.1 | IT_POOL | .10 – .200 |
| 40 | MANAGEMENT | 192.168.40.0/24 | 192.168.40.1 | MGMT_POOL | .10 – .200 |

---

## Router Subinterfaces (Aurelian-R1)

| Subinterface | VLAN | IP Address | DHCP Relay | ACL Applied |
|---|---|---|---|---|
| Gi0/0.10 | 10 — Finance | 192.168.10.1/24 | 192.168.40.2 | BLOCK_HR_TO_FINANCE (in) |
| Gi0/0.20 | 20 — HR | 192.168.20.1/24 | 192.168.40.2 | — |
| Gi0/0.30 | 30 — IT | 192.168.30.1/24 | 192.168.40.2 | — |
| Gi0/0.40 | 40 — Management | 192.168.40.1/24 | — | — |

---

## Trunk Links

| From | To | Encapsulation | Allowed VLANs | Native VLAN |
|---|---|---|---|---|
| Aurelian-R1 Gi0/0 | SW-CORE Fa0/1 | 802.1Q | 10, 20, 30, 40 | 1 |
| SW-CORE Fa0/2 | SW-FINANCE Fa0/1 | 802.1Q | 10 | 1 |
| SW-CORE Fa0/3 | SW-HR Fa0/1 | 802.1Q | 20 | 1 |
| SW-CORE Fa0/4 | SW-IT Fa0/1 | 802.1Q | 30 | 1 |
| SW-CORE Fa0/5 | SW-MGMT Fa0/1 | 802.1Q | 40 | 1 |

---

## ACL Policy

| ACL Name | Type | Rule | Applied To | Direction |
|---|---|---|---|---|
| BLOCK_HR_TO_FINANCE | Extended | Deny 192.168.20.0/24 → 192.168.10.0/24 | Gi0/0.10 | Inbound |
| BLOCK_HR_TO_FINANCE | Extended | Permit ip any any | Gi0/0.10 | Inbound |
| MGMT_ACCESS | Standard | Permit 192.168.40.0/24 | VTY 0-4 | Inbound |
| MGMT_ACCESS | Standard | Deny any | VTY 0-4 | Inbound |

---

## DHCP Server Pools (Centralized — 192.168.40.2)

| Pool Name | Network | Default Gateway | DNS |
|---|---|---|---|
| FINANCE_POOL | 192.168.10.0/24 | 192.168.10.1 | 8.8.8.8 |
| HR_POOL | 192.168.20.0/24 | 192.168.20.1 | 8.8.8.8 |
| IT_POOL | 192.168.30.0/24 | 192.168.30.1 | 8.8.8.8 |
| MGMT_POOL | 192.168.40.0/24 | 192.168.40.1 | 8.8.8.8 |

---

*Aurelian Financial Group — fictional enterprise network. All addresses simulated in Cisco Packet Tracer.*
