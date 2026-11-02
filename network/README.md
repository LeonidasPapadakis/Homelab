# Network plan

The base network `10.0.0.0/8` is split into `/16`s by site.

## Sites
| Site # | Name | Subnet | Notes | Status |
| --- | --- | --- | --- | --- |
| 0 | home | `10.0.0.0/16` | physical home network | **live** |
| 1 | powerhouse | `10.1.0.0/16` | Proxmox internal (GPU inference) | **live** |
| 2 | storage | `10.2.0.0/16` | Proxmox internal (RAID drive array) | **planned** |
| 3 | edge | `10.3.0.0/16` | Proxmox internal (public web services) | **planned** |

## Rules
- second octet = site number
- third octet = VLAN ID
- `.2 - .9` reserved for infrastructure devices
- `.150-.249` is reserved for DHCP, where applicable
- the first address of the subnet is the layer 3 gateway on every subnet
- Inter-VLAN traffic is default-deny

So for `10.1.20.32`: site 1 (powerhouse), VLAN 20 (services), host 32. I use this scheme so that when an address shows up in a firewall log, I instantly know which site and which trust zone it is without needing to consult my documentation.

## Home - site 0
| VLAN | Name | Subnet | Hosts | Egress | Gateway |
| --- | --- | --- | --- | --- | --- |
| 10 | management | `10.0.10.0/24` | management devices | full network access |
| 20 | services | `10.0.20.0/24` | internal service machines | LAN by /32 grants and WAN |
| 30 | infra | `10.0.30.0/24` | network infrastructure devices | none, NTP where needed | 
| 40 | family | `10.0.40.0/24` | trusted personal devices | LAN by /32 grants and WAN |
| 50 | guest | `10.0.50.0/24` | untrusted personal devices | WAN |
| 60 | iot | `10.0.60.0/24` | untrusted smart devices | WAN |
| 70 | cctv | `10.0.70.0/24` | security cameras | none |

## Powerhouse - site 1
| VLAN | Name | Subnet | Hosts | Egress | Gateway |
| --- | --- | --- | --- | --- | --- |
| 10 | management | `10.x.10.0/24` | management machines | full network access | `10.x.10.1` (OPNsense) |
| 20 | services | `10.x.20.0/24` | internal service machines | WAN only | `10.x.20.1` (OPNsense) |
| 30 | agent-web | `10.x.30.0/24` | AI agent machines with WAN access | LiteLLM and WAN via agent-VPN (kill-switched) | `10.x.30.1` (OPNsense) |
| 40 | agent-sandbox | `10.x.40.0/24` | AI agent machines with no WAN access | LiteLLM | `10.x.40.1` (OPNsense) |
| 50 | workbench | `10.x.50.0/24` | general-purpose human-driven machines | WAN via workbench-VPN (kill-switched) | `10.x.50.1` (OPNsense) |
| 60 | intake | `10.x.60.0/24` | web scrapers processing untrusted content | WAN via intake-VPN (kill-switched) | `10.x.60.1` (OPNsense) |
| none | arena | `192.0.2.0/24` | machines with simulated web access | isolated virtual bridge only | `192.0.2.1` (PVE VMBR2) |
| none | vault | none | isolated machines with no network access at all | none | none |
