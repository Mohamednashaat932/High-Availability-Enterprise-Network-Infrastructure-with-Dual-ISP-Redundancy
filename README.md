# Enterprise Network with HA FortiGate Firewalls and Dual-ISP SD-WAN

A resilient enterprise network designed and built in **EVE-NG**, where the failure of a single device or link does not interrupt internet access. It combines two ISPs, a FortiGate high-availability pair, two Layer 3 core switches and dual-homed access switches.

![Topology](images/topology.png)

## Highlights

- **FortiGate HA** (active-passive) with dual heartbeat links and session pickup
- **Dual-ISP SD-WAN** with health checks and automatic failover (Vodafone primary, WE backup)
- **OSPF** between the Layer 3 cores and the firewall cluster over dedicated transit VLANs
- **HSRP + MSTP** per-VLAN load sharing across both cores
- **NAT, firewall policy and web filtering** for controlled internet breakout
- Dual-homed access layer with port security

## Technology Stack

| Layer | Platform |
|---|---|
| Lab platform | EVE-NG |
| Firewalls | 2 x FortiGate VM64-KVM (FortiOS 6.4.6) |
| Core | 2 x Cisco L3 switches (NH_PRIM, NH_SEC) |
| Access | 3 x Cisco L2 switches (ACC-1, ACC-2, ACC-3) |
| WAN | 2 x Cisco routers (ISP1 Vodafone, ISP2 WE) and 2 L2 WAN switches |
| Protocols | OSPF, HSRP, MSTP, EtherChannel, SD-WAN, NAT |

## Architecture and Redundancy

| Layer | Devices | Redundancy mechanism | Failure handled |
|---|---|---|---|
| Internet access | ISP1, ISP2 | SD-WAN zone with health checks | ISP outage or link failure |
| WAN switching | WAN_SWITCH1, WAN_SWITCH2 | Each FortiGate connects to both switches | Loss of one WAN path |
| Firewall | FG-1 (primary), FG-2 (secondary) | Active-passive HA, shared interface IPs and MACs | Firewall failure |
| Core | NH_PRIM, NH_SEC | HSRP, OSPF, dedicated transit VLAN per core | Core failure |
| Access | ACC-1, ACC-2, ACC-3 | Two uplinks per switch, MSTP | Uplink or core failure |

> **Cabling rule for HA:** the same port on both FortiGates must connect to the same switch, because interface settings are synchronized between the units.

![Cabling before](images/cabling_before.png)
![Cabling after](images/cabling_after.png)

## IP Addressing

Full reference: [Subnetting_Table.txt](Subnetting_Table.txt)

| VLAN | Name | Subnet | HSRP VIP | Active core |
|---|---|---|---|---|
| 10 | PCs / Users | 10.10.10.0/24 | 10.10.10.254 | NH_PRIM |
| 20 | IP Phones | 10.20.20.0/24 | 10.20.20.254 | NH_PRIM |
| 30 | Wi-Fi / WLAN | 10.30.30.0/24 | 10.30.30.254 | NH_SEC |
| 40 | Management | 10.40.40.0/24 | 10.40.40.254 | NH_SEC |
| 50 | Servers | 10.50.50.0/24 | 10.50.50.254 | NH_SEC |

| Link | Subnet | Addresses |
|---|---|---|
| FortiGate port1 to NH_PRIM (VLAN 12) | 12.0.0.0/29 | FortiGate .1 (cluster IP), NH_PRIM SVI .2 |
| FortiGate port2 to NH_SEC (VLAN 13) | 12.1.0.0/29 | FortiGate .1 (cluster IP), NH_SEC SVI .2 |
| WAN 1: Vodafone | 100.64.0.0/29 | ISP1 .1, FortiGate port6 .3 |
| WAN 2: WE | 100.65.0.0/29 | ISP2 .1, FortiGate port7 .3 |
| HA heartbeat | port3, port4 | Direct links FG-1 to FG-2 |
| Management (port5) | 192.168.1.0/24 | FG-1 .52, FG-2 .53 |

## Design Details

### Layer 2 and gateway redundancy

- **MSTP** region `SPMST`: instance 1 (VLANs 10, 20) rooted on NH_PRIM; instance 2 (VLANs 30, 40, 50) rooted on NH_SEC.
- **HSRP** virtual gateway `.254` per VLAN; the core that is root for the VLAN has priority 110 with preempt.
- **Port-channel1** (Gi0/3 and Gi1/0) trunks VLANs 10 to 50 between the cores.
- Host ports use sticky port-security and PortFast.

### Core to firewall routing (OSPF)

- Each core reaches both FortiGates through an access port in a transit VLAN (12 on NH_PRIM, 13 on NH_SEC) with an SVI. The active unit owns the cluster IP, so a firewall failover needs no routing change on the cores.
- OSPF process 1, Area 0. Router IDs: NH_PRIM `1.1.1.1`, NH_SEC `2.2.2.2`, FortiGate `3.3.3.3`.
- The FortiGate originates the default route (`default-information-originate always`) and learns VLANs 10 to 50 from both cores over equal-cost paths. WAN links are not part of OSPF.

![OSPF](images/ospf_config.png)

### FortiGate high availability

| Setting | Value |
|---|---|
| Mode / group | Active-passive, `ENTERPRISE-HA` |
| Priority | FG-1 = 200 (primary), FG-2 = 101 (secondary) |
| Heartbeat | port3, port4 |
| Session pickup | Enabled |
| Monitored interfaces | port1, port2 |
| Management interface | port5 (reserved) |

![HA monitor](images/ha_monitor.png)

### Dual-ISP edge and SD-WAN

- ISP routers use a `/29` toward their WAN switch and a DHCP outside interface, with NAT overload. Traffic is translated twice: by the FortiGate, then by the ISP router.
- SD-WAN zone `ISP`: member 1 = port6 (Vodafone, 100.64.0.1), member 2 = port7 (WE, 100.65.0.1).
- Health check `Internet_Check` probes 8.8.8.8 and 1.1.1.1 every 1000 ms (dead after 5 failures, restored after 5 successes).
- SD-WAN rule `Primary_ISP1` prefers member 1 and falls back to member 2. One static default route points to the SD-WAN.

![SD-WAN](images/sdwan_zone.png)
![ISP1 NAT](images/isp1_nat_table.png)

### Security

- Policy `LAN_TO_INTERNET_BLK_SOCIAL`: port1 and port2 to the ISP zone, accept, NAT enabled, certificate inspection, full logging. Everything else is dropped by the implicit deny.
- Web filter: static URL filter for social-media domains with per-domain actions.

![Policy](images/firewall_policy.png)
![URL filter](images/url_filter.png)

## Validation

- Hosts in VLAN 50 (VPCS) and a Windows host reach `8.8.8.8` with 0% packet loss.
- Trace path from VLAN 50: NH_SEC (HSRP gateway) -> FortiGate cluster IP `12.1.0.1` -> ISP1 `100.64.0.1` -> internet.
- Both cores show Full OSPF adjacency with the FortiGate; both FortiGates are Synchronized in the HA monitor.

![Validation](images/validation_vpcs.png)

## Troubleshooting Notes

| Issue | Root cause | Fix |
|---|---|---|
| HA would break on failover | Cables crossed between the two FortiGates | Same port on both units to the same switch |
| Core could not reach the firewall | Routed ports with different subnets cannot share the cluster IP | Transit VLAN with an SVI |
| SVI down/down | VLAN missing from the VLAN database | Create the VLAN and assign the ports |
| Host ports blocked (BKN, BA_Inc) | PortFast network type enabled Bridge Assurance | PortFast edge on host ports |
| No default route on the firewall | Health checks marked both ISPs dead | Fixed the internet cloud for the ISP routers |
| Inconsistent WAN design | Mask mismatch and OSPF on WAN ports | Aligned on /29, removed WAN from OSPF |
| Interface not accepted by SD-WAN | Static routes still referenced the WAN ports | Removed them, then added members |

## Next Steps

- Record failover test results: ISP outage, FortiGate failover, core failover.
- Add IPS, antivirus and DNS filtering profiles.
- Centralized logging (syslog) and monitoring (SNMP).

## Repository Structure

```
.
├── README.md
├── Subnetting_Table.txt
└── images/
```

## Author

**Mohamed Nashaat**, Network Engineer
