# Enterprise EVPN-VXLAN AIOps Lab — Complete Build Documentation

## From Zero to Closed-Loop Self-Healing Network Operations

**Author:** Faisal Mughal  
**Domain:** eve-ng-lab.local  
**Platform:** Cisco UCS C220 M5 → Proxmox VE 9.1 → EVE-NG  
**Date:** March 2026

---

## Table of Contents

1. [Project Overview and Architecture](#1-project-overview)
2. [Phase 1: Hardware and Hypervisor Setup](#2-phase-1-hardware)
3. [Phase 2: Windows Server 2025 — AD/DNS/DHCP/CA](#3-phase-2-windows)
4. [Phase 3: Infoblox NIOS DDI Integration](#4-phase-3-infoblox)
5. [Phase 4: EVPN-VXLAN Fabric Design](#5-phase-4-fabric-design)
6. [Phase 5: Ansible AVD Automation](#6-phase-5-ansible-avd)
7. [Phase 6: Fabric Deployment and Verification](#7-phase-6-deployment)
8. [Phase 7: CloudVision Portal Setup and Onboarding](#8-phase-7-cvp)
9. [Phase 8: Three-Server Consolidation and Migration](#9-phase-8-consolidation)
10. [Phase 9: Monitoring Stack — Prometheus, Grafana, SNMP](#10-phase-9-monitoring)
11. [Phase 10: AIOps Self-Healing Pipeline](#11-phase-10-aiops)
12. [Challenges Encountered and Resolutions](#12-challenges)
13. [Final Service Map and Verification](#13-final-map)

---

## 1. Project Overview and Architecture <a name="1-project-overview"></a>

### 1.1 Goal

Build a production-grade enterprise data center lab that demonstrates end-to-end network automation, from initial fabric deployment through closed-loop AI-driven remediation. The lab mirrors real-world architectures scaled to home lab hardware.

### 1.2 Physical Infrastructure

| Component | Details |
|-----------|---------|
| Server | Cisco UCS C220 M5 |
| Management | CIMC at 192.168.86.211 (dedicated port) |
| Hypervisor | Proxmox VE 9.1 |
| Storage | ZFS RAIDZ across 6× 1TB SAS drives (~4.55TB usable) |
| Network | 10G/1G link negotiation, vmbr0 bridge |

### 1.3 Virtual Infrastructure on Proxmox

| VM ID | Name | Purpose | Disk |
|-------|------|---------|------|
| 100 | EVE-NG | Network lab platform | 200GB |
| 101 | WindowsServer | AD/DNS/DHCP/CA | — |
| 102 | CML | Cisco Modeling Labs | — |

### 1.4 Architecture Diagram

```
┌──────────────────────────────────────────────────────────┐
│                 Cisco UCS C220 M5                        │
│                 Proxmox VE 9.1                           │
│  ┌────────────────────────────────────────────────────┐  │
│  │              EVE-NG (VM 100)                       │  │
│  │                                                    │  │
│  │   ┌──────────┐    ┌──────────┐                     │  │
│  │   │ SPINE-01 │    │ SPINE-02 │   BGP Route         │  │
│  │   │ AS 65000 │    │ AS 65000 │   Reflectors        │  │
│  │   │  .100    │    │  .101    │                     │  │
│  │   └────┬─────┘    └────┬─────┘                     │  │
│  │        │    ╲        ╱  │    eBGP Underlay          │  │
│  │        │     ╲      ╱   │    EVPN Overlay           │  │
│  │   ┌────┴─────┐    ┌────┴─────┐                     │  │
│  │   │ LEAF-01  │    │ LEAF-02  │   VXLAN VTEPs       │  │
│  │   │ AS 65101 │    │ AS 65102 │                     │  │
│  │   │  .110    │    │  .111    │                     │  │
│  │   └──────────┘    └──────────┘                     │  │
│  │                                                    │  │
│  │   ┌──────┐  ┌──────────┐  ┌──────────┐            │  │
│  │   │ CVP  │  │Lab-Server│  │ Infoblox │            │  │
│  │   │ .55  │  │.60/.70   │  │   .51    │            │  │
│  │   │      │  │  /.90    │  │          │            │  │
│  │   └──────┘  └──────────┘  └──────────┘            │  │
│  └────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────┐  │
│  │       Windows Server 2025 (VM 101)                 │  │
│  │   AD DS │ DNS │ DHCP │ CA │ 192.168.86.10         │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

---

## 2. Phase 1: Hardware and Hypervisor Setup <a name="2-phase-1-hardware"></a>

### 2.1 CIMC Configuration

The Cisco UCS C220 M5 was configured with a dedicated management port at 192.168.86.211. CIMC provides out-of-band management, KVM console, and virtual media for OS installation.

### 2.2 Proxmox Installation

Proxmox VE 9.1 was installed via KVM virtual media through CIMC. Key configuration steps included resolving interface naming (10G vs 1G link negotiation) and creating the network bridge `vmbr0` for VM connectivity.

### 2.3 ZFS Storage Pool

A RAIDZ pool named `vmstore` was created across six 1TB SAS drives, yielding approximately 4.55TB of usable storage with single-disk redundancy.

### 2.4 EVE-NG Deployment

EVE-NG was deployed as VM 100 on Proxmox with nested KVM virtualization enabled. This allows EVE-NG to run its own QEMU instances (Arista vEOS, CVP, Infoblox, Ubuntu nodes) inside the Proxmox VM. The EVE-NG VM was allocated 32GB RAM and a 200GB disk.

---

## 3. Phase 2: Windows Server 2025 — AD/DNS/DHCP/CA <a name="3-phase-2-windows"></a>

### 3.1 Server Setup

Windows Server 2025 was deployed on Proxmox as VM 101 and configured as the domain controller for the lab environment.

**Configuration:**
- Hostname: `DC01`
- IP: `192.168.86.10/24`
- Domain: `eve-ng-lab.local`
- NetBIOS: `EVENGLAB`

### 3.2 Active Directory Domain Services

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-WindowsFeature DNS -IncludeManagementTools

Install-ADDSForest `
  -DomainName "eve-ng-lab.local" `
  -DomainNetbiosName "EVENGLAB" `
  -ForestMode "WinThreshold" `
  -DomainMode "WinThreshold" `
  -InstallDns:$true `
  -SafeModeAdministratorPassword (ConvertTo-SecureString "YourDSRMPassword123!" -AsPlainText -Force) `
  -Force:$true
```

### 3.3 DNS Configuration

Forward zone `eve-ng-lab.local` and reverse zone `86.168.192.in-addr.arpa` were created. DNS forwarders point to 8.8.8.8 and 8.8.4.4 for external resolution.

**DNS Records Added:**

| Hostname | IP Address | Purpose |
|----------|-----------|---------|
| cvp | 192.168.86.55 | CloudVision Portal |
| infoblox | 192.168.86.51 | Infoblox NIOS |
| netbox | 192.168.86.60 | NetBox DCIM |
| ansible | 192.168.86.70 | Ansible AVD |
| n8n | 192.168.86.90 | Workflow Automation |
| grafana | 192.168.86.60 | Dashboards |
| eve-ng | 192.168.86.213 | Lab Platform |
| spine-01 | 192.168.86.100 | eBGP Spine |
| spine-02 | 192.168.86.101 | eBGP Spine |
| leaf-01 | 192.168.86.110 | L3 Leaf |
| leaf-02 | 192.168.86.111 | L3 Leaf |

### 3.4 DHCP Configuration

DHCP scope 192.168.86.100-200 was created with exclusion range .1-.50 for static infrastructure devices. Options include router (.1), DNS server (.10), and domain name (eve-ng-lab.local).

### 3.5 Enterprise Root CA

An Enterprise Root CA named `eve-ng-lab-ROOT-CA` was installed with SHA256, 4096-bit keys, and 10-year validity. Web Enrollment was also enabled for certificate requests.

```powershell
Install-AdcsCertificationAuthority `
  -CAType EnterpriseRootCA `
  -CACommonName "eve-ng-lab-ROOT-CA" `
  -KeyLength 4096 `
  -HashAlgorithmName SHA256 `
  -ValidityPeriod Years `
  -ValidityPeriodUnits 10 `
  -Force
```

The root CA certificate was exported in both DER and Base64 formats:

```powershell
$cert = (Get-ChildItem -Path Cert:\LocalMachine\CA | Where-Object {
  $_.Subject -like "*eve-ng-lab-ROOT-CA*"
})[0]
Export-Certificate -Cert $cert -FilePath C:\RootCA.cer -Type CERT
certutil -encode "C:\RootCA.cer" "C:\RootCA-Base64.cer"
```

---

## 4. Phase 3: Infoblox NIOS DDI Integration <a name="4-phase-3-infoblox"></a>

### 4.1 Deployment

Infoblox NIOS 8.2.4 was deployed as EVE-NG node 6 with a qcow2 image. A critical discovery was made during deployment: LAN1 maps to `net1/vunl0_6_1` (not `vunl0_6_0`), requiring the bridge fix:

```bash
brctl addif pnet0 vunl0_6_1
```

### 4.2 Configuration

- Grid setup: Standalone, hostname `infoblox.eve-ng-lab.local`
- IP: 192.168.86.51
- Licenses: DNSone with Grid (60-day evaluation)
- DNS zones: `eve-ng-lab.local` (authoritative), `86.168.192.in-addr.arpa` (reverse)

### 4.3 Architecture Decision

Windows DC01 serves as the primary DNS/DHCP authority, while Infoblox provides IPAM visibility and DDI management overlay. This mirrors enterprise architectures where Infoblox manages Microsoft DNS/DHCP without replacing it.

---

## 5. Phase 4: EVPN-VXLAN Fabric Design <a name="5-phase-4-fabric-design"></a>

### 5.1 Fabric Topology

The fabric follows a spine-leaf architecture with eBGP underlay and EVPN overlay:

| Device | Role | BGP AS | Loopback0 | VTEP (Lo1) | Mgmt IP |
|--------|------|--------|-----------|------------|---------|
| SPINE-01 | Spine/RR | 65000 | 10.255.0.1/32 | — | 192.168.86.100 |
| SPINE-02 | Spine/RR | 65000 | 10.255.0.2/32 | — | 192.168.86.101 |
| LEAF-01 | L3 Leaf | 65101 | 10.255.0.11/32 | 10.255.1.11/32 | 192.168.86.110 |
| LEAF-02 | L3 Leaf | 65102 | 10.255.0.12/32 | 10.255.1.12/32 | 192.168.86.111 |

### 5.2 P2P Underlay Links

All point-to-point links use /31 addressing from the 10.10.0.0/24 pool:

| Link | IP A | IP B |
|------|------|------|
| LEAF-01:Eth1 ↔ SPINE-01:Eth1 | 10.10.0.0/31 | 10.10.0.1/31 |
| LEAF-01:Eth2 ↔ SPINE-02:Eth1 | 10.10.0.2/31 | 10.10.0.3/31 |
| LEAF-02:Eth1 ↔ SPINE-01:Eth2 | 10.10.0.4/31 | 10.10.0.5/31 |
| LEAF-02:Eth2 ↔ SPINE-02:Eth2 | 10.10.0.6/31 | 10.10.0.7/31 |

### 5.3 Design Principles

- **eBGP underlay** for simplicity and scalability (no OSPF/ISIS)
- **EVPN overlay** with spines as Route Reflectors
- **BFD** for sub-second failure detection (300ms intervals, multiplier 3)
- **VXLAN** with Loopback1 as VTEP source interface
- **MTU 9214** on all fabric links for jumbo frame support

### 5.4 Switch Bootstrap Configuration

Each switch was bootstrapped via console with management access before AVD automation:

```
enable
configure
hostname <HOSTNAME>
username admin privilege 15 role network-admin secret admin
enable password admin
service routing protocols model multi-agent
management api http-commands
   protocol https
   no shutdown
vrf instance MGMT
ip routing vrf MGMT
interface Management1
   vrf MGMT
   ip address 192.168.86.X/24
ip route vrf MGMT 0.0.0.0/0 192.168.86.1
ip name-server vrf MGMT 192.168.86.10
management api http-commands
   vrf MGMT
      no shutdown
write memory
```

---

## 6. Phase 5: Ansible AVD Automation <a name="6-phase-5-ansible-avd"></a>

### 6.1 Environment Setup

The Ansible AVD server runs on the consolidated lab-server at 192.168.86.70. A Python virtual environment (`avd-venv`) contains all required packages:

```bash
python3 -m venv avd-venv
source avd-venv/bin/activate
pip install pyavd==6.0.1 anta>=1.7.0 netaddr jsonschema treelib netutils pyeapi pynetbox cvprac
ansible-galaxy collection install arista.avd arista.eos
```

### 6.2 Inventory Structure

```yaml
# inventory.yml
all:
  vars:
    ansible_network_os: arista.eos.eos
    ansible_connection: ansible.netcommon.httpapi
    ansible_httpapi_use_ssl: true
    ansible_httpapi_validate_certs: false
    ansible_user: admin
    ansible_password: admin
    ansible_become: true
    ansible_become_method: enable
    ansible_become_password: admin
  children:
    DC1_FABRIC:
      children:
        SPINES:
          hosts:
            SPINE-01: {ansible_host: 192.168.86.100}
            SPINE-02: {ansible_host: 192.168.86.101}
        LEAFS:
          hosts:
            LEAF-01: {ansible_host: 192.168.86.110}
            LEAF-02: {ansible_host: 192.168.86.111}
```

### 6.3 AVD Group Variables

The fabric design is expressed declaratively in YAML files:

**SPINES.yml** — Defines spine defaults (BGP AS 65000, loopback pool)
**LEAFS.yml** — Defines leaf defaults (loopback pools, VTEP pool, uplink mapping)
**FABRIC.yml** — Defines fabric-wide settings (eBGP underlay, EVPN overlay, P2P links)
**DC1_FABRIC.yml** — Defines management interface and gateway

### 6.4 NetBox Integration Scripts

Custom Python scripts auto-generate AVD inventory from NetBox:
- `netbox_to_avd_inventory.py` — Generates host_vars and group_vars from NetBox devices
- `netbox_to_avd_p2p.py` — Generates P2P link definitions from NetBox cables
- `netbox_underlay_neighbors.py` — Generates interface descriptions from cable connections

---

## 7. Phase 6: Fabric Deployment and Verification <a name="7-phase-6-deployment"></a>

### 7.1 Build, Push, Verify Pipeline

The entire fabric is deployed with three Ansible playbooks:

```bash
cd /root && source avd-venv/bin/activate

# Step 1: Generate structured configs and CLI configs from AVD data model
ansible-playbook build.yml -i inventory.yml

# Step 2: Push configs to all switches (merge mode)
ansible-playbook push_merge.yml -i inventory.yml

# Step 3: Verify fabric health (BGP sessions, EVPN, underlay)
ansible-playbook verify_fabric.yml -i inventory.yml
```

### 7.2 Generated Configurations

AVD generates complete EOS configurations including:
- Interface configurations with MTU 9214 and /31 addressing
- BGP configuration with peer groups (IPv4-UNDERLAY-PEERS, EVPN-OVERLAY-PEERS)
- Route maps and prefix lists for connected route redistribution
- VXLAN interface configuration on leafs
- BFD for fast convergence

### 7.3 Verification Results

All playbooks completed successfully:
- Build: PASSED (4 structured configs + 4 CLI configs generated)
- Push: PASSED (configs merged to all 4 switches)
- Verify: PASSED (all BGP sessions Established)

---

## 8. Phase 7: CloudVision Portal Setup and Onboarding <a name="8-phase-7-cvp"></a>

### 8.1 CVP Installation

CVP was deployed as an EVE-NG node with the following configuration:
- Hostname: `cvp.eve-ng-lab.local`
- IP: 192.168.86.55
- Installation: Singlenode mode
- Gateway: 192.168.86.1
- DNS: 192.168.86.10

The installation took approximately 60 minutes to deploy all Kubernetes pods (244 components, 230 running, 14 disabled).

### 8.2 Service Account and Token

A service account `terminattr` was created in CVP with `network-admin` role:
- Settings → Access Control → Service Accounts → Add
- Generated token with long expiry (2028)

### 8.3 The TerminAttr Onboarding Challenge

This was the most complex part of the project, taking extensive troubleshooting to resolve. The core issue: EOS 4.35.1F with TerminAttr v1.42.0 enforces TLS certificate enrollment that fails against CVP's internal CA.

#### Attempt 1: Standard `-cvaddr` with token auth
```
exec /usr/bin/TerminAttr -cvaddr=192.168.86.55:9910 -cvauth=token,/tmp/token -cvvrf=MGMT
```
**Result:** Enrollment failed — "certificate has no type"

#### Attempt 2: Adding `-disableaaa`
**Result:** Still tried enrollment, got 404/502 from CVP nginx

#### Attempt 3: Legacy `-ingestgrpcurl` 
```
exec /usr/bin/TerminAttr -ingestgrpcurl=192.168.86.55:9910 -ingestauth=token,/tmp/token -ingestvrf=MGMT
```
**Result:** No outbound connections — the flag silently does nothing with `-ingestvrf`

#### Attempt 4: Self-signed client certificates
Generated certificates with device serial number as CN, but got "certificate has no type" — Arista requires specific certificate extensions.

#### Attempt 5: `-cvserverCA` flag
```
exec /usr/bin/TerminAttr -cvaddr=192.168.86.55:9910 -cvserverCA=/persist/secure/ssl/certs/cvpCA.pem
```
**Result:** Flag exists but does nothing ("provided for compatibility")

#### Attempt 6: Certs auth with CVP CA
```
exec /usr/bin/TerminAttr -cvaddr=192.168.86.55:9910 -cvauth=certs,/cert.crt,/cert.key,/cvpCA.pem -cvvrf=MGMT
```
**Result:** Local Sysdb sync only, no CVP connection

### 8.4 The Solution

The root cause was that Management1 was in the MGMT VRF, and none of the VRF flags (`-cvvrf`, `-ingestvrf`) worked properly in this EOS/TerminAttr version combination.

**Fix: Move Management1 to the default VRF**

```
configure
interface Management1
   no vrf MGMT
   ip address 192.168.86.X/24
ip route 0.0.0.0/0 192.168.86.1
management api http-commands
   no shutdown
   vrf default
      no shutdown
daemon TerminAttr
   exec /usr/bin/TerminAttr -cvaddr=192.168.86.55:9910 -cvauth=token,/tmp/token -smashexcludes=ale,flexCounter,hardware,kni,pulse,strata -ingestexclude=/Sysdb/cell/1/agent,/Sysdb/cell/2/agent -taillogs -disableaaa
   no shutdown
end
write memory
```

Additionally, NTP clock sync was critical — CVP rejects messages with future timestamps. After setting clocks correctly via `clock set` and configuring `ntp server 192.168.86.10`, all four switches appeared in CVP inventory as **Active**.

### 8.5 CVP Device Onboarding via API

The CVP web UI device onboarding kept failing with "Error received from device" due to credential mismatches. The `cvprac` Python library was used to programmatically add devices:

```python
from cvprac.cvp_client import CvpClient

client = CvpClient()
client.connect(['192.168.86.55'], 'cvpadmin', 'Test1234')

for ip, name in [('192.168.86.100','SPINE-01'), ('192.168.86.101','SPINE-02'),
                 ('192.168.86.110','LEAF-01'), ('192.168.86.111','LEAF-02')]:
    client.api.retry_add_to_inventory(ip, name, username='admin', password='admin')
```

### 8.6 Key Lessons from CVP Onboarding

1. TerminAttr v1.42.0 on EOS 4.35.1F requires Management1 in default VRF for reliable streaming
2. `-cvserverCA`, `-ingestgrpcurl` with `-ingestvrf`, and certificate enrollment paths all have issues
3. CVP's API for device onboarding requires `{"hosts": ["ip"]}` format (array of strings)
4. NTP sync between switches and CVP is mandatory — even a few minutes of clock drift causes "Unacceptable future timestamp" errors
5. eAPI must be enabled on both MGMT and default VRFs for CVP to manage devices

---

## 9. Phase 8: Three-Server Consolidation and Migration <a name="9-phase-8-consolidation"></a>

### 9.1 Original Architecture (3 Servers)

| Server | IP | Services |
|--------|-----|----------|
| Ubuntu 1 | 192.168.86.60 | NetBox, Grafana, Prometheus, Alertmanager |
| Ubuntu 2 | 192.168.86.70 | Ansible AVD, OPA/Rego |
| Ubuntu 3 | 192.168.86.90 | n8n, MCP |

### 9.2 Backup Strategy

Each server was backed up with a comprehensive script that captured:
- System configs (`/etc/hostname`, `/etc/netplan/`, etc.)
- All project files (`/opt/`, `/root/`)
- Docker images list
- Docker volumes (exported via Alpine container)
- Pip requirements (for Python environments)

```bash
# Example: Ubuntu 3 (n8n) backup
mkdir -p /backup
tar czf /backup/opt-full.tar.gz /opt/
docker volume ls -q > /backup/docker-volumes.txt
for vol in $(docker volume ls -q); do
  docker run --rm -v ${vol}:/data -v /backup:/backup alpine \
    tar czf /backup/vol-${vol}.tar.gz -C /data .
done
tar czf /tmp/ubuntu3-full.tar.gz -C /backup .
```

### 9.3 Consolidated Architecture (1 Server)

All three servers were merged into a single Ubuntu 22.04 node on EVE-NG using multiple IP aliases:

```yaml
# /etc/netplan/01-static.yaml
network:
  version: 2
  ethernets:
    ens3:
      addresses:
        - 192.168.86.60/24
        - 192.168.86.70/24
        - 192.168.86.90/24
      routes:
        - to: default
          via: 192.168.86.1
      nameservers:
        addresses: [192.168.86.10]
        search: [eve-ng-lab.local]
```

### 9.4 Service Restoration

Services were restored by extracting backup archives and starting Docker compose stacks. A critical finding during n8n restoration: the original n8n used bind-mounted PostgreSQL at `/opt/lab/n8n/postgres/` (not a Docker volume), so the old compose file at `/opt/lab/n8n/docker-compose.yml` had to be used instead of the new `agentic` compose.

### 9.5 Disk Space Challenge

The consolidated server's LVM was only 19GB of a 40GB disk. After running out of space (100% usage), the LVM was extended:

```bash
lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

---

## 10. Phase 9: Monitoring Stack <a name="10-phase-9-monitoring"></a>

### 10.1 Stack Components

| Service | Port | Image |
|---------|------|-------|
| Prometheus | 9090 | prom/prometheus:latest |
| Grafana | 3000 | grafana/grafana:latest |
| Alertmanager | 9093 | prom/alertmanager:latest |
| SNMP Exporter | 9116 | prom/snmp-exporter:v0.21.0 |
| Blackbox Exporter | 9115 | prom/blackbox-exporter:latest |
| Node Exporter | 9100 | prom/node-exporter:latest |

### 10.2 SNMP Configuration

SNMP was enabled on all Arista switches for Prometheus scraping:

```bash
ansible -i inventory.yml all -m arista.eos.eos_config \
  -a "lines='snmp-server community public ro'"
```

The SNMP exporter uses a custom `snmp.yml` with BGP4-MIB for collecting:
- `bgpPeerState` — BGP peer state (6=established)
- `bgpPeerRemoteAs` — Remote AS number
- `bgpPeerFsmEstablishedTime` — Session uptime
- `bgpPeerInUpdates` — Update messages received
- `bgpPeerFsmEstablishedTransitions` — Flap count

### 10.3 Grafana Dashboards

**Critical Infrastructure Dashboard:** Shows service status (UP/DOWN), host resources (CPU, RAM, disk), Docker container health, and interface traffic for leafs and spines.

**EVPN/VXLAN Health Dashboard:** Shows BGP sessions per leaf, VXLAN tunnels active, SNMP device reachability, BGP prefixes received, interface traffic, EVPN message rates, interface error rates, and a BGP neighbor summary table.

### 10.4 Blackbox Monitoring

Blackbox exporter monitors HTTP endpoints of all services:

```yaml
# blackbox_targets.yml
- targets:
    - http://192.168.86.60:8000   # NetBox
    - http://192.168.86.60:3000   # Grafana
    - http://192.168.86.60:9090   # Prometheus
    - http://192.168.86.70:8181   # OPA
    - http://192.168.86.90:5678   # n8n
    - http://192.168.86.90:8088   # MCP
```

---

## 11. Phase 10: AIOps Self-Healing Pipeline <a name="11-phase-10-aiops"></a>

### 11.1 Pipeline Architecture

The closed-loop self-healing pipeline follows this flow:

```
Grafana Alert → Alertmanager → n8n Webhook → Claude AI Analysis
    → OPA/Rego Policy Check → Slack Approval → Ansible Remediation
    → Verification → Prometheus Metrics Update
```

### 11.2 Component Details

**Grafana Alerts:** Configured on Prometheus metrics for BGP session state, interface errors, and SNMP reachability. Alerts fire when thresholds are breached.

**Alertmanager:** Routes alerts to n8n via webhook. Configuration includes grouping by alertname and instance with appropriate wait intervals.

**n8n Workflow Automation:** The core orchestration engine. The workflow:
1. Receives alert webhook from Alertmanager
2. Parses alert payload (device, interface, BGP neighbor, severity)
3. Sends alert context to Claude AI for analysis
4. Claude returns a remediation recommendation with risk assessment
5. OPA/Rego validates the recommendation against policy
6. Slack message sent for human approval (or auto-execute in production)
7. On approval, triggers Ansible playbook for remediation
8. Post-remediation verification confirms the fix

**Claude AI Integration:** Used for intelligent alert analysis. Claude receives the alert context, device configuration, and network state, then returns structured remediation steps with risk scoring.

**OPA/Rego Policy Engine:** Enforces guardrails on AI recommendations:
- Only approved remediation types are allowed
- Maximum number of changes per window
- Rollback requirements for high-risk changes
- Device-specific restrictions

**MCP (Model Context Protocol) Server:** Bridges Claude AI to live network state. Exposes Prometheus metrics and OPA policies via REST API, enabling Claude to query real-time network health during analysis.

**Slack Approval:** Sends formatted messages to a Slack channel with approve/reject buttons. Initially used webhook pre-fetching which caused failures — resolved by switching to auto-execute mode for the lab environment.

**Ansible Remediation:** Executes EOS commands on the target device:
- BGP session recovery: `clear bgp neighbor X`
- Interface bounce: `shutdown` / `no shutdown`
- EVPN reset: Clear EVPN routes

### 11.3 n8n Environment Variables

```env
OPENAI_API_KEY=sk-proj-...
OPENAI_MODEL=gpt-4o-mini
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...
N8N_HOST=192.168.86.90
N8N_PORT=5678
WEBHOOK_URL=http://192.168.86.90:5678/
N8N_SECURE_COOKIE=false
N8N_METRICS=true
N8N_API_KEY=aiops-lab-key-2026
```

### 11.4 Pipeline Challenges and Resolutions

**Challenge 1: Slack Webhook Pre-fetching**
Slack pre-fetches webhook URLs, which triggered false approval confirmations. 
**Resolution:** Switched to auto-execute mode for lab environment; added unique tokens to prevent replay attacks.

**Challenge 2: n8n Secure Cookie Error**
n8n v1.x+ requires HTTPS by default, showing a cookie error on HTTP.
**Resolution:** Set `N8N_SECURE_COOKIE=false` in environment variables.

**Challenge 3: Claude AI Response Parsing**
Unstructured AI responses made it difficult to extract actionable remediation steps.
**Resolution:** Used structured prompts requesting JSON output with specific fields (action, device, commands, risk_level).

**Challenge 4: OPA Policy Enforcement**
Initial OPA rules were too restrictive, blocking valid remediations.
**Resolution:** Iteratively refined Rego policies to balance safety with operational effectiveness.

---

## 12. Challenges Encountered and Resolutions <a name="12-challenges"></a>

### 12.1 Infrastructure Challenges

| Challenge | Root Cause | Resolution |
|-----------|-----------|------------|
| Infoblox LAN1 mapping | EVE-NG maps e1 (not e0) to LAN1 | `brctl addif pnet0 vunl0_6_1` |
| CVP installation timeout | K8s pods need 60+ minutes | Patient wait + `cvpi status all` monitoring |
| Proxmox nested virtualization | Not enabled by default | Enable in VM CPU settings |
| ZFS pool creation | Drive naming on UCS | Identified correct SAS drive paths |

### 12.2 TerminAttr/CVP Challenges

| Challenge | Root Cause | Resolution |
|-----------|-----------|------------|
| "Certificate has no type" | Self-signed certs lack Arista extensions | Move Management1 to default VRF |
| "Network is unreachable" | TerminAttr can't reach CVP via MGMT VRF | Remove VRF flags, use default VRF |
| `-cvserverCA` does nothing | Flag is no-op in v1.42.0 | Not needed after VRF fix |
| `-ingestgrpcurl` no connections | Legacy ingest broken with VRF | Use `-cvaddr` instead |
| "Token not found" | Empty /tmp/token file | Re-paste token from CVP UI |
| "Unacceptable future timestamp" | Switch clocks set to wrong date | `clock set` + NTP configuration |
| "Error received from device" | eAPI not on MGMT VRF | Enable eAPI on both VRFs |
| CVP 401 auth errors | Wrong credentials | Match cvpadmin password on switches and CVP |

### 12.3 Migration Challenges

| Challenge | Root Cause | Resolution |
|-----------|-----------|------------|
| Disk 100% full | 19GB LVM on 40GB disk | `lvextend` + `resize2fs` |
| n8n workflows lost | Docker volume removed during cleanup | Restored from backup volume |
| n8n cookie error | HTTPS required by default | `N8N_SECURE_COOKIE=false` |
| Duplicate Grafana containers | Old + new monitoring stacks | Stop old stack, keep new |
| SNMP 500 errors | SNMP not configured on switches | Ansible playbook to enable SNMP |

---

## 13. Final Service Map and Verification <a name="13-final-map"></a>

### 13.1 Complete Service Map

| Service | IP Address | Port | Status |
|---------|-----------|------|--------|
| Windows DC01 (AD/DNS/DHCP/CA) | 192.168.86.10 | — | Running |
| Infoblox NIOS (DDI/IPAM) | 192.168.86.51 | 443 | Running |
| CloudVision Portal | 192.168.86.55 | 443/9910 | 4 switches Active |
| NetBox (DCIM/IPAM) | 192.168.86.60 | 8000 | Running |
| Grafana (Dashboards) | 192.168.86.60 | 3000 | Running |
| Prometheus (Metrics) | 192.168.86.60 | 9090 | Running |
| Alertmanager | 192.168.86.60 | 9093 | Running |
| Ansible AVD | 192.168.86.70 | SSH | Ready |
| OPA/Rego (Policy) | 192.168.86.70 | 8181 | Running |
| n8n (Workflows) | 192.168.86.90 | 5678 | Running |
| MCP (AI Bridge) | 192.168.86.90 | 8088 | Running |
| SPINE-01 | 192.168.86.100 | 443 | Streaming to CVP |
| SPINE-02 | 192.168.86.101 | 443 | Streaming to CVP |
| LEAF-01 | 192.168.86.110 | 443 | Streaming to CVP |
| LEAF-02 | 192.168.86.111 | 443 | Streaming to CVP |

### 13.2 Fabric Verification

```
SPINE-01# show bgp summary
  Neighbor    AS    State   PfxRcd
  10.10.0.41  65101 Estab   5
  10.10.0.45  65102 Estab   5
  10.255.0.11 65101 Estab   0  (EVPN)
  10.255.0.12 65102 Estab   0  (EVPN)
```

### 13.3 One-Line Fabric Deploy

```bash
cd /root && source avd-venv/bin/activate && \
ansible-playbook build.yml -i inventory.yml && \
ansible-playbook push_merge.yml -i inventory.yml && \
ansible-playbook verify_fabric.yml -i inventory.yml
```

### 13.4 Technologies Used

**Network:** Arista vEOS 4.35.1F, EVPN-VXLAN, eBGP, BFD, VXLAN
**Automation:** Ansible AVD 6.0.1, Python, pyavd, pynetbox, cvprac
**Monitoring:** Prometheus, Grafana 12.4, Alertmanager, SNMP Exporter
**AIOps:** Claude AI, n8n, OPA/Rego, MCP, Slack
**Infrastructure:** Proxmox VE 9.1, EVE-NG, Docker, ZFS RAIDZ
**Identity:** Windows Server 2025 AD DS, DNS, DHCP, AD CS
**DDI:** Infoblox NIOS 8.2.4
**Telemetry:** Arista CloudVision Portal, TerminAttr v1.42.0
**Source of Truth:** NetBox

---

*This document represents the complete technical record of building an enterprise-grade AIOps network lab from bare metal to closed-loop self-healing operations.*
