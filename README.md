# SR Linux EVPN with Symmetric IRB (Interface-Less Pure IP-VRF / Type-5 Host Routes)

> [!NOTE]
> **EVPN Symmetric IRB Architectural Variants in this Series:**
> - **Method 1 (Interface-Less with Dual-Label Type-2, RFC 9135):** [srl-evpn-ifl-irb](https://github.com/andywhitaker/srl-evpn-ifl-irb)
> - **Method 2 (This Lab - Interface-Less with Type-5 Host Routes, RFC 9136 §4.3):** [srl-evpn-type5-irb](https://github.com/andywhitaker/srl-evpn-type5-irb)
> - **Method 3 (Interface-Ful with SBD, RFC 9136 §4.4):** [srl-evpn-iff-irb](https://github.com/andywhitaker/srl-evpn-iff-irb)

## Topology
![topology](lab-topology.png)

## Lab Description
This lab demonstrates **Nokia SR Linux EVPN using Symmetric IRB with Interface-Less (IFL) Pure IP-VRF Routing**, standardized under **RFC 9136 Section 4.3**.

In this architecture:
- **Clean L2/L3 Decoupling:** Client MAC-VRFs (`app`, `web`) handle pure Layer 2 switching and advertise EVPN **Type-2 MAC routes carrying only their single L2 VNI** (VNI 10010 or 10020). There is no dual-label encapsulation on Type-2 routes.
- **Dynamic ARP Host Population:** Tenant IRB subinterfaces (`irb0.1`, `irb0.2`) use `ipv4 arp host-route populate dynamic` to dynamically program learned `/32` ARP entries into the tenant IP-VRF routing table.
- **Type-5 IP Prefix Host Routing:** The tenant IP-VRF (`tenant1`) directly originates **EVPN Type-5 IP Prefix routes** for both subnet prefixes (`/24`) and dynamic host routes (`/32`) across the fabric using **L3 VNI 10000** and the router's Gateway MAC.
- **Direct IP-VRF Tunnel Termination:** The L3 VNI terminates directly in `network-instance tenant1 type ip-vrf` via `vxlan0.100 (type routed)`. No transit bridge domain (Supplementary Broadcast Domain / SBD) or unnumbered IRB interface is required.
- **Industry Standard for Multi-Vendor Fabrics:** This pure IP-VRF Type-5 model is the standard implementation in Cisco NX-OS, Arista EOS, and Linux FRR, ensuring seamless multi-vendor interoperability.

---

### Architectural Comparison: The Three Symmetric IRB Models

| Architectural Dimension | Method 1: IFL Dual-Label (RFC 9135) | Method 2: IFL Type-5 Host Routes (RFC 9136 §4.3) | Method 3: IFF with SBD (RFC 9136 §4.4) |
| :--- | :--- | :--- | :--- |
| **Standard / Reference** | RFC 9135 | RFC 9136 Section 4.3 | RFC 9136 Section 4.4 |
| **L3 VNI Network Instance** | `tenant1 (type ip-vrf)` | `tenant1 (type ip-vrf)` | `sbd (type mac-vrf)` |
| **VXLAN Interface Type** | `vxlan0.100 (type routed)` | `vxlan0.100 (type routed)` | `vxlan0.100 (type bridged)` |
| **Client MAC-VRF Type-2 Labels** | **Dual Labels:** `10010 + 10000` | **Single Label:** `10010` only | **Single Label:** `10010` only |
| **Host Route Carrier** | EVPN Type-2 MAC-IP | **EVPN Type-5 IP Prefix (/32)** | **EVPN Type-5 IP Prefix (/32)** via SBD |
| **Tenant Routing Table Type** | `bgp-evpn-ifl-host` | `bgp-evpn` | `bgp-evpn-iff` |
| **Next-Hop Resolution** | Direct to Remote VTEP Tunnel | Direct to Remote VTEP Tunnel | Two-Stage via SBD Bridge Table |
| **Inner Wire Payload** | Raw IPv4 / Direct L3 Payload | Raw IPv4 / Direct L3 Payload | Full Ethernet Frame (DMAC=Router MAC) |
| **Multicast Support (OISM)** | Unsupported | Unsupported | Mandatory for RFC 9251 OISM |
| **Target Use-Case** | Pure Nokia / Lowest BGP Prefix Count | **Multi-Vendor / Hyperscale IP-VRFs** | OISM Multicast & Legacy ASICs |

---

## Configuration Highlights

### 1. Client IRB Interface: Dynamic Host Route Population
Unlike Method 1 (which used `interface-less-routing`), Method 2 uses standard `host-route populate dynamic`:

```bash
set / interface irb0 subinterface 1 ipv4 admin-state enable
set / interface irb0 subinterface 1 ipv4 address 192.168.10.254/24 anycast-gw true
set / interface irb0 subinterface 1 ipv4 arp learn-unsolicited true
set / interface irb0 subinterface 1 ipv4 arp host-route populate dynamic
set / interface irb0 subinterface 1 anycast-gw virtual-router-id 1
```

### 2. Tenant IP-VRF with Routed VXLAN Interface
The L3 VNI is configured directly inside `tenant1`:

```bash
set / network-instance tenant1 type ip-vrf
set / network-instance tenant1 admin-state enable
set / network-instance tenant1 interface irb0.1
set / network-instance tenant1 interface irb0.2
set / network-instance tenant1 vxlan-interface vxlan0.100
set / network-instance tenant1 protocols bgp-evpn bgp-instance 1 vxlan-interface vxlan0.100
set / network-instance tenant1 protocols bgp-evpn bgp-instance 1 evi 10000
set / network-instance tenant1 protocols bgp-evpn bgp-instance 1 ecmp 8
set / network-instance tenant1 protocols bgp-evpn bgp-instance 1 routes route-table mac-ip advertise-gateway-mac true
set / network-instance tenant1 protocols bgp-vpn bgp-instance 1 route-target export-rt target:10000:10000
set / network-instance tenant1 protocols bgp-vpn bgp-instance 1 route-target import-rt target:10000:10000
```

---

## Containerlab Deployment

```bash
# Clone the repository
git clone https://github.com/andywhitaker/srl-evpn-type5-irb.git
cd srl-evpn-type5-irb

# Deploy the fabric
sudo clab deploy -t srl-evpn-type5-irb.clab.yaml
```

---

## Verification & Validation

### 1. End-to-End Connectivity (All-to-All Ping Matrix)
Testing all 56 host-to-host combinations across different subnets (`192.168.10.0/24` <-> `192.168.20.0/24`) and different leaf switches:

```text
Test Results: 56/56 passed (100% reachability)
```

### 2. Underlay Routing Table
```text
A:admin@leaf1# show network-instance default ipv4 route
========================================================================================================================================================================================================
IPv4-unicast route table for default network-instance
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Flags: > (best), * (unviable), ! (failed)
     : L (leaked route from another network-instance)
     : B (backup NHG active and displayed)
     : S (statistics supported)
     : D (dynamic LB), R (resilient LB)
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Prefix               Route Type   Metric   Pref    Flags    Next-Hop(s)
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
2.2.2.2/32           bgp          0        170     >        10.1.10.10(route:local)                                                                                                                 
                                                            10.1.20.20(route:local)
3.3.3.3/32           bgp          0        170     >        10.1.10.10(route:local)                                                                                                                 
                                                            10.1.20.20(route:local)
4.4.4.4/32           bgp          0        170     >        10.1.10.10(route:local)                                                                                                                 
                                                            10.1.20.20(route:local)
10.1.10.0/24         local        0        0       >        10.1.10.1(ethernet-1/1.0)
10.1.20.0/24         local        0        0       >        10.1.20.1(ethernet-1/2.0)
10.10.10.10/32       bgp          0        170     >        10.1.10.10(route:local)
20.20.20.20/32       bgp          0        170     >        10.1.20.20(route:local)
```

### 3. BGP Neighbors
```text
A:admin@leaf1# show network-instance protocols bgp neighbor
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
BGP neighbor summary for network-instance "default"
Flags: S static, D dynamic, L discovered by LLDP, B BFD enabled, - disabled, * slow
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+----------------------+--------------------------------+----------------------+--------+------------+------------------+------------------+----------------+--------------------------------+
|       Net-Inst       |              Peer              |        Group         | Flags  |  Peer-AS   |      State       |      Uptime      |    AFI/SAFI    |         [Rx/Active/Tx]         |
+======================+================================+======================+========+============+==================+==================+================+================================+
| default              | 10.1.10.10                     | ebgp-evpn            | S      | 100        | established      | 0d:0h:8m:58s     | evpn           | [33/30/11]                     |
|                      |                                |                      |        |            |                  |                  | ipv4-unicast   | [4/4/2]                        |
| default              | 10.1.20.20                     | ebgp-evpn            | S      | 100        | established      | 0d:0h:8m:58s     | evpn           | [33/0/44]                      |
|                      |                                |                      |        |            |                  |                  | ipv4-unicast   | [4/4/5]                        |
+----------------------+--------------------------------+----------------------+--------+------------+------------------+------------------+----------------+--------------------------------+
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Summary:
2 configured neighbors, 2 configured sessions are established, 0 disabled peers
0 dynamic peers
```

### 4. EVPN Route Type-2 (Single Label Only)
Notice that Type-2 routes carry **only a single label** (`10010` for app, `10020` for web). The dual label stack (`10010 + 10000`) found in Method 1 is absent:

```text
A:admin@leaf1# show network-instance protocols bgp routes evpn route-type 2 summary
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Show report for the BGP route table of network-instance "*"
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Status codes: u=used, *=valid, >=best, x=stale, b=backup, w=unused-weight-only
Origin codes: i=IGP, e=EGP, ?=incomplete
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
BGP Router ID: 1.1.1.1      AS: 1      Local AS: 1
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Type 2 MAC-IP Advertisement Routes
+-------+------------------+-----------+------------------+------------------+------------------+-------+------------------+------------------+-------------------------------+------------------+
| Statu |      Route-      |  Tag-ID   |   MAC-address    |    IP-address    |     neighbor     | Path- |     Next-Hop     |      Label       |              ESI              |   MAC Mobility   |
|   s   |  distinguisher   |           |                  |                  |                  |  id   |                  |                  |                               |                  |
+=======+==================+===========+==================+==================+==================+=======+==================+==================+===============================+==================+
| *>    | 2.2.2.2:10000    | 0         | 1A:8F:05:FF:00:0 | 0.0.0.0          | 10.1.10.10       | 0     | 2.2.2.2          | 10000            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | 0                |                  |                  |       |                  |                  |                               |                  |
| *     | 2.2.2.2:10000    | 0         | 1A:8F:05:FF:00:0 | 0.0.0.0          | 10.1.20.20       | 0     | 2.2.2.2          | 10000            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | 0                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 2.2.2.2:10010    | 0         | 00:00:5E:00:01:0 | 192.168.10.254   | 10.1.10.10       | 0     | 2.2.2.2          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 2.2.2.2:10010    | 0         | 00:00:5E:00:01:0 | 192.168.10.254   | 10.1.20.20       | 0     | 2.2.2.2          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 2.2.2.2:10010    | 0         | 1A:8F:05:FF:00:4 | 0.0.0.0          | 10.1.10.10       | 0     | 2.2.2.2          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 2.2.2.2:10010    | 0         | 1A:8F:05:FF:00:4 | 0.0.0.0          | 10.1.20.20       | 0     | 2.2.2.2          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 2.2.2.2:10020    | 0         | 00:00:5E:00:01:0 | 192.168.20.254   | 10.1.10.10       | 0     | 2.2.2.2          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 2.2.2.2:10020    | 0         | 00:00:5E:00:01:0 | 192.168.20.254   | 10.1.20.20       | 0     | 2.2.2.2          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 2.2.2.2:10020    | 0         | 1A:8F:05:FF:00:4 | 0.0.0.0          | 10.1.10.10       | 0     | 2.2.2.2          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 2.2.2.2:10020    | 0         | 1A:8F:05:FF:00:4 | 0.0.0.0          | 10.1.20.20       | 0     | 2.2.2.2          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *>    | 3.3.3.3:10000    | 0         | 1A:38:06:FF:00:0 | 0.0.0.0          | 10.1.10.10       | 0     | 3.3.3.3          | 10000            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | 0                |                  |                  |       |                  |                  |                               |                  |
| *     | 3.3.3.3:10000    | 0         | 1A:38:06:FF:00:0 | 0.0.0.0          | 10.1.20.20       | 0     | 3.3.3.3          | 10000            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | 0                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 3.3.3.3:10010    | 0         | 00:00:5E:00:01:0 | 192.168.10.254   | 10.1.10.10       | 0     | 3.3.3.3          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 3.3.3.3:10010    | 0         | 00:00:5E:00:01:0 | 192.168.10.254   | 10.1.20.20       | 0     | 3.3.3.3          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 3.3.3.3:10010    | 0         | 1A:38:06:FF:00:4 | 0.0.0.0          | 10.1.10.10       | 0     | 3.3.3.3          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 3.3.3.3:10010    | 0         | 1A:38:06:FF:00:4 | 0.0.0.0          | 10.1.20.20       | 0     | 3.3.3.3          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 3.3.3.3:10020    | 0         | 00:00:5E:00:01:0 | 192.168.20.254   | 10.1.10.10       | 0     | 3.3.3.3          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 3.3.3.3:10020    | 0         | 00:00:5E:00:01:0 | 192.168.20.254   | 10.1.20.20       | 0     | 3.3.3.3          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 3.3.3.3:10020    | 0         | 1A:38:06:FF:00:4 | 0.0.0.0          | 10.1.10.10       | 0     | 3.3.3.3          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 3.3.3.3:10020    | 0         | 1A:38:06:FF:00:4 | 0.0.0.0          | 10.1.20.20       | 0     | 3.3.3.3          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *>    | 4.4.4.4:10000    | 0         | 1A:D6:07:FF:00:0 | 0.0.0.0          | 10.1.10.10       | 0     | 4.4.4.4          | 10000            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | 0                |                  |                  |       |                  |                  |                               |                  |
| *     | 4.4.4.4:10000    | 0         | 1A:D6:07:FF:00:0 | 0.0.0.0          | 10.1.20.20       | 0     | 4.4.4.4          | 10000            | 00:00:00:00:00:00:00:00:00:00 | -                |
|       |                  |           | 0                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 4.4.4.4:10010    | 0         | 00:00:5E:00:01:0 | 192.168.10.254   | 10.1.10.10       | 0     | 4.4.4.4          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 4.4.4.4:10010    | 0         | 00:00:5E:00:01:0 | 192.168.10.254   | 10.1.20.20       | 0     | 4.4.4.4          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 4.4.4.4:10010    | 0         | 1A:D6:07:FF:00:4 | 0.0.0.0          | 10.1.10.10       | 0     | 4.4.4.4          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 4.4.4.4:10010    | 0         | 1A:D6:07:FF:00:4 | 0.0.0.0          | 10.1.20.20       | 0     | 4.4.4.4          | 10010            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 4.4.4.4:10020    | 0         | 00:00:5E:00:01:0 | 192.168.20.254   | 10.1.10.10       | 0     | 4.4.4.4          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 4.4.4.4:10020    | 0         | 00:00:5E:00:01:0 | 192.168.20.254   | 10.1.20.20       | 0     | 4.4.4.4          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| u*>   | 4.4.4.4:10020    | 0         | 1A:D6:07:FF:00:4 | 0.0.0.0          | 10.1.10.10       | 0     | 4.4.4.4          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
| *     | 4.4.4.4:10020    | 0         | 1A:D6:07:FF:00:4 | 0.0.0.0          | 10.1.20.20       | 0     | 4.4.4.4          | 10020            | 00:00:00:00:00:00:00:00:00:00 | Seq:0/Static     |
|       |                  |           | 1                |                  |                  |       |                  |                  |                               |                  |
+-------+------------------+-----------+------------------+------------------+------------------+-------+------------------+------------------+-------------------------------+------------------+
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
30 MAC-IP Advertisement routes 12 used, 30 valid, 0 stale
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

### 5. EVPN Route Type-5 (Pure IP-VRF Host Routes)
The IP-VRF directly originates both the `/24` subnet prefixes and the `/32` host routes across L3 VNI 10000:

```text
A:admin@leaf1# show network-instance protocols bgp routes evpn route-type 5 summary
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Show report for the BGP route table of network-instance "*"
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Status codes: u=used, *=valid, >=best, x=stale, b=backup, w=unused-weight-only
Origin codes: i=IGP, e=EGP, ?=incomplete
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
BGP Router ID: 1.1.1.1      AS: 1      Local AS: 1
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Type 5 IP Prefix Routes
+--------+----------------------------+------------+---------------------+----------------------------+--------+----------------------------+----------------------------+----------------------------+
| Status |    Route-distinguisher     |   Tag-ID   |     IP-address      |          neighbor          | Path-  |          Next-Hop          |           Label            |          Gateway           |
|        |                            |            |                     |                            |   id   |                            |                            |                            |
+========+============================+============+=====================+============================+========+============================+============================+============================+
| u*>    | 2.2.2.2:10000              | 0          | 192.168.10.0/24     | 10.1.10.10                 | 0      | 2.2.2.2                    | 10000                      | 0.0.0.0                    |
| *      | 2.2.2.2:10000              | 0          | 192.168.10.0/24     | 10.1.20.20                 | 0      | 2.2.2.2                    | 10000                      | 0.0.0.0                    |
| u*>    | 2.2.2.2:10000              | 0          | 192.168.20.0/24     | 10.1.10.10                 | 0      | 2.2.2.2                    | 10000                      | 0.0.0.0                    |
| *      | 2.2.2.2:10000              | 0          | 192.168.20.0/24     | 10.1.20.20                 | 0      | 2.2.2.2                    | 10000                      | 0.0.0.0                    |
| u*>    | 2.2.2.2:10000              | 0          | 192.168.10.2/32     | 10.1.10.10                 | 0      | 2.2.2.2                    | 10000                      | 0.0.0.0                    |
| *      | 2.2.2.2:10000              | 0          | 192.168.10.2/32     | 10.1.20.20                 | 0      | 2.2.2.2                    | 10000                      | 0.0.0.0                    |
| u*>    | 2.2.2.2:10000              | 0          | 192.168.20.2/32     | 10.1.10.10                 | 0      | 2.2.2.2                    | 10000                      | 0.0.0.0                    |
| *      | 2.2.2.2:10000              | 0          | 192.168.20.2/32     | 10.1.20.20                 | 0      | 2.2.2.2                    | 10000                      | 0.0.0.0                    |
| u*>    | 3.3.3.3:10000              | 0          | 192.168.10.0/24     | 10.1.10.10                 | 0      | 3.3.3.3                    | 10000                      | 0.0.0.0                    |
| *      | 3.3.3.3:10000              | 0          | 192.168.10.0/24     | 10.1.20.20                 | 0      | 3.3.3.3                    | 10000                      | 0.0.0.0                    |
| u*>    | 3.3.3.3:10000              | 0          | 192.168.20.0/24     | 10.1.10.10                 | 0      | 3.3.3.3                    | 10000                      | 0.0.0.0                    |
| *      | 3.3.3.3:10000              | 0          | 192.168.20.0/24     | 10.1.20.20                 | 0      | 3.3.3.3                    | 10000                      | 0.0.0.0                    |
| u*>    | 3.3.3.3:10000              | 0          | 192.168.10.3/32     | 10.1.10.10                 | 0      | 3.3.3.3                    | 10000                      | 0.0.0.0                    |
| *      | 3.3.3.3:10000              | 0          | 192.168.10.3/32     | 10.1.20.20                 | 0      | 3.3.3.3                    | 10000                      | 0.0.0.0                    |
| u*>    | 3.3.3.3:10000              | 0          | 192.168.20.3/32     | 10.1.10.10                 | 0      | 3.3.3.3                    | 10000                      | 0.0.0.0                    |
| *      | 3.3.3.3:10000              | 0          | 192.168.20.3/32     | 10.1.20.20                 | 0      | 3.3.3.3                    | 10000                      | 0.0.0.0                    |
| u*>    | 4.4.4.4:10000              | 0          | 192.168.10.0/24     | 10.1.10.10                 | 0      | 4.4.4.4                    | 10000                      | 0.0.0.0                    |
| *      | 4.4.4.4:10000              | 0          | 192.168.10.0/24     | 10.1.20.20                 | 0      | 4.4.4.4                    | 10000                      | 0.0.0.0                    |
| u*>    | 4.4.4.4:10000              | 0          | 192.168.20.0/24     | 10.1.10.10                 | 0      | 4.4.4.4                    | 10000                      | 0.0.0.0                    |
| *      | 4.4.4.4:10000              | 0          | 192.168.20.0/24     | 10.1.20.20                 | 0      | 4.4.4.4                    | 10000                      | 0.0.0.0                    |
| u*>    | 4.4.4.4:10000              | 0          | 192.168.10.4/32     | 10.1.10.10                 | 0      | 4.4.4.4                    | 10000                      | 0.0.0.0                    |
| *      | 4.4.4.4:10000              | 0          | 192.168.10.4/32     | 10.1.20.20                 | 0      | 4.4.4.4                    | 10000                      | 0.0.0.0                    |
| u*>    | 4.4.4.4:10000              | 0          | 192.168.20.4/32     | 10.1.10.10                 | 0      | 4.4.4.4                    | 10000                      | 0.0.0.0                    |
| *      | 4.4.4.4:10000              | 0          | 192.168.20.4/32     | 10.1.20.20                 | 0      | 4.4.4.4                    | 10000                      | 0.0.0.0                    |
+--------+----------------------------+------------+---------------------+----------------------------+--------+----------------------------+----------------------------+----------------------------+
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
24 IP Prefix routes 12 used, 24 valid, 0 stale
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

### 6. Tenant IP-VRF Route Table
Remote `/32` host routes are installed with route type **`bgp-evpn`**, resolving directly via the VXLAN tunnel to the remote VTEP on VNI 10000:

```text
A:admin@leaf1# show network-instance tenant1 ipv4 route
========================================================================================================================================================================================================
IPv4-unicast route table for ip-vrf network-instance: tenant1
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Flags: > (best), * (unviable), ! (failed)
     : L (leaked route from another network-instance)
     : B (backup NHG active and displayed)
     : S (statistics supported)
     : D (dynamic LB), R (resilient LB)
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Prefix               Route Type   Metric   Pref    Flags    Next-Hop(s)
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
192.168.10.0/24      local        0        0       >        192.168.10.254(irb0.1)
192.168.10.1/32      arp-nd       0        1       >        192.168.10.1(irb0.1)
192.168.10.2/32      bgp-evpn     0        170     >        2.2.2.2(tunnel:vxlan, vni:10000)
192.168.10.3/32      bgp-evpn     0        170     >        3.3.3.3(tunnel:vxlan, vni:10000)
192.168.10.4/32      bgp-evpn     0        170     >        4.4.4.4(tunnel:vxlan, vni:10000)
192.168.20.0/24      local        0        0       >        192.168.20.254(irb0.2)
192.168.20.1/32      arp-nd       0        1       >        192.168.20.1(irb0.2)
192.168.20.2/32      bgp-evpn     0        170     >        2.2.2.2(tunnel:vxlan, vni:10000)
192.168.20.3/32      bgp-evpn     0        170     >        3.3.3.3(tunnel:vxlan, vni:10000)
192.168.20.4/32      bgp-evpn     0        170     >        4.4.4.4(tunnel:vxlan, vni:10000)
```
