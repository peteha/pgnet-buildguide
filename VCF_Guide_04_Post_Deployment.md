# VCF 9.0.2 Deployment Guide - Part 4: Post-Deployment (Day 2)

## Phase 4: Post-Deployment

### 4.1 Configure Identity & Certificates (Day 2)

**Pre-requisite:** Ensure your Windows Active Directory server is fully deployed and operational. Refer to [**VCF Guide Part 6: Windows AD Server Deployment**](VCF_Guide_06_Windows_AD_Deployment.md) for setup instructions.

Before deploying additional operations or workload components, we must establish the chain of trust and identity source.

1. **Active Directory Integration:**
   - **Domain:** `pgnet.local` (Authentication Domain)
   - **Action:** Log in to SDDC Manager and/or vCenter Server.
   - **Identity Provider (IDP):** Configure LDAPS connection to your Active Directory Domain Controller (ensuring `10.200.1.240` correctly forwards these requests).
   - **Groups:** Map your VCF Admin group from `pgnet.local` to the Cloud Admin role.
2. **Certificate Management:**
   - **CA Source:** Microsoft CA (joined to `pgnet.local`).
   - **Action:**
     1. Use SDDC Manager to generate CSRs for SDDC Manager, vCenter, and NSX Manager.
     2. Submit CSRs to your Microsoft CA.
     3. Import the signed certificates back into SDDC Manager.
     4. Execute the "Replace Certificates" workflow to propagate trust across the stack.

### 4.2 Deploy VCF Operations Suite

1. **Operations (Aria Ops):**
   - Log in to SDDC Manager and deploy VCF Operations.
   - **Deploy Remote Collector:** Deploy the Ops Remote Collector node to `opscol.pgnet.io` (10.200.1.13).
2. **Operations for Networks (Aria Ops for Networks/vRNI):**
   - **Platform Node:** Deploy to `opsnet.pgnet.io` (10.200.1.44).
   - **Collector Node:** Deploy to `opsnetcol.pgnet.io` (10.200.1.45).
   - **Configuration:** Pair the collector with the platform and configure flow data collection from the Distributed Switches.

### 4.3 Populating the Offline Depot (William Lam Method)

Since our SDDC Manager is offline/air-gapped for control, we will use the **Depot Workstation** (internet connected) to harvest the required bundles using the method documented by William Lam.

**Process Overview:**

1. **Generate Marker File:**
   - Log in to your offline SDDC Manager via SSH.
   - Generate the `markerFile` (JSON) which represents the current state of your VCF environment.
   - Command: `/opt/vmware/vcf/lcm/lcm-app/bin/generate-marker-file`
   - Transfer this file to your **Depot Workstation**.
2. **Download Bundles (On Depot Workstation):**
   - On the workstation, run the **Bundle Transfer Utility** (OBTU).
   - Point it to the `markerFile` and your Broadcom Support credentials.
   - The tool will identify exactly which bundles are needed for VCF 9.0.2 (and future patches) and download them to a local directory.
   - *Reference:* [William Lam - Automating VCF Offline Bundle Download](https://williamlam.com) (Search for his specific script `download_vcf_bundles.py` or his guide on OBTU filters to avoid downloading petabytes of unnecessary data).
3. **Upload to SDDC Manager:**
   - Transfer the downloaded bundle directory from the Workstation to the SDDC Manager (via SCP/WinSCP).
   - Run the OBTU in upload mode on the SDDC Manager to ingest the bundles into the Lifecycle Management database.
   - Command: `lcm-bundle-transfer-util --upload --bundleDirectory ...`
4. **Verify:**
   - Refresh the SDDC Manager UI -> Bundle Management. The new bundles should appear as "Available."

### 4.4 Verify Memory Tiering

*Note: Memory Tiering was enabled during the Kickstart (Phase 2.2).*

1. **Verification:** Check Host Summary for each host (`pgesxa1-3`) to ensure "Tiered Memory" is active and total capacity reflects the augmentation (Physical RAM + 300% NVMe Tier).
2. **No Action Required:** Since this was handled in Day 0, no maintenance mode or reboot is required in Day 2 unless the configuration failed.

### 4.5 Physical Router Configuration (BGP) (Day 2)

Before proceeding with the NSX Edge Peering, we must ensure the upstream physical router is configured to peer with the new Edge Nodes.

- **Gateway Status:** Ensure the interfaces `10.200.250.1` (VLAN 250) and `10.200.251.1` (VLAN 251) are active on your Physical Router (UDM).
- **Action:** Apply the FRR / BGP configuration to the UDM. This sets up the router to listen for connections from the NSX Edge Nodes on AS 65001.
- **Reference:** See **Appendix B** in the Appendices document for the full FRR (UDM) Router Configuration used in this lab.

### 4.6 NSX Edge Peering & Overlay Setup (Day 2)

Once the VCF deployment is complete and the physical router is prepped, the Edge Cluster is deployed but not yet routing north-south traffic to your physical core.

**Step 1: Configure Tier-0 Gateway**

1. Log in to **NSX Manager**.
2. Navigate to **Networking > Tier-0 Gateways**.
3. Create a new T0 Gateway (or edit the default one if created by VCF).
4. **HA Mode:** Active/Active (Recommended for simple ECMP) or Active/Standby.
5. **Local AS:** Set to **65001** (Must match the remote-as in your FRR config).

**Step 2: Configure Interfaces** Configure the uplinks to match the "RouterNet" subnets defined in your BOM:

- **Edge Node 1 / Uplink 1:** `10.200.250.11/24` (Connects to VLAN 250)
- **Edge Node 1 / Uplink 2:** `10.200.251.11/24` (Connects to VLAN 251)
- **Edge Node 2 / Uplink 1:** `10.200.250.12/24` (Connects to VLAN 250)
- **Edge Node 2 / Uplink 2:** `10.200.251.12/24` (Connects to VLAN 251)

**Step 3: Configure BGP Neighbors** Under the T0 Gateway BGP settings, add your physical router peers:

- **Neighbor 1:** `10.200.250.1` (Remote AS: 65000)
- **Neighbor 2:** `10.200.251.1` (Remote AS: 65000)
- **Password:** `pggbnet` (As defined in FRR config)
- **BFD:** Enabled (recommended for fast failover).

**Step 4: Route Redistribution**

1. In the T0 Gateway, expand **Route Re-distribution**.
2. Enable redistribution for:
   - Connected Interfaces / Segments
   - Tier-1 Connected Subnets (Load Balancer VIPs, Workload segments)

**Verification:** Run `get bgp neighbor summary` on the NSX Edge CLI. You should see state `Established` for neighbors `10.200.250.1` and `10.200.251.1`.

### 4.7 Enable Supervisor Cluster (VKS) (Day 2)

To enable Kubernetes support (vSphere with Tanzu), we will activate the Supervisor Cluster on the Management Domain.

1. **Workload Management:** In vSphere Client, go to **Workload Management**.
2. **Select Cluster:** Choose the Management Domain cluster.
3. **Network Stack:** Select "NSX" as the networking stack.
4. **Management Network:** Map to **VLAN 208 (VKS Mgmt)**.
   - *Note:* This provides IP addresses for the Supervisor Control Plane VMs.
5. **Workload Network:** Map to **VLAN 209 (K8 Workload)**.
   - *Note:* This is the IP range used for the actual Tanzu Kubernetes Grid (TKG) clusters you deploy later.
6. **Content Library:** Select a Subscribed Content Library to pull TKG images.
7. **Size:** Select "Small" or "Tiny" for this lab environment.

### 4.8 Configure VCF Automation & VPCs (End State)

To support modern applications and IAAS, we will deploy VCF Automation and configure the Tenant structure.

1. **Deploy VCF Automation (Aria Automation):**
   - Deploy the Automation appliance to `auto.pgnet.io` (10.200.1.16).
   - Integrate with Identity Manager (vIDM/Workspace ONE).
2. **Cloud Account:**
   - Add the local vCenter (`vc.pgnet.io`) and NSX Manager (`nsx.pgnet.io`) as a "vSphere Cloud Account".
3. **Tenancy & VPC Structure:**
   - **Project:** Create a "Default Project" for lab usage.
   - **Cloud Zone:** Map the Consolidated Cluster as the Cloud Zone.
   - **VPC Policy:** Create a Virtual Private Cloud (VPC) policy to allow on-demand network creation.
   - **Image/Flavor Mappings:** Create mappings for your VM templates (e.g., "small" = 2vCPU/4GB).