# Phase 2: Windows Server Core & Active Directory Domain Services

**Node:** `DC01` (`lab-dc-01`)  
**Operating System:** Windows Server 2022 Standard (Server Core)  
**Forest Root Domain:** `lab.local`  
**NetBIOS Name:** `LAB`  
**Assigned Static IP:** `10.10.10.10/24`  

---

## 1. Phase Overview

Phase 2 focuses on deploying and configuring the core enterprise identity infrastructure using **Windows Server 2022 Server Core**. 

Rather than relying on the Windows Desktop Experience (GUI), this entire phase is executed via command-line tools: `sconfig`, `cmd.exe`, and **PowerShell**. This mirrors modern enterprise datacenter and cloud administration standards, minimizing attack surfaces and resource consumption.

---

## 2. Phase 2 Roadmap & Documentation Index

| Step | Document | Focus & Key Technologies | Status |
| :--- | :--- | :--- | :--- |
| **2.1 – 2.2** | [01-server-core-initial-config.md](./01-server-core-initial-config.md) | Server Core deployment, host renaming to `DC01`, static IP assignment via `sconfig` | **Complete** |
| **2.3** | [02-adds-forest-deployment.md](./02-adds-forest-deployment.md) | AD DS role installation, forest promotion (`lab.local`), DSRM password, post-install verification | **Complete** |
| **2.4** | [03-dhcp-configuration.md](./03-dhcp-configuration.md) | Windows DHCP Server role, scope `10.10.10.100–200`, DNS options, authorization | **Complete** |
| **2.5** | [04-ou-users-groups.md](./04-ou-users-groups.md) | Tiered OU structure (`Sales`, `IT`, `Finance`), security groups, bulk AD accounts | **Complete** |
| **2.6** | `05-group-policy.md` *(Planned)* | Group Policy Object creation, linking, and password policy enforcement | *Pending* |
| **2.7** | `06-domain-join-client.md` *(Planned)* | Domain-joining a Windows client VM via PowerShell | *Pending* |
| **2.8** | `07-ticket-simulation.md` *(Planned)* | Helpdesk scenario: User onboarding, group membership, share mapping, ticket documentation | *Pending* |

---

## 3. Key Design Decisions

1. **Why Server Core instead of Desktop Experience?**
   * **Security**: Dramatically reduces the OS attack surface by eliminating graphical shells, Internet Explorer/Edge runtimes, and unnecessary UI binaries.
   * **Resource Efficiency**: Idles under 1.2 GB of RAM compared to ~2.8 GB for full Desktop Experience.
   * **Operational Discipline**: Enforces PowerShell-first administration, building transferable skills for enterprise cloud environments (Azure, AWS, automated CI/CD infrastructure).

2. **Loopback DNS Configuration (`127.0.0.1`)**:
   * Prior to promoting the server to a Domain Controller, its DNS was pointed to `127.0.0.1`. When `Install-ADDSForest` executes with `-InstallDns:$true`, the DC initializes an integrated Active Directory DNS zone, serving lookups for itself and the entire subnet.
