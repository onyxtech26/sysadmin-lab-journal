# Incident Response & Troubleshooting Log

This log documents issues, configuration errors, and diagnostic steps encountered across the lab lifecycle, written in **STAR format** (Situation, Task, Action, Result). 

Technical interviewers frequently ask: *"Tell me about a time a system or service didn't work and how you diagnosed it."* This log serves as direct reference material for those discussions.

---

## Incident 01: Host Machine SSH Connection Timeout to `lab-linux-01`

* **Date:** Phase 1 (Step 1.4)
* **Impacted Node:** `lab-linux-01` (`10.10.10.11`)
* **Category:** Network Adapter Configuration / Hypervisor Routing

### Situation
After completing the Ubuntu Server installation and applying the static IP `10.10.10.11` via Netplan, executing `ssh <user>@10.10.10.11` from the Windows host terminal resulted in an immediate connection timeout (`Connection timed out` / unreachable host).

### Task
Determine why the host machine could not route packets to the guest VM on `10.10.10.11` and establish stable remote terminal access.

### Action
1. Inspected VM network adapter status from the VirtualBox hypervisor settings.
2. Discovered Adapter 1 was left on default `NAT` instead of `NAT Network` (`LabNet`). Standard VirtualBox NAT isolates the VM in an individual internal sandbox with port translation, preventing direct inbound addressing from the host subnet without manual port forwarding.
3. Switched Adapter 1 attachment to **NAT Network** -> **LabNet**.
4. Inside the VM, verified interface state using `ip a show dev enp0s3` and confirmed the default gateway `10.10.10.1` was reachable via `ping -c 2 10.10.10.1`.
5. Tested port 22 listener state on the guest using `sudo ss -tulpn | grep :22`.

### Result
Once attached to `LabNet`, the host machine immediately completed the TCP 3-way handshake on port 22, and SSH key authentication succeeded. Documented this prerequisite in Phase 0 hypervisor standards.

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
