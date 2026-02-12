# VCF 9.0.2 Deployment Guide - Part 2: Infrastructure Preparation

## Phase 2: Infrastructure Prep (The "Plumbing")

### 2.1 Physical Switch Configuration

- **MTU:** Set global MTU to 9000 (Jumbo Frames) for VLANs 202, 203, 205, and 206.
- **Trunking:** Configure switch ports connected to the 3 ESXi hosts as **Trunks** (All VLANs tagged).
  - *Native VLAN:* Set the **Native VLAN** on these ports to **201** (Management). This ensures that untagged traffic from the host (like the initial PXE boot or Kickstart network request) lands on the correct management subnet without requiring manual VLAN tagging in the script.

### 2.2 ESXi Host bootstrapping (Automated Kickstart)

We will utilize **Kickstart Scripts** (`pgesxa1_ks.cfg` through `pgesxa3_ks.cfg`) to automate the "Day 0" configuration of the ESXi hosts. This ensures consistency across the cluster.

**The Kickstart scripts perform the following automated actions:**

1. **Disk Cleaning:** Wipes specific NVMe partitions to ensure a clean state for vSAN ESA.
2. **OS Install:** Installs ESXi to the designated boot NVMe.
3. **Networking:** Configures Management IP (`10.200.1.x`), Gateway (`10.200.1.1`), and VLAN 201.
   - *Note:* The scripts configure the management interface (`vmk0`) with MTU 9000. Ensure your switch supports this on the access ports.
4. **Security:** Enables SSH, suppresses shell warnings, and injects SSH Authorized Keys.
5. **Memory Tiering (Crucial):**
   - Enables `MemoryTiering`.
   - Creates a tier device on a designated NVMe drive.
   - Sets NVMe Tier Percentage to 300%.
6. **Lab Specific Customization:**
   - **vSAN ESA Mock VIB:** The scripts download and install a community VIB.
     - *Offline Depot Note:* Since your environment is offline, you must host this VIB on your internal Infrastructure Server (e.g., `http://10.200.1.240/vib/nested-vsan-esa-mock-hw.vib`) and update the Kickstart script URL accordingly. The provided scripts currently point to GitHub; **you must edit them** to point to your local web server.
     - *Context:* VCF strictly validates hardware against the HCL. This mock VIB tricks VCF into accepting consumer NVMe drives for vSAN ESA.
     - *Ref:* [William Lam's vSAN ESA Mock VIB Documentation](https://williamlam.com/2025/02/vsan-esa-hardware-mock-vib-for-physical-esxi-deployment-for-vmware-cloud-foundation-vcf.html).
   - **AMD Ryzen Workaround:** The scripts append `monitor_control.disable_apichv = "TRUE"` to `/etc/vmware/config`.
     - *Context:* This disables hardware APIC virtualization. This is critical for consumer AMD Ryzen CPUs to ensure that nested VMs (like NSX Edge Nodes) and features like Memory Tiering boot reliably without hanging or PSODing.
     - **Additional AMD Command:** The scripts also append `cpuid.brandstring = "AMD EPYC Ryzen 9 9955HX"` to `/etc/vmware/config`.
     - *Ref:* [William Lam's NVMe Tiering & AMD Workaround Documentation](https://williamlam.com/2025/06/nvme-tiering-with-amd-ryzen-cpu-workaround-for-vcf-9-0.html) and [Improved workaround for NSX Edge on AMD Ryzen (VCF 9.0.2)](https://williamlam.com/2026/01/improved-workaround-for-nsx-edge-deployment-upgrade-to-vcf-9-0-2-running-amd-ryzen-cpus.html).

*Reference:* See **[Appendices](VCF_Guide_05_Appendices.md)** in the Appendices document for the full Kickstart configuration files.