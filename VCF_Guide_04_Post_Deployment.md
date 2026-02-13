# VCF 9.0.2 Deployment Guide - Part 4: Post-Deployment (Day 2)

## Phase 4: Post-Deployment

### Configure Identity & Certificates (Day 2)

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

### Identity and Access - VCF Operations
**Task - SSO Configuration for VCF**

This will add the users and groups from Active Directory as authenticated users in the VCF Operations suite. This allows you to use your existing AD credentials to log in to these platforms and manage permissions via AD groups.

Create and embedded identity broker SSO domain:

1. Select VCF domain - `pgvcf1`
2. Select Identity Broker (Embedded)
3. Select Directory-Based Identity Provider - AD/LDAP
4. Configure LDAP ([LDAP Configuration table](#ldap-configuration))
5. Configure Group Mappings ([Group Mapping table](#group-mapping))
6. Specify the base group DN - `DC=pgnet,DC=local`
7. Search VCF and Add ([User Groups in Active Directory table](#user-groups-in-active-directory))
8. Specify the base user DN - `DC=pgnet,DC=local`
9. Add the users from Active Directory ([Users in Active Directory table](#users-in-active-directory))


##### LDAP Configuration

| Field                                  | Value                                                    | Notes                                                        |
| -------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------ |
| **Identity Source Type**               | Active Directory over LDAP                               |                                                              |
| **Identity source name**               | `pgnet.local`                                            |                                                              |
| **Base distinguished name for users**  | `DC=pgnet,DC=local`                                      | Sets scope to entire domain to include users in `it`, `corp`, `sales`, etc. |
| **Base distinguished name for groups** | `DC=pgnet,DC=local`                                      | Sets scope to entire domain to include groups in all OUs.    |
| **Domain name**                        | `pgnet.local`                                            |                                                              |
| **Domain alias**                       | `pgnet`                                                  |                                                              |
| **Username**                           | `svc-ldap@pgnet.local` | Bind account created by Salt.                                |
| **Password**                           | `VMware123!VMware123!`                                   | Default service account password.                            |
| **Connect to**                         | Specific domain controllers                              |                                                              |
| **Primary server URL**                 | `ldaps://winsrv1.pgnet.local:636`                        | Requires CA cert trust.                                      |
| **Certificates**                       | *Upload Root CA Certificate*                             | Export from NAS/SMB share (`rootca.cer`).                    |

##### Group Mapping

| Attribute Name in VCF Identity broker | Attribute Name in Active Directory |
| ------------------------------------- | ---------------------------------- |
| **userName \***                       | `sAMAccountName`                   |
| **firstName**                         | `givenName`                        |
| **lastName**                          | `sn`                               |
| **distinguishedName**                 | `distinguishedName`                |
| **employeeID**                        | *(Not Mapped)*                     |
| **email**                             | `mail`                             |
| **userPrincipalName**                 | `userPrincipalName`                |

##### User Groups in Active Directory

| Name                   | Path                                                         |
| ---------------------- | ------------------------------------------------------------ |
| VCF-Auditors           | `CN=VCF-Auditors,OU=Groups,OU=it,DC=pgnet,DC=local`          |
| VCF-Certificate-Admins | `CN=VCF-Certificate-Admins,OU=Groups,OU=it,DC=pgnet,DC=local` |
| VCF-Cloud-Admins       | `CN=VCF-Cloud-Admins,OU=Groups,OU=it,DC=pgnet,DC=local`      |
| VCF-NSX-Admins         | `CN=VCF-NSX-Admins,OU=Groups,OU=it,DC=pgnet,DC=local`        |
| VCF-Operations         | `CN=VCF-Operations,OU=Groups,OU=it,DC=pgnet,DC=local`        |
| VCF-vSphere-Admins     | `CN=VCF-vSphere-Admins,OU=Groups,OU=it,DC=pgnet,DC=local`    |


##### Users in Active Directory


| Name                 | Path                                                       |
| -------------------- | ---------------------------------------------------------- |
| Domain Administrator | `CN=Domain Administrator,OU=Users,OU=it,DC=pgnet,DC=local` |
| DevOps Admin         | `CN=DevOps Admin,OU=Users,OU=it,DC=pgnet,DC=local`         |
| Infrastructure Admin | `CN=Infrastructure Admin,OU=Users,OU=it,DC=pgnet,DC=local` |
| Peter Hauck          | `CN=Peter Hauck,OU=Users,OU=it,DC=pgnet,DC=local`          |

**Task Complete - SSO Domain Configured**

#### Task - Add SSO to VCF Components

1. Under 'VCF Management` select `operations appliance` and Enable Single Sign-On
2. Select the `vc.pgnet.io` identity broker.
3. Under `VCF Management` select `automation appliance` and Enable Single Sign-On
4. Select the `vc.pgnet.io` identity broker.

#### Task - Add Certitificate CA to VCF

1. Under Fleet Management -> Certificates, `Configure CA`

| Field | Value |
| ----- | ----- |
| CA Server URL | `https://winsrv1.pgnet.local/certsrv` |
| User Name | `svc-vcf-ca@pgnet.local` |
| Password | `VMware123!VMware123!` |
| Template Name | `VMware` |

#### Task - Set Certifates on VCF Components

Select the TLS Certificate types -> ... Generate CSR 

Example Parameters for `fleet.pgnet.io`:

| Parameter                    | Value / Options |
| ---------------------------- | --------------- |
| **Common Name**              | `fleet.pgnet.io`  |
| **Organization**             | `pgnet`           |
| **Organizational Unit**      | `PGGB`            |
| **Country**                  | `Australia`       |
| **State/Province**           | `Queensland`      |
| **Locality**                 | `Brisbane`        |
| **Email Address**            | `admin@pgnet.io`  |
| **Host**                     | `fleet.pgnet.io`  |
| **Subject Alternative Name** | `fleet.pgnet.io`  |
| **Key Size**                 | `4096`            |

Example device table:

| Name             | Hostname         | Status | Issuer         | Expiration  | Support Status | Type            |
| ---------------- | ---------------- | ------ | -------------- | ----------- | -------------- | --------------- |
| Fleet Management | `fleet.pgnet.io` | Active | Microsoft CA   | 12 Feb 2028 | Activated      | TLS Certificate |
| VCF Automation   | `pgauto-w2npg`   | Active | Self Signed CA | 4 Feb 2028  | Activated      | TLS Certificate |
| VCF Operations   | `ops.pgnet.io`   | Active | Self Signed CA | 4 Feb 2028  | Activated      | TLS Certificate |

The CA certificate will need to be added to the host computer access the devices.



### Physical Router Configuration (BGP) (Day 2)

Before proceeding with the NSX Edge Peering, we must ensure the upstream physical router is configured to peer with the new Edge Nodes.

- **Gateway Status:** Ensure the interfaces `10.200.250.1` (VLAN 250) and `10.200.251.1` (VLAN 251) are active on your Physical Router (UDM).
- **Action:** Apply the FRR / BGP configuration to the UDM. This sets up the router to listen for connections from the NSX Edge Nodes on AS 65001.
- **Reference:** See **Appendix B** in the Appendices document for the full FRR (UDM) Router Configuration used in this lab.

### VPC Configuration - NSX Edge / Overlay

Once the VCF deployment is complete and the physical router is prepped, the Edge Cluster is deployed but not yet routing north-south traffic to your physical core.

1. From vCenter ([vc.pgnet.io](https://vc.pgnert.io))  --> Go to networks for the VCF cluster
   
   <img src="data/img/CleanShot 2026-02-12 at 21.46.29@2x.png" alt="CleanShot 2026-02-12 at 21.46.29@2x" style="zoom:25%;" />
   
2. Setup Network Connectivity --> Go To Network Connectivity. -> Configure Network Connectivity

   <img src="data/img/CleanShot 2026-02-12 at 21.49.09@2x.png" alt="CleanShot 2026-02-12 at 21.49.09@2x" style="zoom:25%;" />

3. Select Centralized Connectivity

   
   
   
   
   <img src="data/img/CleanShot 2026-02-12 at 21.51.36@2x.png" alt="CleanShot 2026-02-12 at 21.51.36@2x" style="zoom:25%;" />
   
4. Continue because all pre-reqs are configured and ready.

   

   

   <img src="data/img/CleanShot 2026-02-12 at 21.51.58@2x.png" alt="CleanShot 2026-02-12 at 21.51.58@2x" style="zoom:25%;" />

5. Configure Edge Node

   
   
   pgen1:
   
   | Field               | Value                        |
   | ------------------- | ---------------------------- |
   | **FQDN**            | pgen1.pgnet.io               |
   | **vSphere Cluster** | pgmgmt-cl01                  |
   | **Data Store**      | pgmgmt-cl01-ds-vsan01        |
   | **Size**            | Small                        |
   | **Management IP**   | 10.200.1.50/24               |
   | **Default Gateway** | 10.200.1.1                   |
   | **Port Group**      | pgmgmt-cl01-vds01-pg-vm-mgmt |
   
   
   
   | Virtual Interface | Interface | Active PNIC | Standby PNIC |
   | ----------------- | --------- | ----------- | ------------ |
   | 1                 | fp-eth0   | vmnic1      | vmnic2       |
   | 2                 | fp-eth1   | vmnic2      | vmnic1       |
   
   
   
   
   
   pgen2:
   
   | Field               | Value                        |
   | ------------------- | ---------------------------- |
   | **FQDN**            | pgen2.pgnet.io               |
   | **vSphere Cluster** | pgmgmt-cl01                  |
   | **Data Store**      | pgmgmt-cl01-ds-vsan01        |
   | **Size**            | Small                        |
   | **Management IP**   | 10.200.1.51/24               |
   | **Default Gateway** | 10.200.1.1                   |
   | **Port Group**      | pgmgmt-cl01-vds01-pg-vm-mgmt |
   
   
   
   | Virtual Interface | Interface | Active PNIC | Standby PNIC |
   | ----------------- | --------- | ----------- | ------------ |
   | 1                 | fp-eth0   | vmnic1      | vmnic2       |
   | 2                 | fp-eth1   | vmnic2      | vmnic1       |
   
   
   
   
   
      <img src="data/img/CleanShot 2026-02-12 at 22.09.04@2x.png" alt="CleanShot 2026-02-12 at 22.09.04@2x" style="zoom:25%;" />
   
   
      <img src="data/img/CleanShot 2026-02-12 at 22.09.23@2x.png" alt="CleanShot 2026-02-12 at 22.09.23@2x" style="zoom:25%;" />



6. Set Pool for TEP

   

   Pool configuration:


   | Parameter         | Value                       |
   | ----------------- | --------------------------- |
   | **TEP VLAN**      | 205                         |
   | **IP Allocation** | IP Pool                     |
   | **IP Pool Name**  | Pool-TEP1                   |
   | **CIDR**          | 10.200.5.0/24               |
   | **IP Range**      | 10.200.5.100 - 10.200.5.199 |
   | **Gateway IP**    | 10.200.5.1                  |
   | **DNS Servers**   | 10.200.1.240, 10.200.10.75  |
   | **DNS Suffix**    | pgnet.io                    |

​      

   


   <img src="data/img/CleanShot 2026-02-12 at 21.58.32@2x.png" alt="CleanShot 2026-02-12 at 21.58.32@2x" style="zoom:25%;" />

   

7. Setup Workload Domain Connectivity 

   

   | Category         | Field                                | Value          |
   | ---------------- | ------------------------------------ | -------------- |
   | **Identity**     | Gateway Name                         | pgrt1          |
   | **Availability** | High Availability Mode               | Active Standby |
   | **Routing**      | Gateway Routing Type                 | BGP            |
   | **Routing**      | Local Autonomous System Number (ASN) | 65001          |
   | **VPC**          | VPC External IP Blocks               | 10.210.0.0/16  |
   | **VPC**          | Private - Transit Gateway IP Blocks  | 10.220.0.0/16  |

   
   
      <img src="data/img/CleanShot 2026-02-12 at 22.08.25@2x.png" alt="CleanShot 2026-02-12 at 22.08.25@2x" style="zoom:25%;" />

   


8. Setup Gateway Uplinks

   Configure both uplinks and both edge nodes.
   
   
   
   
   
   pgen1:
   
   | Parameter             | First Uplink     | Second Uplink    |
   | --------------------- | ---------------- | ---------------- |
   | **Interface VLAN**    | 250              | 251              |
   | **Interface CIDR**    | 10.200.250.11/24 | 10.200.251.11/24 |
   | **BGP Peer IP**       | 10.200.250.1     | 10.200.251.1     |
   | **BGP Peer ASN**      | 65000            | 65000            |
   | **BFD**               | Enabled (Yes)    | Enabled (Yes)    |
   | **MTU**               | 9000             | 9000             |
   | **Target Router**     | TOR-1            | TOR-1            |
   | **Host Interface**    | vmnic1           | vmnic2           |
   | **Gateway Interface** | fp-eth0          | fp-eth1          |
   
   
      pgen2:
   
   | Parameter             | First Uplink     | Second Uplink    |
   | --------------------- | ---------------- | ---------------- |
   | **Interface VLAN**    | 250              | 251              |
   | **Interface CIDR**    | 10.200.250.12/24 | 10.200.251.12/24 |
   | **BGP Peer IP**       | 10.200.250.1     | 10.200.251.1     |
   | **BGP Peer ASN**      | 65000            | 65000            |
   | **BFD**               | Enabled (Yes)    | Enabled (Yes)    |
   | **MTU**               | 9000             | 9000             |
   | **Target Router**     | TOR-1            | TOR-1            |
   | **Host Interface**    | vmnic1           | vmnic2           |
   | **Gateway Interface** | fp-eth0          | fp-eth1          |
   
      
   
      
   
      
   
      <img src="data/img/CleanShot 2026-02-12 at 22.06.42@2x.png" alt="CleanShot 2026-02-12 at 22.06.42@2x" style="zoom:25%;" />
   
      




9. Finsihed Diagrams from Wizard

   
   
   <img src="data/img/CleanShot 2026-02-12 at 22.10.23@2x.png" alt="CleanShot 2026-02-12 at 22.10.23@2x" style="zoom:25%;" />
   
9. Build take a while and it will deploy Edge Nodes to the cluster.

   The process will fail if you do not have the AMD setting in the kickstart script see [Infrastructure Prep](VCF_Guide_02_Infrastructure_Prep.md)



