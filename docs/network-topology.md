# Lab Network Topology & Addressing Scheme

This document defines the network architecture, addressing layout, and hardware allocations across all phases of the lab.

---

## 1. Subnet Architecture

| Network Name | Purpose | Subnet CIDR | Usable Range | Gateway | Default DNS |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **LabNet** *(Phases 0–3)* | Isolated NAT Network for initial build | `10.10.10.0/24` | `10.10.10.2 – 10.10.10.254` | `10.10.10.1` | `10.10.10.10` / `8.8.8.8` |
| **Servers VLAN 10** *(Phase 4+)* | Protected server infrastructure | `10.10.20.0/24` | `10.10.20.2 – 10.10.20.254` | `10.10.20.1` | `10.10.20.10` |
| **Clients VLAN 20** *(Phase 4+)* | Workstation & end-user machines | `10.10.30.0/24` | `10.10.30.100 – 10.10.30.200` (DHCP) | `10.10.30.1` | `10.10.20.10` |

---

## 2. Node Specifications & Static IP Assignments

### Phase 1 & 2 Allocations (`LabNet`: `10.10.10.0/24`)

| Hostname | Role | OS | IP Address | MAC / Interface | RAM | vCPU | Storage |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **DC01** | Root Domain Controller (`lab.local`), DNS, DHCP | Windows Server 2022 Core | `10.10.10.10/24` | Adapter 1 (LabNet) | 4096 MB | 2 | 40 GB dynamic |
| **lab-linux-01** | Linux Services, Web, Storage | Ubuntu Server 24.04/26.04 | `10.10.10.11/24` | Adapter 1 (`enp0s3`) | 2048 MB | 2 | 25 GB dynamic |
| **lab-client-01** | Domain-Joined Workstation | Windows 10/11 Pro | DHCP (`10.10.10.100–200`) | Adapter 1 (LabNet) | 2048 MB | 2 | 30 GB dynamic |

---

## 3. Port Matrix & Required Traffic Paths

| Source | Destination | Protocol / Port | Service / Purpose |
| :--- | :--- | :--- | :--- |
| `Host Machine` | `10.10.10.11` | TCP / 22 | OpenSSH Remote Administration (Linux) |
| `Host Machine` | `10.10.10.10` | TCP / 22 | OpenSSH Remote Administration (Windows Server Core) |
| `lab-client-01` | `10.10.10.10` | TCP/UDP 53 | DNS Resolution |
| `lab-client-01` | `10.10.10.10` | TCP/UDP 88 | Kerberos Authentication |
| `lab-client-01` | `10.10.10.10` | TCP/UDP 389 | LDAP Directory Lookups |
| `lab-client-01` | `10.10.10.10` | TCP 445 | SMB / Group Policy Retrieval (`SYSVOL`) |
| `All Nodes` | `10.10.10.1` | ICMP / Any | Gateway NAT routing to Internet |
