# Addressing

The base network `10.0.0.0/8` is carved up into `/16`s by site.

## Sites
| Site # | Name | Subnet | Notes | Status |
| --- | --- | --- | --- | --- |
| 0 | home | `10.0.0.0/16` | physical home network | **live** |
| 1 | powerhouse | `10.1.0.0/16` | Proxmox internal (GPU inference) | **live** |
| 2 | storage | `10.2.0.0/16` | Proxmox internal (low-power always-on) | **planned** |
| 3 | edge | `10.3.0.0/16` | Proxmox internal (public web services) | **planned** |

## Rules
- second octet = site number
- third octet = VLAN ID
- `10.x.x.254` is the layer 3 gateway on every subnet
- `.1` to `.9` reserved for infrastructure devices
- `.150-.249` is reserved for DHCP, where applicable
- Inter-VLAN traffic is default-deny

So for `10.1.30.32`: site 1 (powerhouse), VLAN 30 (compute), host 32. I use this scheme so that when an address shows up in a firewall log, I instantly know which site and which trust zone it is without needing to consult my documentation.

## VLANs - home (site 0)
| VLAN | Name | Subnet | Purpose |
| --- | --- | --- | --- |
| 10 | management | `10.0.10.0/24` | management devices |
| 20 | services | `10.0.20.0/24` | internal service machines |
| 30 | infra | `10.0.30.0/24` | network infrastructure devices |
| 40 | family | `10.0.40.0/24` | trusted personal devices |
| 50 | guest | `10.0.50.0/24` | untrusted personal devices |
| 60 | iot | `10.0.60.0/24` | untrusted smart devices |
| 70 | cctv | `10.0.70.0/24` | security cameras |

## VLANs - powerhouse (site 1)
| VLAN | Name | Subnet | Purpose |
| --- | --- | --- | --- |
| 10 | management | `10.x.10.0/24` | management machines |
| 20 | services | `10.x.20.0/24` | internal service machines |
| 30 | compute | `10.x.30.0/24` | inference with GPU passthrough |
| 40 | web | `10.x.40.0/24` | untrusted machines needing WAN access |
| 50 | isolated | `10.x.50.0/24` | untrusted machines with no network access |
