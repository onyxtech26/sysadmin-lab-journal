# Incident Response & Troubleshooting Log

This log documents issues, configuration errors, and diagnostic steps encountered across the lab lifecycle, written in **STAR format** (Situation, Task, Action, Result). 

Technical interviewers frequently ask: *"Tell me about a time a system or service didn't work and how you diagnosed it."* This log serves as direct reference material for those discussions.

---

## Incident 01: Host Machine SSH Connection Timeout to `lab-linux-01`

* **Date:** Phase 1 (Step 1.4)
* **Impacted Node:** `lab-linux-01` (`10.10.10.11`)
* **Category:** Network Adapter Configuration / Hypervisor Routing

### Situation
After completing the Ubuntu Server installation and applying static IP `10.10.10.11` via Netplan on the `LabNet` NAT Network, attempting to connect directly from the Windows host terminal (`ssh user@10.10.10.11`) resulted in a connection timeout (`Destination Host Unreachable` / timeout).

### Task
Establish reliable host-to-guest SSH terminal access into both `lab-linux-01` and `DC01` without exposing the VMs to the local physical LAN (Bridged mode) or adding unnecessary secondary virtual adapters.

### Action
1. Analyzed hypervisor network topology: In VirtualBox, a **NAT Network** isolates the guest subnet from host routing. Packets originating from the physical host cannot reach `10.10.10.x` directly without host-side route tables or port translation.
2. Navigated to VirtualBox `Tools -> Network Manager -> NAT Networks -> LabNet -> Port Forwarding`.
3. Created explicit forwarding rules:
   * **Rule 1 (Linux):** Host Port `2222` -> Guest IP `10.10.10.11`, Guest Port `22`.
   * **Rule 2 (Windows Server):** Host IP `127.0.0.1`, Host Port `2223` -> Guest IP `10.10.10.10`, Guest Port `22`.
4. Tested listener response on host via PowerShell: `Test-NetConnection -ComputerName 127.0.0.1 -Port 2222` and `-Port 2223`.

### Result
Both guest virtual machines became instantly accessible from the host terminal using standard port switches:
* Linux: `ssh -p 2222 <username>@127.0.0.1`
* Windows DC: `ssh -p 2223 Administrator@127.0.0.1`
This preserved the isolated lab network boundary while delivering responsive, headless administration across both operating systems.

---

## Incident 02: Active Directory Installation Prerequisite Check Warning (Loopback DNS)

* **Date:** Phase 2 (Step 2.3)
* **Impacted Node:** `DC01` (`10.10.10.10`)
* **Category:** Domain Services / DNS Configuration

### Situation
During preliminary checks before running `Install-ADDSForest`, the network adapter had an upstream public DNS server (`8.8.8.8`) configured from the initial setup instead of local loopback.

### Task
Ensure that promoting the server to a Domain Controller does not cause split-brain DNS resolution or fail the Active Directory pre-promotion validation tests.

### Action
1. Launched `sconfig` -> Option `8` (Network Settings).
2. Pointed the primary DNS server address to `127.0.0.1`.
3. Executed `Install-ADDSForest` with the `-InstallDns:$true` parameter to ensure the integrated DNS service automatically provisions the forward lookup zone `lab.local` and `_msdcs` SRV records locally.

### Result
The AD DS forest installation completed without DNS delegation errors. Upon reboot, running `Resolve-DnsName -Name "_ldap._tcp.dc._msdcs.lab.local" -Type SRV` verified that `DC01` successfully resolved its own Kerberos and LDAP service records.
