# Enterprise IT Infrastructure & Active Directory High Availability Lab

An end-to-end implementation and stress-testing of a resilient corporate IT infrastructure virtualized inside **PNETLab**. This project converges core network engineering (**Cisco L3 Routing & Switching**) with enterprise systems administration (**Windows Server 2022 AD DS, Identity Governance, High Availability, and Disaster Recovery**).

---

## Architectural Topology & Network Matrix

![Enterprise Topology](topologies/pnetlab-enterprise-topology.png)

### Addressing & VLAN Segmentation

| Segment / Role | Subnet | Gateway / SVI | Interface / Node | Enterprise Scope |
| :--- | :--- | :--- | :--- | :--- |
| **Servers (VLAN 10)** | `192.168.10.0/24` | `192.168.10.1` | L3 Core SVI | DC01 (`.10`), ADC (`.11`) |
| **IT Clients (VLAN 20)** | `192.168.20.0/24` | `192.168.20.1` | L3 Core SVI | DHCP Scope (`.50` - `.150`) |
| **HR Clients (VLAN 30)** | `192.168.30.0/24` | `192.168.30.1` | L3 Core SVI | DHCP Scope (`.50` - `.100`) |
| **Transit Link** | `192.168.99.0/30` | — | Gi0/0 (Router: `.1` / Switch: `.2`) | Routed Point-to-Point |
| **WAN / Cloud** | Dynamic | ISP Gateway | Router Gi0/1 | PAT / NAT Overload |

---

## Core Engineering Pillars

### 1. Network Fabric & Transit Routing (Cisco IOS)
* **Layer 3 Switching:** SVIs configured for VLANs 10, 20, and 30 to terminate default gateways locally. Access ports hardened with `spanning-tree portfast`.
* **DHCP Cross-Subnet Forwarding:** Dual `ip helper-address` statements applied to client SVIs (`Vlan20`, `Vlan30`) targeting both primary and secondary servers. The switch intercepts broadcast `DHCPDISCOVER` frames (UDP 67), injects its SVI IP into the `giaddr` field, and unicasts packets to the identity tier.
* **Edge Routing & NAT:** Symmetric static return routes configured on the Cisco Edge Router pointing to `192.168.99.2` for internal subnets; outbound translation handled via NAT Overload (PAT) on the outside interface.

### 2. Identity Baseline & Storage Access Governance
* **Forest Provisioning:** Root domain `amazon.local` deployed at Windows Server 2016 functional level.
* **DNS Services:** Integrated zones with dynamic updates, domain locator SRV records (`_msdcs`), reverse lookup (`10.168.192.in-addr.arpa`), and an alias `files.amazon.local` (CNAME).
* **Storage Isolation:** Central SMB repository (`C:\Shares`) enforcing explicit zero-trust DACLs:
  * Inherited permissions severed; generic `BUILTIN\Users` entries purged.
  * Departmental Global Security Groups (`IT-Staff`, `HR-Staff`) mapped to isolated folders with Modify rights.
  * Registry bypass for NetBIOS/FQDN SMB alias validation:
    * `HKLM\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters\DisableStrictNameChecking = 1`
    * `HKLM\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters\OptionalNames = "files"`
* **Group Policy Governance:**
  * IT OU: Enforced restriction of the Windows Run dialog (`Win+R`).
  * HR OU: Prohibited access to Control Panel and PC Settings.
  * Workstations: Assigned machine-targeted silent deployment of `7-Zip.msi` during pre-logon boot under the `NT AUTHORITY\SYSTEM` context.

### 3. Hypervisor Clone Remediation & Secondary Controller (ADC)
* **Machine SID Duplication Bypass:** Overrode the Sysprep generalization fatal crash on cloned hypervisor instances via registry manipulation before domain join:
  ```cmd
  reg add "HKLM\SYSTEM\Setup\Status\SysprepStatus" /v GeneralizationState /t REG_DWORD /d 7 /f
  reg add "HKLM\SYSTEM\Setup\Status\SysprepStatus" /v CleanupState /t REG_DWORD /d 2 /f
  reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SoftwareProtectionPlatform" /v SkipRearm /t REG_DWORD /d 1 /f
  sysprep.exe /oobe /generalize /reboot
  ```
* **ADC Promotion:** Promoted `ADC.amazon.local` (`192.168.10.11`) as a full replica hosting DNS and Global Catalog (GC), synchronized within Kerberos clock-skew tolerances (<5 minutes).
* **AD Sites and Services:** Partitioned enterprise topology into `Amman` (HQ) and `Irbid` (Branch) linked across `DEFAULTIPSITELINK`.
* **AD Recycle Bin:** Enabled forest-wide; verified object attribute preservation (`objectSid`, `memberOf`) via tombstone bypass and instant recovery of deleted administrative principals.

### 4. High-Availability DHCP Clustering
* Deployed DHCP Failover in **Hot Standby** mode between `DC01` (Active) and `ADC` (Standby) across TCP 647.
* **Parameters:** 5% Reserve Pool, 1-hour Maximum Client Lead Time (MCLT), 10-minute State Switchover Interval.
* Verified state transitions: Upon primary isolation, ADC serviced address requests under `Communications Interrupted` within the MCLT lease boundary.

---

## Disaster Recovery: Forced FSMO Seizure & Cleanup

Simulated sudden, permanent hardware destruction of Primary Domain Controller `DC01` (hard power down).

### 1. FSMO Role Seizure
Bypassed RPC connection timeout deadlocks in MMC snap-ins by unilaterally seizing all five roles via PowerShell on `ADC`:
```powershell
Move-ADDirectoryServerOperationMasterRole -Identity "ADC" `
  -OperationMasterRole SchemaMaster, DomainNamingMaster, PDCEmulator, RIDMaster, InfrastructureMaster `
  -Force
```

### 2. Directory Metadata Cleanup
* **ADUC:** Deleted orphaned `DC01` computer object from `OU=Domain Controllers`, selecting the permanently offline flag to strip directory references.
* **Sites and Services:** Purged stale `DC01` server object and lingering NTDS connection objects under the `Amman` container.
* **DNS Remediation:** Deleted obsolete `A`, `NS`, and `CNAME` records pointing to `192.168.10.10` in `amazon.local` and `_msdcs.amazon.local`.

### 3. Core Services Alignment
* Updated DHCP Scope Option 006 (DNS Servers) to distribute surviving DC `192.168.10.11`.
* Validated dynamic lease allocation on endpoint `PC-IT2` via switch relay, confirming 1-hour MCLT window enforcement.
* Confirmed autonomous RID pool allocation and writable directory status by provisioning new security principals without SID collisions.

---

## Engineering Troubleshooting Log

| Symptom / Failure | Root Technical Cause | Remediation Applied |
| :--- | :--- | :--- |
| **SVI Vlan10 state `down/down`** | Cisco Autostate algorithm suppressed SVI because no physical access ports were actively forwarding in VLAN 10 after node startup without persisted config. | Re-assigned physical interfaces `g0/1` and `g0/2` to VLAN 10; enabled immediate Spanning Tree forwarding via `spanning-tree portfast`. |
| **MMC FSMO transfer hung indefinitely** | Management snap-ins require synchronous RPC communication with the offline role holder before offering transfer prompts. | Bypassed RPC timeouts by issuing `Move-ADDirectoryServerOperationMasterRole` with the `-Force` parameter via PowerShell. |
| **Workstations failed DNS post-outage** | Active DHCP leases distributed decommissioned DC (`192.168.10.10`) under Option 006. | Reconfigured DHCP Option 006 on surviving server to point to `192.168.10.11`; forced renewal across endpoints via `ipconfig /renew`. |

---

## Verification & Health Check Commands

```cmd
:: Verify FSMO role ownership
netdom query fsmo

:: Active Directory operational diagnostics
dcdiag /v

:: Replication health check
repadmin /replsummary

:: Endpoint lease and DNS audit
ipconfig /all
nslookup amazon.local
```
