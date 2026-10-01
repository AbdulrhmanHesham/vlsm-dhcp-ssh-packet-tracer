# VLSM, DHCP & SSH — Small Company Network (Cisco Packet Tracer)

A hands-on network design project built in Cisco Packet Tracer, covering subnetting with VLSM, automated addressing with DHCP, and secure remote management with SSH.

## Scenario

A small company is given the address block **172.16.50.0/24** and has three departments:

| Department | Hosts needed |
|---|---|
| Sales | 50 |
| Support | 20 |
| Accounting | 10 |

The goal: use VLSM to assign each department the smallest subnet that fits, automate addressing with DHCP, and secure every device with SSH-only management (Telnet disabled).

## Topology

![Topology](screenshots/topology.png)

- **1 router (2911)**: one interface per department, each acting as that subnet's gateway
- **3 switches (2960)**, one per department — no links between switches, so each department stays in its own broadcast domain and subnet
- **3 PCs per switch**, each getting its address from DHCP

## VLSM Addressing Plan

![VLSM Table](screenshots/vlsm-table.png)

| Department | Hosts | Subnet | Mask | Gateway | Usable range | Broadcast |
|---|---|---|---|---|---|---|
| Sales | 50 | 172.16.50.0/26 | 255.255.255.192 | .1 | .1 – .62 | .63 |
| Support | 20 | 172.16.50.64/27 | 255.255.255.224 | .65 | .65 – .94 | .95 |
| Accounting | 10 | 172.16.50.96/28 | 255.255.255.240 | .97 | .97 – .110 | .111 |

**Address space:** 112 addresses used (172.16.50.0 – .111), 144 unused (172.16.50.112 – .255), reserved for future growth.

Full hand-written calculations: [`docs/vlsm-handwritten.jpg`](docs/vlsm-handwritten.jpg)

## Device Addressing

| Device | Interface | IP address | Connects to | Gateway |
|---|---|---|---|---|
| R1 | Gi0/0 | 172.16.50.1 | SW-Sales | n/a |
| R1 | Gi0/1 | 172.16.50.65 | SW-Support | n/a |
| R1 | Gi0/2 | 172.16.50.97 | SW-Accounting | n/a |
| SW-Sales | VLAN 1 | 172.16.50.5 | R1 Gi0/0 | 172.16.50.1 |
| SW-Support | VLAN 1 | [your IP] | R1 Gi0/1 | 172.16.50.65 |
| SW-Accounting | VLAN 1 | [your IP] | R1 Gi0/2 | 172.16.50.97 |

Full table: [`VLSM_Addressing_Table.xlsx`](VLSM_Addressing_Table.xlsx)

## DHCP

The router runs one pool per department, with the gateway and switch management addresses excluded from each pool.

![DHCP Bindings](screenshots/dhcp-binding.png)

All 9 PCs receive an address automatically within their department's range.

## SSH

- SSH is enabled on the router and all three switches.
- Telnet is disabled everywhere (`transport input ssh` on every VTY line).
- A single lab account is shared across devices for simplicity; in production, I'd use AAA with per-admin accounts (TACACS+/RADIUS).

![SSH Login](screenshots/ssh-login.png)

Verified: SSH login from a PC in one department into a switch in a different department, proving both SSH and inter-subnet routing.

## Connectivity

![Ping Test](screenshots/ping-test.png)

PCs can reach their own gateway and PCs in other departments, confirming the router is routing correctly between subnets.

## Troubleshooting Notes

Two real bugs came up while building this, and tracking them down was the most useful part of the project.

**1. Sales PCs weren't getting DHCP addresses.**
The config had `ip dhcp excluded-address 172.16.50.1 172.16.50.65`. With two addresses, this command excludes the entire *range* between them, not just the two endpoints — so it excluded all of Sales' usable range. Found by comparing `show ip dhcp pool` (leased: 0) against `show running-config | section dhcp`. Fixed by excluding each address individually.

**2. A switch wasn't reachable over SSH from other departments.**
SSH worked from a PC in the same subnet as the switch, but not from other departments. The switch had no default gateway configured, so it had no route to send return traffic outside its own subnet. Fixed with `ip default-gateway <router-interface-IP>`.

## What I'd add next

- VLANs and trunking, to reduce the router/switch port count
- A routing protocol (OSPF) instead of directly-connected routes
- ACLs to restrict SSH access to specific hosts
- AAA for per-admin SSH credentials

## Files

- `VLSM-DHCP-SSH-Lab.pkt` — Packet Tracer project file
- `VLSM_Addressing_Table.xlsx` — full addressing tables
- `docs/vlsm-handwritten.jpg` — hand-worked VLSM calculations
- `screenshots/` — verification screenshots (topology, DHCP, ping, SSH)
