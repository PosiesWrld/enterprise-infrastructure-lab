# Enterprise Infrastructure Lab

**Windows Server 2025 · Active Directory · Microsoft Entra ID · Azure Networking · Ubuntu Linux · Managed Disks · Security**

Hands-on enterprise infrastructure lab spanning on-premises Windows identity and Microsoft Azure. Focused on deployment, access control, network segmentation, hybrid identity, and systematic troubleshooting.

> **Project Status:** Ongoing hands-on infrastructure lab. Architecture diagrams illustrate the implemented environment and will evolve as new components are deployed.

## Architecture

### Hybrid identity
```mermaid
flowchart LR
  subgraph OnPrem["On-premises lab"]
    DC["Windows Server 2025<br/>AD DS + DNS<br/>corp.local"]
    PC["Domain-joined Windows 11 client"]
    OU["Pilot organizational unit"]
    PC --> DC
    OU --> DC
  end
  DC -->|"Entra Cloud Sync<br/>gMSA · Password Hash Sync<br/>OU-scoped provisioning"| ENTRA["Microsoft Entra ID<br/>Synced user identities"]
```

### Azure network environments

The Windows infrastructure and the newer Ubuntu AZ-104 lab are shown side by side. **They are separate environments/subscriptions; no cross-tenant connection is implemented yet.**
```mermaid
flowchart TB
  subgraph V1["VNet1 · 10.10.0.0/16"]
    subgraph S1["ServerSubnet · 10.10.1.0/24 · NSG"]
      SRV["AZ-SRV01 · 10.10.1.4<br/>Public IP · source-restricted RDP"]
    end
    subgraph S2["TestSubnet · 10.10.2.0/24 · NSG"]
      TEST["AZ-TEST01 · 10.10.2.4<br/>Private IP only"]
      NAT["NAT Gateway"]
      TEST -->|"Outbound SNAT"| NAT
    end
  end
  subgraph V2["VNet2 · 10.11.0.0/16"]
    subgraph APPNET["AppSubnet · 10.11.10.0/24"]
      APP["AZ-APP01 · private-only"]
    end
  end
  V1 <-->|"VNet peering"| V2
  DNS["Private DNS · lab.internal<br/>Links to both VNets<br/>Manual AZ-APP01 A record"]
  DNS -.-> V1
  DNS -.-> V2
  NAT --> INTERNET["Internet"]
```

### Windows and Linux Azure labs — side-by-side view

```mermaid
flowchart LR
  subgraph WIN["Existing Azure Windows infrastructure"]
    direction TB
    W1["VNet1 · 10.10.0.0/16"]
    W1 --> WS["ServerSubnet · 10.10.1.0/24<br/>AZ-SRV01 · Windows · restricted RDP"]
    W1 --> WT["TestSubnet · 10.10.2.0/24<br/>AZ-TEST01 · private-only · NAT egress"]
    W2["VNet2 · 10.11.0.0/16"]
    W2 --> WA["AppSubnet · 10.11.10.0/24<br/>AZ-APP01 · private-only"]
    W1 <-->|"VNet peering"| W2
  end
  subgraph LINUX["New AZ-104 Linux lab · separate subscription"]
    direction TB
    LV["VNet-Lab · 10.20.0.0/16"]
    LV --> LS["ComputeSubnet · 10.20.1.0/24"]
    LS --> VM["AZ-LINUX01 · Ubuntu<br/>Public IP · SSH/22 restricted by NSG"]
    VM --> OS["OS disk · 30 GiB · ext4 root"]
    VM --> DATA["Data disk · 32 GiB · Standard SSD LRS<br/>Attached as /dev/sdb · not yet mounted"]
  end
  WIN -.-|"Future cross-tenant integration<br/>NOT configured"| LINUX
```

*Linux lab names and CIDRs are the intended design; confirm deployed values in Azure Portal. The dotted link is a future plan, not a working connection.*

*Hybrid identity synchronization does not imply an on-premises-to-Azure VPN or private network connection.*

## What I implemented

| Domain | Implementation |
|---|---|
| Windows identity | Windows Server 2025 DC, `corp.local`, AD DS, DNS, domain-joined Windows 11 client, OUs and groups |
| Group Policy | Password and account lockout policies; standard-user restrictions |
| Hybrid identity | Entra Cloud Sync provisioning agent, gMSA, password hash synchronization, pilot OU scoping |
| Azure compute | Windows Server VMs in segmented subnets, private addressing, restricted RDP; Ubuntu VM with source-restricted SSH |
| Networking | Two VNets, subnet NSGs, NAT Gateway, peering, private DNS, route-table experiments |
| Linux and storage (Nov 14, 2026) | Deployed `AZ-LINUX01`, authenticated using an SSH key from PowerShell, attached a 32 GiB Standard SSD LRS managed disk; verified `/dev/sdb` with `lsblk` (not yet partitioned or mounted) |
| Diagnostics | Network Watcher Next Hop, IP Flow Verify, Connection Troubleshoot; PowerShell, Windows and Linux CLI tools |

## Troubleshooting and validation

| Scenario | Diagnostic approach | Observed result |
|---|---|---|
| Inter-subnet ping failure | `Test-NetConnection` TCP/3389 and Windows Firewall inspection | RDP reachable; ICMP succeeded after guest firewall rule change |
| Azure deployment denied | Activity Log and Azure Policy | Found allowed-region restriction; redeployed in North Central US |
| Entra Cloud Sync provisioning errors | Provisioning logs, OU scope and distinguished names | Corrected scoping; pilot user synchronized |
| Private VM lacked Internet egress | NAT Gateway association and external IP verification | AZ-TEST01 retained private IP; outbound used NAT public IP |
| Test UDR disrupted RDP | Effective routes, `10.10.1.0/24 → None` | Identified broken return path; removed test UDR and restored access |
| Cross-VNet DNS resolution failure | Verified TCP reachability separately from DNS | Linked private DNS zone and added A record; name resolution succeeded |

## Engineering decisions

- **Least-exposed management:** allowed public RDP to AZ-SRV01 only from the administrator's current public IP.
- **Private workloads:** AZ-TEST01 and AZ-APP01 have no dedicated public IP; NAT Gateway provides AZ-TEST01 outbound egress.
- **Layered diagnostics:** distinguished Azure routing, NSGs, Windows Firewall, DNS, and service availability.
- **Scoped rollout:** synchronized a pilot OU before broader hybrid identity expansion.

## Roadmap

Next: partition/format the Linux data disk, mount by UUID using `/etc/fstab`, verify persistence after reboot, and test disk snapshots. Later: monitoring/logging, backup/recovery, hybrid device management, and **cross-tenant connectivity** between the currently separate Azure environments.

*Supporting configuration screenshots and validation evidence are documented in a separate internal wiki. The full wiki is intentionally excluded from this public repository; sanitized screenshots may be added under `docs/evidence/`.*
