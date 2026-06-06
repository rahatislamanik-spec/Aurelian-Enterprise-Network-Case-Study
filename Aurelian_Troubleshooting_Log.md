# Aurelian Financial Group — Network Lab
## Full Troubleshooting & Build Log
### Session Date: May 16, 2026

---

## Project Overview

**Objective:** Build and validate a segmented enterprise network for a fictional financial firm (Aurelian Financial Group) using Cisco Packet Tracer, demonstrating Tier 2/3 networking skills including VLAN segmentation, inter-VLAN routing, DHCP relay, and STP troubleshooting.

**Topology:**
- 1x Cisco 2911 Router → `Aurelian-RTR-01`
- 1x Cisco 3560-24PS Core Switch → `Aurelian-SW-CORE`
- 4x Cisco 2960-24TT Access Switches → `SW-FINANCE`, `SW-HR`, `SW-IT`, `SW-MGMT`
- 8x PCs → `PC-FIN-01/02`, `PC-HR-01/02`, `PC-IT-01/02`, `PC-MGMT-01/02`
- 1x Server → `Aurelian-DHCP-SRV`

**VLAN Design:**

| VLAN | Name | Subnet | Gateway |
|---|---|---|---|
| 10 | FINANCE | 192.168.10.0/24 | 192.168.10.1 |
| 20 | HR | 192.168.20.0/24 | 192.168.20.1 |
| 30 | IT | 192.168.30.0/24 | 192.168.30.1 |
| 40 | MGMT | 192.168.40.0/24 | 192.168.40.1 |

---

## Pre-Session State (Carried Forward)

The following were already completed before this session:
- All 4 access switches: VLANs created, access ports assigned, uplink trunk ports configured
- SW-CORE: All 4 VLANs created, trunk ports set
- Router-on-a-Stick: All 4 subinterfaces (Gi0/0.10/.20/.30/.40) configured
- Scenario 1 completed: Native VLAN Mismatch on Fa0/2 between SW-CORE and SW-IT — fixed with `switchport trunk native vlan 1` on both sides
- DHCP Server: Service ON, FINANCE_POOL configured

---

## Session Work Log

---

### PHASE 1 — DHCP Pool Configuration

**Objective:** Add remaining DHCP pools for HR, IT, and MGMT VLANs.

**Action:** On `Aurelian-DHCP-SRV` → Services → DHCP, added three pools:

| Pool | Start IP | Gateway | DNS | Mask | Max Users |
|---|---|---|---|---|---|
| HR_POOL | 192.168.20.10 | 192.168.20.1 | 8.8.8.8 | 255.255.255.0 | 50 |
| IT_POOL | 192.168.30.10 | 192.168.30.1 | 8.8.8.8 | 255.255.255.0 | 50 |
| MGMT_POOL | 192.168.40.10 | 192.168.40.1 | 8.8.8.8 | 255.255.255.0 | 50 |

**Note:** A default `serverPool` existed on the server pointing to `192.168.40.0/24` with no gateway. This was left temporarily but caused issues later (documented in Phase 6).

**Result:** All 4 DHCP pools confirmed visible in the server's pool table. ✅

---

### PHASE 2 — DHCP Server Static IP

**Issue Identified:** The DHCP server (`Aurelian-DHCP-SRV`) was connected to SW-MGMT (VLAN 40). For DHCP relay (ip helper-address) to function, the server needed a fixed IP so the router could forward DHCP Discovers to it.

**Action:** Set static IP on the DHCP server:
```
IP Address:      192.168.40.2
Subnet Mask:     255.255.255.0
Default Gateway: 192.168.40.1
DNS:             8.8.8.8
```

**Result:** Server interface FastEthernet0 confirmed UP with 192.168.40.2. ✅

---

### PHASE 3 — ip helper-address Configuration

**Objective:** Configure DHCP relay on the router so PCs in VLANs 10, 20, and 30 can reach the DHCP server in VLAN 40.

**Action:** On `Aurelian-RTR-01`:
```
conf t
interface GigabitEthernet0/0.10
 ip helper-address 192.168.40.2
interface GigabitEthernet0/0.20
 ip helper-address 192.168.40.2
interface GigabitEthernet0/0.30
 ip helper-address 192.168.40.2
end
write memory
```

**Note:** VLAN 40 (MGMT) does not need a helper because the DHCP server resides on that VLAN — PCs reach it directly via broadcast.

**Initial Result:** Router ping to 192.168.40.2 failed (0/5). Server not yet reachable.

---

### ISSUE 1 — Server Port Misconfigured on SW-MGMT

**Symptom:** Router could not ping 192.168.40.2. DHCP relay non-functional.

**Diagnosis:** Ran `show cdp neighbors` on SW-MGMT. Confirmed:
- Fa0/3 on SW-MGMT → connected to SW-CORE (Fa0/3 on the 3560)

**Root Cause:** When investigating which port the server was connected to, Fa0/3 was incorrectly set as an access port (VLAN 40) instead of remaining a trunk port. The command `switchport mode access` + `switchport access vlan 40` was applied to Fa0/3, which was actually the **trunk uplink to SW-CORE** — not the server port.

**Error Generated:**
```
%CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered on 
FastEthernet0/3 (40), with Switch FastEthernet0/3 (1).
%SPANTREE-2-RECV_PVID_ERR: Received 802.1Q BPDU on non trunk Fa0/3 VLAN1
%SPANTREE-2-BLOCK_PVID_LOCAL: Blocking FastEthernet0/3 on VLAN0001. 
Inconsistent port type.
```

**Fix Applied — SW-MGMT:**
```
conf t
interface fa0/3
 no switchport access vlan 40
 switchport mode trunk
 switchport trunk native vlan 1
end
write memory
```

**Result:** Fa0/3 bounced (down → up), CDP mismatch warnings stopped. ✅

---

### ISSUE 2 — SW-CORE FastEthernet Ports Not Configured as Trunks

**Symptom:** After fixing SW-MGMT's Fa0/3, new CDP native VLAN mismatch errors appeared on SW-CORE's Fa0/2 and Fa0/3. Router still couldn't reach 192.168.40.2.

**Diagnosis:** Ran `show interfaces trunk` on SW-CORE. Output revealed:
```
Only Gi0/1 and Gi0/2 were trunking.
Fa0/1, Fa0/2, Fa0/3 — NOT in trunk table.
```

**Root Cause:** The Cisco 3560 requires explicit `switchport trunk encapsulation dot1q` before `switchport mode trunk` can be applied. This step had been missed on all FastEthernet uplinks to access switches. Additionally, `switchport trunk native vlan 1` had been set on individual ports without setting the encapsulation or trunk mode first — meaning the ports were still operating as access ports.

**Error Generated:**
```
%SPANTREE-2-RECV_PVID_ERR: Received 802.1Q BPDU on non trunk Fa0/3 VLAN1
%SPANTREE-2-BLOCK_PVID_LOCAL: Blocking FastEthernet0/3 on VLAN0001. 
Inconsistent port type.
%CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch on Fa0/2 (1) with 
Switch Fa0/2 (30)
```

**Fix Applied — SW-CORE:**
```
conf t
interface fa0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1
interface fa0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1
interface fa0/3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1
end
write memory
```

**Result:** All 5 SW-CORE trunk ports confirmed active:
```
Fa0/1  trunking  native 1  VLANs 1,10,20,30,40 ✅
Fa0/2  trunking  native 1  VLANs 1,10,20,30,40 ✅
Fa0/3  trunking  native 1  VLANs 1,10,20,30,40 ✅
Gi0/1  trunking  native 1  VLANs 1,10,20,30,40 ✅
Gi0/2  trunking  native 1  VLANs 1,10,20,30,40 ✅
```
Topology went **all green**. ✅

---

### ISSUE 3 — SW-IT Fa0/2 Native VLAN Mismatch (Residual)

**Symptom:** After SW-CORE trunks were fixed, a persistent CDP mismatch remained:
```
%CDP-4-NATIVE_VLAN_MISMATCH: FastEthernet0/2 (1) with Switch Fa0/2 (30)
```

**Diagnosis:** SW-CORE's Fa0/2 was now native VLAN 1, but the connected switch (SW-IT) still had its uplink Fa0/2 with native VLAN 30. Additionally, SW-IT's Fa0/2 was not in trunk mode — same encapsulation issue as SW-CORE's FastEthernet ports.

**Fix Applied — SW-IT:**
```
conf t
interface fa0/2
 switchport mode trunk
 switchport trunk native vlan 1
end
write memory
```

**Result:** SW-IT Fa0/2 confirmed trunking with native VLAN 1. CDP mismatch cleared. ✅

---

### PHASE 4 — DHCP Testing

**Objective:** Set all 8 PCs to DHCP and verify correct pool assignment.

**First test:** PC-FIN-01 set to DHCP.

**Result:**
```
DHCP request successful.
IP: 192.168.10.10 / 255.255.255.0
Gateway: 192.168.10.1
DNS: 8.8.8.8
```
✅ FINANCE VLAN DHCP relay working.

---

### ISSUE 4 — PC-HR-01 DHCP Failed (Wrong Access Port VLAN Assignment)

**Symptom:** PC-HR-01 returned "DHCP request failed." Static ping test to gateway 192.168.20.1 also timed out (100% loss).

**Diagnosis — Layer by Layer:**

1. `show spanning-tree vlan 20` on SW-HR → All ports FWD ✅
2. `show interfaces fa0/1 switchport` on SW-CORE → Administrative + Operational mode trunk ✅
3. `show ip interface brief` on router → Gi0/0.20 up/up, IP 192.168.20.1 ✅
4. `show interfaces status` on SW-HR → Fa0/2 and Fa0/3 showing **notconnect** in VLAN 20

**Root Cause:** PC-HR-01 and PC-HR-02 were physically connected to **SW-HR's Gi0/1 and Fa0/24** respectively — not Fa0/2 and Fa0/3 as assumed. The VLAN 20 access port assignments had been made to the wrong ports. Gi0/1 was also incorrectly configured as a trunk port.

**Discovered via:** Hovering over cables in Packet Tracer canvas:
- PC-HR-01 → SW-HR **Gi0/1** (was trunk — no traffic)
- PC-HR-02 → SW-HR **Fa0/24** (was VLAN 1 — wrong VLAN)

**Fix Applied — SW-HR:**
```
conf t
interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 20
interface FastEthernet0/24
 switchport mode access
 switchport access vlan 20
end
write memory
```

**Result:** PC-HR-01 DHCP successful:
```
IP: 192.168.20.10 / 255.255.255.0
Gateway: 192.168.20.1
DNS: 8.8.8.8 ✅
```

---

### ISSUE 5 — Same Wrong-Port Pattern on SW-IT and SW-MGMT

**Diagnosis:** Same root cause identified across remaining switches. PCs were connected to GigabitEthernet uplink ports (Gi0/1, Gi0/2) which were incorrectly in trunk mode, rather than FastEthernet access ports.

**Discovered via canvas cable hover:**

| Switch | PC | Actual Port | Was Configured As |
|---|---|---|---|
| SW-IT | PC-IT-01 | Gi0/1 | Trunk (wrong) |
| SW-IT | PC-IT-02 | Gi0/2 | VLAN 1 (wrong) |
| SW-MGMT | PC-MGMT-01 | Gi0/1 | Trunk (wrong) |
| SW-MGMT | PC-MGMT-02 | Gi0/2 | VLAN 1 (wrong) |

**Fix Applied — SW-IT:**
```
conf t
interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 30
interface GigabitEthernet0/2
 switchport mode access
 switchport access vlan 30
end
write memory
```

**Fix Applied — SW-MGMT:**
```
conf t
interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 40
interface GigabitEthernet0/2
 switchport mode access
 switchport access vlan 40
end
write memory
```

**Results:**
```
PC-IT-01: 192.168.30.10  GW: 192.168.30.1 ✅
PC-IT-02: 192.168.30.11  GW: 192.168.30.1 ✅
```

---

### ISSUE 6 — PC-MGMT DHCP Serving Wrong Pool (serverPool Conflict)

**Symptom:** PC-MGMT-01 and PC-MGMT-02 received DHCP leases but with wrong IPs and no gateway:
```
PC-MGMT-01: 192.168.40.3   GW: 0.0.0.0 ✗
PC-MGMT-02: 192.168.40.4   GW: 0.0.0.0 ✗
```

**Root Cause:** A default `serverPool` existed on the DHCP server (created automatically by Packet Tracer) with:
- Start IP: 192.168.40.0
- No gateway (0.0.0.0)
- No DNS

Since MGMT PCs are on the same subnet as the server (VLAN 40) and reach it directly without relay, they received leases from `serverPool` (starting at .0) before `MGMT_POOL` (starting at .10) could serve them. The serverPool had no gateway configured, hence 0.0.0.0.

**Fix Attempted:** Clicked "Remove" on serverPool — button was **grayed out** (Packet Tracer does not allow deletion of the default serverPool).

**Workaround Applied:** Modified serverPool's Start IP to `10.0.0.1` (a non-existent subnet in the network) and saved. This neutralized the pool — no VLAN 40 DHCP requests would match the 10.0.0.x range.

**Result:** After toggling DHCP on MGMT PCs:
```
PC-MGMT-01: 192.168.40.11  GW: 192.168.40.1 ✅
PC-MGMT-02: 192.168.40.12  GW: 192.168.40.1 ✅
```

---

## Final DHCP Results Summary

| PC | IP Assigned | Gateway | Status |
|---|---|---|---|
| PC-FIN-01 | 192.168.10.10 | 192.168.10.1 | ✅ |
| PC-FIN-02 | 192.168.10.10 | 192.168.10.1 | ✅ |
| PC-HR-01 | 192.168.20.10 | 192.168.20.1 | ✅ |
| PC-IT-01 | 192.168.30.10 | 192.168.30.1 | ✅ |
| PC-IT-02 | 192.168.30.11 | 192.168.30.1 | ✅ |
| PC-MGMT-01 | 192.168.40.11 | 192.168.40.1 | ✅ |
| PC-MGMT-02 | 192.168.40.12 | 192.168.40.1 | ✅ |

---

## Errors Encountered — Master Reference

| # | Error | Device | Cause | Fix |
|---|---|---|---|---|
| 1 | CDP Native VLAN Mismatch (Fa0/3) | SW-MGMT | Trunk uplink set as access VLAN 40 | Restored trunk mode + native VLAN 1 |
| 2 | SPANTREE BLOCK_PVID_LOCAL (Fa0/3) | SW-MGMT | Port type inconsistency from access→trunk change | Trunk restore + port bounce |
| 3 | SW-CORE Fa0/1/2/3 not trunking | SW-CORE | 3560 requires `encapsulation dot1q` before trunk mode | Added encapsulation + trunk mode on all Fa ports |
| 4 | CDP Native VLAN Mismatch (Fa0/2, VLAN 30) | SW-IT | SW-IT uplink Fa0/2 had native VLAN 30, not in trunk mode | Set trunk mode + native VLAN 1 |
| 5 | DHCP APIPA on PC-HR-01 | PC-HR-01 | PC connected to Gi0/1 (trunk port), not VLAN 20 access port | Changed Gi0/1 to access VLAN 20 |
| 6 | DHCP APIPA on PC-IT/MGMT | Multiple PCs | PCs on Gi ports (trunk mode), not correct VLAN access ports | Changed Gi0/1, Gi0/2 to correct access VLANs |
| 7 | MGMT DHCP wrong pool / no gateway | PC-MGMT-01/02 | serverPool (192.168.40.0, no GW) overriding MGMT_POOL | Moved serverPool to 10.0.0.x subnet |
| 8 | Management access unprotected | Router and switches | Telnet allowed and VTY lines lacked access restrictions | Enabled SSH-only access and restricted management to VLAN 40 |

---

## Key Technical Lessons

**1. Cisco 3560 Trunk Encapsulation**
Unlike the 2960 (which auto-detects encapsulation), the 3560 requires explicit configuration:
```
switchport trunk encapsulation dot1q
switchport mode trunk
```
Skipping the encapsulation command leaves the port in access mode even if `switchport mode trunk` is attempted.

**2. CDP Native VLAN Mismatch Detection**
Cisco CDP actively detects when trunk ports on adjacent switches have mismatched native VLANs and generates:
```
%CDP-4-NATIVE_VLAN_MISMATCH
```
When detected alongside STP, the switch will block the port on the mismatched VLAN to prevent potential loops:
```
%SPANTREE-2-BLOCK_PVID_LOCAL: Blocking on VLAN0001. Inconsistent port type.
```

**3. DHCP Relay (ip helper-address)**
For DHCP to cross VLAN boundaries, each router subinterface serving a VLAN without a local DHCP server must be configured with:
```
ip helper-address <DHCP_server_IP>
```
The relay converts the broadcast DHCP Discover into a unicast packet forwarded to the server IP.

**4. Physical Port Verification**
In Packet Tracer, always verify actual cable connections by hovering over cables in the canvas view before assigning VLANs to ports. PCs in this lab were connected to GigabitEthernet uplink ports rather than expected FastEthernet access ports.

**5. DHCP Pool Conflicts**
When multiple DHCP pools exist for the same subnet, the pool with the lowest starting IP wins. The Packet Tracer default `serverPool` (starting at the subnet base address) will override intentional pools with higher start IPs if not neutralized.

---

## Scenario 1 — Previously Documented

**Title:** Native VLAN Mismatch — SW-CORE to SW-IT

**Break:** Fa0/2 between SW-CORE and SW-IT had native VLAN set to VLAN 30 (IT) on SW-IT side, while SW-CORE expected native VLAN 1.

**Symptom:** CDP NATIVE_VLAN_MISMATCH error, STP port blocking.

**Fix:**
```
! On SW-CORE Fa0/2 and SW-IT Fa0/2:
switchport trunk native vlan 1
```

---

## Follow-Up Opportunities

- Add exported final device configurations if needed for deeper review
- Add HSRP, EtherChannel, and OSPF as future advanced scenarios
- Add syslog/SNMP-style monitoring notes to extend the operations workflow

---

*Log compiled: May 16, 2026 — Aurelian Financial Group Network Lab*
*Engineer: Md Rahat Islam Anik*
