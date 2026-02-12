# VCF 9.0.2 Deployment Guide - Part 1: Architecture & Planning

## Introduction: The "Consolidated" Architecture

For this home lab build, we are deploying a **Consolidated Architecture** on a **3-Node Cluster**.

In a Consolidated deployment, the Management Domain functions as a hybrid environment. It runs the VCF management stack (vCenter, NSX, SDDC Manager) alongside your user workloads (VMs, Kubernetes). This eliminates the need for a separate VI Workload Domain, significantly reducing hardware requirements.

**Key VCF 9.0.2 Feature:** Support for 1GbE NICs on the Management Network. You no longer need 10GbE for the management interface, though 10GbE is used here for Data/Storage.

**Target End State:**

- **Identity Management** (Active Directory Authentication & Custom Certificates).
- **Offline Lifecycle Management** (Offline Depot).
- Full BGP dynamic routing.
- vSAN ESA with Memory Tiering.
- **NFS Storage for Bulk Data** (Additional Datastores).
- **VCF Operations for Networks** (Network Insight).
- **Supervisor Cluster** (vSphere with Tanzu/VKS).
- **VCF Automation** configured for VPCs and Tenancy.

## Phase 1: Architecture & Planning (The "Paperwork")

*Do not skip this. 90% of VCF failures happen because of missing DNS records or VLAN mismatches here.*

### 1.1 Bill of Materials (BOM)

- **Software:**
  - VMware Cloud Foundation 9.0.2 Installer OVA.
  - ESXi 9.0 ISO (Ensure hosts are on the exact supported build).
  - **VCF Bundle Transfer Utility (OBTU):** Required for downloading bundles on the internet-connected workstation.
- **Hardware (Your Lab Spec):**
  - **VCF Cluster (Target):** 3x Physical Hosts (Group B).
    - `pgesxa1.pgnet.io` (10.200.1.220)
    - `pgesxa2.pgnet.io` (10.200.1.222)
    - `pgesxa3.pgnet.io` (10.200.1.224)
  - **Supporting Cluster:** A separate, existing cluster.
    - **Role:** Hosts the "Day 0" and "Day 1" infrastructure services (DNS/NTP, Router/Firewall, VCF Installer).
    - **Requirement:** Must have network reachability to VLAN 201 (Mgmt).
  - **Depot Workstation:** A separate physical or virtual machine.
    - **Role:** Has **Internet Access** to Broadcom Support.
    - **Purpose:** Runs the Bundle Transfer Utility to populate the repository "Offline" for speed and control.
  - **NFS NAS Appliance:**
    - **Hostname:** `pgnas.pgnet.io`
    - **Role:** Provides additional datastores for bulk storage (ISO files, backup targets, heavy content libraries) to offload the vSAN ESA tier.
    - **Connection:** Connected via 10GbE to VLAN 204.
  - **CPU:** Minimum 16 Cores per VCF host.
  - **RAM:** 128GB minimum per VCF host (Augmented by **Memory Tiering**).
  - **Storage:** **vSAN ESA (Express Storage Architecture)**.
    - *Requirement:* vSAN ESA compliant NVMe drives for both storage pool and system boot.
  - **Network:** Physical Uplinks configured for Trunking.
- **Lab Specific Workarounds:**
  - **vSAN ESA Mock VIB:** Required to bypass strict Hardware Compatibility List (HCL) checks on consumer NVMe drives. (Reference: [William Lam - vSAN ESA Hardware Mock VIB](https://williamlam.com/2025/02/vsan-esa-hardware-mock-vib-for-physical-esxi-deployment-for-vmware-cloud-foundation-vcf.html))
  - **AMD Ryzen Configuration:** Specific kernel parameters required to allow sensitive workloads (like NSX Edge Nodes and Memory Tiering) to function correctly on consumer AMD CPUs. (Reference: [William Lam - AMD Ryzen Workarounds](https://williamlam.com/2025/06/nvme-tiering-with-amd-ryzen-cpu-workaround-for-vcf-9-0.html))

### 1.2 VLAN & Subnet Schedule

The following schedule matches your specific lab network (UniFi/PGGB-UDM).

| Traffic Type      | VLAN ID | CIDR            | Purpose                                       |
| ----------------- | ------- | --------------- | --------------------------------------------- |
| **VM Management** | **201** | 10.200.1.0/24   | ESXi Mgmt, vCenter, SDDC Mgr, NSX Mgrs        |
| **vMotion**       | **202** | 10.200.2.0/24   | Host-to-Host live migration                   |
| **vSAN**          | **203** | 10.200.3.0/24   | Storage traffic (East-West)                   |
| **NFS Storage**   | **204** | 10.200.4.0/24   | NAS Appliance / Bulk Data Datastores          |
| **Host TEP**      | **205** | 10.200.5.0/24   | NSX Host Overlay (Geneve) - *No DNS required* |
| **Edge TEP**      | **206** | 10.200.6.0/24   | NSX Edge Overlay - *No DNS required*          |
| **VM Net**        | **207** | 10.200.7.0/24   | General VM Workload Traffic                   |
| **VKS Mgmt**      | **208** | 10.200.8.0/24   | Tanzu / K8s Management                        |
| **K8 Workload**   | **209** | 10.200.9.0/24   | Tanzu / K8s Workload                          |
| **RouterNet 1**   | **250** | 10.200.250.0/24 | BGP Uplink 1 (Peer: 10.200.250.1)             |
| **RouterNet 2**   | **251** | 10.200.251.0/24 | BGP Uplink 2 (Peer: 10.200.251.1)             |

### 1.3 DNS & Identity Strategy

Your **Infrastructure Server (10.200.1.240)** hosted on the **Supporting Cluster** is the single source of truth for core services.

- **DNS Server:** `10.200.1.240` (Running **BIND 9**).
  - **Role:** Authoritative for the `pgnet.io` infrastructure zone.
  - **Forwarding:** Forwards authentication requests (SRV records, Kerberos) to the Active Directory domain controllers (`pggb.local`).
- **NTP Server:** `10.200.1.240` (Running NTPD/Chrony).
  - **Role:** Point all VCF hosts and appliances to `10.200.1.240` to ensure time sync matches the DNS/Identity environment.
- **Authentication Domain:** `pggb.local` (Active Directory).
  - **Note:** While infrastructure lives in `pgnet.io`, all user authentication will occur against `pggb.local`.

**DNS Validation Table**

| Hostname (FQDN)              | VCF Role                         | IP Address   | PTR Status      |
| ---------------------------- | -------------------------------- | ------------ | --------------- |
| **sddc.pgnet.io**            | SDDC Manager                     | 10.200.1.27  | ✅ Valid         |
| **fleet.pgnet.io**           | Fleet Manager                    | 10.200.1.10  | ✅ Valid         |
| **vc.pgnet.io**              | Management vCenter               | 10.200.1.11  | ✅ Valid         |
| **nsx.pgnet.io**             | NSX Manager VIP                  | 10.200.1.15  | ✅ Valid         |
| **nsxm1.pgnet.io**           | NSX Manager Node 1               | 10.200.1.24  | ✅ Valid         |
| **installer.pgnet.io**       | VCF Installer Appliance          | 10.200.1.30  | ✅ Valid         |
| **pgesxa1.pgnet.io**         | ESXi Host 1 (Target)             | 10.200.1.220 | ✅ Valid         |
| **pgesxa2.pgnet.io**         | ESXi Host 2 (Target)             | 10.200.1.222 | ✅ Valid         |
| **pgesxa3.pgnet.io**         | ESXi Host 3 (Target)             | 10.200.1.224 | ✅ Valid         |
| **pgen1.pgnet.io**           | NSX Edge Node 1                  | 10.200.1.50  | ✅ Valid         |
| **pgen2.pgnet.io**           | NSX Edge Node 2                  | 10.200.1.51  | ✅ Valid         |
| **ops.pgnet.io**             | VCF Operations                   | 10.200.1.12  | ✅ Valid         |
| **opscol.pgnet.io**          | VCF Ops Collector                | 10.200.1.13  | ✅ Valid         |
| **opsnet.pgnet.io**          | VCF Ops for Networks (Platform)  | 10.200.1.44  | ✅ Valid         |
| **opsnetcol.pgnet.io**       | VCF Ops for Networks (Collector) | 10.200.1.45  | ✅ Valid         |
| **auto.pgnet.io**            | VCF Automation                   | 10.200.1.16  | ✅ Valid         |
| **log.pgnet.io**             | VCF Logs                         | 10.200.1.19  | ✅ Valid         |
| **pgnas.pgnet.io**           | NFS NAS Appliance                | 10.200.1.110 | ✅ Valid         |
| **t0-gateway.pgnet.io**      | Tier-0 Gateway VIP               | *TBD*        | *Create Record* |
| **router-uplink-1.pgnet.io** | Physical Router Peer 1           | 10.200.250.1 | ✅ Valid         |
| **router-uplink-2.pgnet.io** | Physical Router Peer 2           | 10.200.251.1 | ✅ Valid         |