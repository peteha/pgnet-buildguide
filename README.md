# VCF 9.0.2 Deployment Guide - Project PGNET

This repository contains the comprehensive build documentation and "Gold Path" orchestration for deploying **VMware Cloud Foundation 9.0.2** in a consolidated home lab environment (`pgnet.local`).

This is not a generic VCF deployment guide. It is a detailed, step-by-step reference for building the exact environment described in the documentation, using the specific hardware and software configurations outlined in the Bill of Materials (BOM).

This is not supported by VMware. It is a reference implementation for educational and lab purposes only. Use at your own risk.


## Architecture Overview

The deployment utilizes a **Consolidated Architecture** on a 3-node ESXi cluster, leveraging vSAN ESA and Memory Tiering to maximize resources. The environment is designed to be fully offline-capable, using local repositories and infrastructure services.

- **Primary Domain:** `pgnet.local` (Active Directory)
- **Infrastructure Domain:** `pgnet.io` (DNS/BIND)
- **VCF Version:** 9.0.2
- **Storage:** vSAN ESA (NVMe) + NFS (Bulk)

## Documentation Structure

The deployment guide is broken down into logical phases for easier version control and execution.

| Part   | Document                                                     | Description                                                  |
| ------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **01** | [Architecture & Planning](VCF_Guide_01_Architecture_Planning.md) | Bill of Materials (BOM), Network Schedule (VLANs/IPs), and Identity Strategy. |
| **02** | [Infrastructure Prep](VCF_Guide_02_Infrastructure_Prep.md) | Physical Switch configuration and ESXi Host Bootstrapping (Kickstart). |
| **03** | [The Deployment](VCF_Guide_03_Deployment.md) | Deploying the VCF Installer (Cloud Builder) and running the Bring-Up wizard. |
| **04** | [Post-Deployment](VCF_Guide_04_Post_Deployment.md) | Day 2 configurations: Identity (AD/LDAP), Certificates, Operations, and BGP Peering. |
| **05** | [Appendices](VCF_Guide_05_Appendices.md) | Reference configurations: FRR Router Config, Full Kickstart Scripts, JSON Specs, and Credentials. |
| **06** | [Windows AD Deployment](VCF_Guide_06_Windows_AD_Deployment.md) | Specialized Salt-based runbook for building the core Windows Infrastructure Server (`pglin1`). |

## Key Configurations

- **Orchestration:** The Windows Infrastructure Server is built using Salt Stack to ensure consistency.
- **Routing:** BGP is used between the NSX Edge Nodes and the physical core router (UDM/FRR).
- **Offline Mode:** All lifecycle bundles are managed via an Offline Depot workstation using the OBTU tool.

## Contributing / Updating

This documentation is maintained via an AI-assisted workflow.

1. **Stage:** Updates are generated in the Gemini Canvas environment.
2. **Commit:** Changes are copied to this repository and committed.
3. **Deploy:** Configurations (JSON/YAML) are applied to the VCF environment.