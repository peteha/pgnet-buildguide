# VCF 9.0.2 Deployment Guide - Appendices

## Appendix A: DNS Configuration

*Placeholder for your Salt/YAML DNS configuration to be inserted here.*

## Appendix B: Physical Router Configuration (FRR)

The following configuration is applied to the physical upstream router (UDM) to enable BGP peering with the NSX Edge nodes.

```shell
!
frr version 8.1
frr defaults traditional
hostname udmpggbnet
domainname pgnet.io
allow-external-route-update
no bgp send-extra-data zebra
service integrated-vtysh-config
!
router bgp 65000
bgp router-id 10.200.250.1
neighbor NSX peer-group
neighbor NSX remote-as 65001
neighbor NSX bfd 3 300 300
neighbor NSX password pggbnet
neighbor 10.200.250.11 peer-group NSX
neighbor 10.200.250.12 peer-group NSX
neighbor 10.250.251.11 peer-group NSX
neighbor 10.250.251.12 peer-group NSX
!
address-family ipv4 unicast
redistribute connected
neighbor NSX default-originate
neighbor NSX soft-reconfiguration inbound
neighbor NSX route-map ALLOW-ALL in
neighbor NSX route-map ALLOW-ALL out
exit-address-family
exit
!
route-map ALLOW-ALL permit 10
exit
!
end
```

## Appendix C: ESXi Kickstart Configurations

The following configurations are used to bootstrap the 3 physical hosts. **DNS, NTP, and VIB URLs have been updated** to point to the local Infrastructure Server (`10.200.1.240`) to ensure critical Day 0 connectivity in an offline environment.

```shell
vmaccepteula

clearpart --all --overwritevmfs --drives=t10.NVMe____SPCC_M.2_PCIe_SSD_______________________0000800431D5820C
clearpart --all --overwritevmfs --drives=t10.NVMe____CT2000T500SSD8__________________________C6A8704E0175A000
clearpart --all --overwritevmfs --drives=t10.NVMe____CT1000T500SSD8__________________________6667614E0175A000

install --drive=t10.NVMe____SPCC_M.2_PCIe_SSD_______________________0000800431D5820C --overwritevmfs
reboot

network --bootproto=static --device=vmnic1 --ip=10.200.1.224 --netmask=255.255.255.0 --gateway=10.200.1.1 --hostname=pgesxa3.pgnet.io --nameserver=10.200.1.1 --vlanid=201 --addvmportgroup=1

rootpw VMware123!VMware123!

%firstboot --interpreter=busybox

while ! vim-cmd hostsvc/runtimeinfo; do
sleep 10
done

esxcli network ip dns search add --domain=pgnet.io

# Configure NTP
esxcli system ntp set -s 203.14.0.251
esxcli system ntp set --enabled true

# enable & start SSH
vim-cmd hostsvc/enable_ssh
vim-cmd hostsvc/start_ssh

# enable & start ESXi Shell
vim-cmd hostsvc/enable_esx_shell
vim-cmd hostsvc/start_esx_shell

# Suppress ESXi Shell warning
esxcli system settings advanced set -o /UserVars/SuppressShellWarning -i 1

esxcli network firewall ruleset set -e true -r sshServer
esxcli system settings advanced set -o /UserVars/SuppressShellWarning -i 1

vim-cmd hostsvc/datastore/rename datastore1 pg-ds-pgesxa3-1


# Enable Memory Tiering
esxcli system settings kernel set -s MemoryTiering -v TRUE
esxcli system tierdevice create -d /vmfs/devices/disks/t10.NVMe____CT1000T500SSD8__________________________6667614E0175A000
esxcli system settings advanced set -o /Mem/TierNvmePct -i 300


/bin/generate-certificates


# AMD-specific configuration
# Add your AMD-specific ESXi configuration here
# Workaround required for AMD Ryzen-based CPU
echo 'monitor_control.disable_apichv ="TRUE"' >> /etc/vmware/config
echo 'cpuid.brandstring = "AMD EPYC Ryzen 9 9955HX"' >> /etc/vmware/config


# Memory Optimizations
esxcli system settings advanced set -o /Mem/ShareForceSalting -i 0
esxcli system settings advanced set -o /Mem/AllocGuestLargePage -i 0

# vSAN Optimizations
esxcli system settings advanced set -i 1 -o /VSAN/DOMNetworkSchedulerThrottleComponent

# Install vSAN ESA Mock VIB
esxcli network firewall ruleset set -e true -r httpClient
esxcli software acceptance set --level CommunitySupported
esxcli software vib install -v https://github.com/lamw/nested-vsan-esa-mock-hw-vib/releases/download/1.0/nested-vsan-esa-mock-hw.vib --no-sig-check
esxcli network firewall ruleset set -e false -r httpClient

esxcli network vswitch standard set --vswitch-name=vSwitch0 --mtu=9000
esxcli network ip interface set -i vmk0 -m 9000
esxcli network vswitch standard portgroup set --portgroup-name="Management Network" --vlan-id=201

reboot
```

## Appendix D: SDDC Manager Import JSON

This JSON file contains the configuration spec used to import settings into the VCF Installer UI.

```json
{
  "sddcId": "pgmgmt",
  "vcfInstanceName": "pbvcf1",
  "workflowType": "VCF",
  "version": "9.0.2.0",
  "ceipEnabled": true,
  "dnsSpec": {
    "nameservers": [
      "10.200.1.240"
    ],
    "subdomain": "pgnet.io"
  },
  "ntpServers": [
    "10.200.1.240"
  ],
  "vcenterSpec": {
    "vcenterHostname": "vc.pgnet.io",
    "rootVcenterPassword": "VMware123!VMware123!",
    "vmSize": "small",
    "storageSize": "",
    "adminUserSsoPassword": "VMware123!VMware123!",
    "ssoDomain": "vsphere.local",
    "useExistingDeployment": false
  },
  "clusterSpec": {
    "clusterName": "pgmgmt-cl01",
    "datacenterName": "pgmgmt-dc01",
    "datastoreSpec": {
      "vsanSpec": {
        "esaConfig": {
          "enabled": true
        },
        "datastoreName": "pgmgmt-cl01-ds-vsan01"
      }
    }
  },
  "nsxtSpec": {
    "nsxtManagerSize": "medium",
    "nsxtManagers": [
      {
        "hostname": "nsxm1.pgnet.io",
        "vipFqdn": "nsx.pgnet.io",
        "useExistingDeployment": false,
        "nsxtAdminPassword": "VMware123!VMware123!",
        "nsxtAuditPassword": "VMware123!VMware123!",
        "rootNsxtManagerPassword": "VMware123!VMware123!"
      }
    ],
    "skipNsxOverlayOverManagementNetwork": true,
    "transportVlanId": "205"
  },
  "vcfOperationsSpec": {
    "nodes": [
      {
        "hostname": "ops.pgnet.io",
        "rootUserPassword": "VMware123!VMware123!",
        "type": "master",
        "adminUserPassword": "VMware123!VMware123!",
        "applianceSize": "small",
        "useExistingDeployment": false
      }
    ]
  },
  "vcfOperationsFleetManagementSpec": {
    "hostname": "fleet.pgnet.io",
    "rootUserPassword": "VMware123!VMware123!",
    "adminUserPassword": "VMware123!VMware123!",
    "useExistingDeployment": false
  },
  "vcfOperationsCollectorSpec": {
    "hostname": "opscol.pgnet.io",
    "applianceSize": "small",
    "rootUserPassword": "VMware123!VMware123!",
    "useExistingDeployment": false
  },
  "vcfAutomationSpec": {
    "hostname": "auto.pgnet.io",
    "adminUserPassword": "VMware123!VMware123!",
    "ipPool": [
      "10.200.1.17",
      "10.200.1.26"
    ],
    "nodePrefix": "pgauto",
    "internalClusterCidr": "198.18.0.0/15",
    "useExistingDeployment": false
  },
  "hostSpecs": [
    {
      "hostname": "pgesxa1.pgnet.io",
      "credentials": {
        "username": "root",
        "password": "VMware123!VMware123!"
      }
    },
    {
      "hostname": "pgesxa2.pgnet.io",
      "credentials": {
        "username": "root",
        "password": "VMware123!VMware123!"
      }
    },
    {
      "hostname": "pgesxa3.pgnet.io",
      "credentials": {
        "username": "root",
        "password": "VMware123!VMware123!"
      }
    }
  ],
  "networkSpecs": [
    {
      "networkType": "MANAGEMENT",
      "subnet": "10.200.1.0/24",
      "gateway": "10.200.1.1",
      "vlanId": "201",
      "mtu": "9000",
      "teamingPolicy": "loadbalance_loadbased",
      "activeUplinks": [
        "uplink1",
        "uplink2"
      ],
      "portGroupKey": "pgmgmt-cl01-vds01-pg-esx-mgmt"
    },
    {
      "networkType": "VM_MANAGEMENT",
      "subnet": "10.200.1.0/24",
      "gateway": "10.200.1.1",
      "vlanId": "201",
      "mtu": "9000",
      "teamingPolicy": "loadbalance_loadbased",
      "activeUplinks": [
        "uplink1",
        "uplink2"
      ],
      "portGroupKey": "pgmgmt-cl01-vds01-pg-vm-mgmt"
    },
    {
      "networkType": "VMOTION",
      "subnet": "10.200.2.0/24",
      "gateway": "10.200.2.1",
      "includeIpAddressRanges": [
        {
          "startIpAddress": "10.200.2.100",
          "endIpAddress": "10.200.2.199"
        }
      ],
      "vlanId": "202",
      "mtu": "9000",
      "teamingPolicy": "loadbalance_loadbased",
      "activeUplinks": [
        "uplink1",
        "uplink2"
      ],
      "portGroupKey": "pgmgmt-cl01-vds01-pg-vmotion"
    },
    {
      "networkType": "VSAN",
      "subnet": "10.200.3.0/24",
      "gateway": "10.200.3.1",
      "includeIpAddressRanges": [
        {
          "startIpAddress": "10.200.3.100",
          "endIpAddress": "10.200.3.199"
        }
      ],
      "vlanId": "203",
      "mtu": "9000",
      "teamingPolicy": "loadbalance_loadbased",
      "activeUplinks": [
        "uplink1",
        "uplink2"
      ],
      "portGroupKey": "pgmgmt-cl01-vds01-pg-vsan"
    }
  ],
  "dvsSpecs": [],
  "sddcManagerSpec": {
    "hostname": "sddc.pgnet.io",
    "useExistingDeployment": false,
    "rootPassword": "VMware123!VMware123!",
    "sshPassword": "VMware123!VMware123!",
    "localUserPassword": "VMware123!VMware123!"
  }
}
```

## Appendix E: Appliance Credentials Reference

```json
{
  "components": [
    {
      "componentName": "VCF Operations Appliance",
      "FQDN": "ops.pgnet.io",
      "credentials": [
        {
          "type": "Root credentials",
          "user": "root",
          "password": "VMware123!VMware123!"
        },
        {
          "type": "Administrator credentials",
          "user": "admin",
          "password": "VMware123!VMware123!"
        }
      ]
    },
    {
      "componentName": "Fleet Management Appliance",
      "FQDN": "fleet.pgnet.io",
      "credentials": [
        {
          "type": "Root credentials",
          "user": "root",
          "password": "VMware123!VMware123!"
        },
        {
          "type": "Administrator credentials",
          "user": "admin@local",
          "password": "VMware123!VMware123!"
        }
      ]
    },
    {
      "componentName": "Operations Collector Appliance",
      "FQDN": "opscol.pgnet.io",
      "credentials": [
        {
          "type": "Administrator credentials",
          "user": "root",
          "password": "VMware123!VMware123!"
        }
      ]
    },
    {
      "componentName": "VCF Automation",
      "FQDN": "auto.pgnet.io",
      "credentials": [
        {
          "type": "Administrator credentials",
          "user": "admin",
          "password": "VMware123!VMware123!"
        }
      ]
    },
    {
      "componentName": "vCenter",
      "FQDN": "vc.pgnet.io",
      "credentials": [
        {
          "type": "Root credentials",
          "user": "root",
          "password": "VMware123!VMware123!"
        },
        {
          "type": "Administrator credentials",
          "user": "administrator@vsphere.local",
          "password": "VMware123!VMware123!"
        }
      ]
    },
    {
      "componentName": "NSX Manager",
      "FQDN": "nsx.pgnet.io",
      "credentials": [
        {
          "type": "Root credentials",
          "user": "root",
          "password": "VMware123!VMware123!"
        },
        {
          "type": "Administrator credentials",
          "user": "admin",
          "password": "VMware123!VMware123!"
        },
        {
          "type": "Audit credentials",
          "user": "audit",
          "password": "VMware123!VMware123!"
        }
      ]
    },
    {
      "componentName": "Hosts",
      "FQDN": "pgesxa1.pgnet.io",
      "credentials": [
        {
          "type": "Root credentials",
          "user": "root",
          "password": "VMware123!VMware123!"
        }
      ]
    },
    {
      "componentName": "Hosts",
      "FQDN": "pgesxa2.pgnet.io",
      "credentials": [
        {
          "type": "Root credentials",
          "user": "root",
          "password": "VMware123!VMware123!"
        }
      ]
    },
    {
      "componentName": "Hosts",
      "FQDN": "pgesxa3.pgnet.io",
      "credentials": [
        {
          "type": "Root credentials",
          "user": "root",
          "password": "VMware123!VMware123!"
        }
      ]
    },
    {
      "componentName": "SDDC Manager",
      "FQDN": "sddc.pgnet.io",
      "credentials": [
        {
          "type": "Root credentials",
          "user": "root",
          "password": "VMware123!VMware123!"
        },
        {
          "type": "VCF credentials",
          "user": "vcf",
          "password": "VMware123!VMware123!"
        },
        {
          "type": "Local Admin Credentials",
          "user": "admin@local",
          "password": "VMware123!VMware123!"
        }
      ]
    }
  ]
}
```