---
layout: post
title: "Setting Up a Windows Server Core 2025 Hyper-V Cluster with iSCSI Server on VMware Workstation (Without Active Directory)"
categories: windows
---

## Introduction

In this article, we'll explain **how to set up** a [storage server](https://www.broadberry.fr/storage-servers) using [Windows Server Core 2025](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025) with [iSCSI](https://www.techtarget.com/searchstorage/definition/iSCSI), configured to be shared by two [nodes](https://docs.vmware.com/en/VMware-Tanzu-Service-Mesh/services/concepts-guide/GUID-6BA4B828-C778-47BD-8159-37847260148E.html) ([virtual machines](https://www.vmware.com/topics/virtual-machine) running [Windows Server Core 2025](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025)) within a [cluster environment](https://www.techopedia.com/definition/31922/virtual-machine-cluster-vm-cluster#:~\:text=Virtual%20machine%20clusters%20work%20by%20protecting%20the%20physical,virtual%20machine%20clustering%20provides%20a%20dynamic%20backup%20processes.).

## Basic VM Hardware Prerequisites

| Component        | Minimum Requirement                              |
| ---------------- | ------------------------------------------------ |
| **CPU**          | 2 vCPUs per VM                                   |
| **Memory (RAM)** | 4 GB per VM (8 GB recommended)                   |
| **Storage**      | - 50 GB system disk per VM                       |
|                  | - Additional storage for iSCSI VM as needed      |
| **Network**      | - 1 network interface per VM                     |
|                  | - Optional : a second interface for iSCSI traffic |
| **VM Count**     | 3 VMs :                                           |
|                  | - 2 for cluster nodes                            |
|                  | - 1 for iSCSI Target Server                      |

## Additional Network Considerations for iSCSI

To optimize the iSCSI performance, consider the following configurations :

- **MTU (Jumbo Frames) :** Set the MTU to 9000 on the network interfaces dedicated to iSCSI traffic.
- **Dedicated Network :** Use a separate VLAN or subnet for iSCSI traffic to isolate it from general network traffic.
- **QoS (Quality of Service) :** Configure QoS to prioritize iSCSI traffic for improved reliability.

## Configure VM3 (iSCSI Target Server)

Once the **[3 virtual machines](https://www.vmware.com/topics/virtual-machine)** have been set up correctly, we can get started. We'll start by configuring the [storage server](https://www.broadberry.fr/storage-servers) (the [iSCSI](https://www.techtarget.com/searchstorage/definition/iSCSI) Target Server). First, we assign a static `IP address`, `subnet mask`, and `default gateway` to a specific **network interface**, ensuring consistent network configuration for environments like servers or devices that require fixed IP addresses for reliable communication.

```powershell
New-NetIPAddress -InterfaceAlias "<InterfaceName>" -IPAddress "<IP-VM3>" -PrefixLength 24 -DefaultGateway "<IP-Gateway>"
```

After, we set the `DNS server address` for the specified **network interface**, ensuring that the interface uses the provided `<IP-Gateway>` as its **DNS server** for name resolution. It is useful in environments where manual **DNS configuration** is required for consistent and reliable network operations.

```powershell
Set-DnsClientServerAddress -InterfaceAlias (Get-NetAdapter -Name "<InterfaceName>" | Select-Object -ExpandProperty Name) -ServerAddresses "<IP-Gateway>"
```

The following syntax is a PowerShell command used to install a specific feature. [FS-iSCSITarget-Server](https://learn.microsoft.com/en-us/windows-server/storage/iscsi/iscsi-target-server), is the name of the feature that installs the tools and services required to configure an [iSCSI](https://www.techtarget.com/searchstorage/definition/iSCSI) Target Server, enabling the server to host storage that can be accessed over the network by iSCSI initiators.

```powershell
Install-WindowsFeature -Name FS-iSCSITarget-Server
```

Once we've done that, we create a virtual disk for use with iSCSI. The **`New-IscsiVirtualDisk`** cmdlet creates a new virtual disk file at the specified path (`"C:\<FolderName>\<StorageDiskName>.vhdx"`) with a size of **100GB**. This virtual disk can later be mapped to an iSCSI target, allowing it to be shared and accessed by iSCSI initiators over the network.

```powershell
New-IscsiVirtualDisk -Path "C:\<FolderName>\<StorageDiskName>.vhdx" -Size 100GB
# 100GB is our example, but you can size it as you wish.
```

The following syntax is a PowerShell command used to create a new iSCSI target. The **`New-IscsiServerTarget`** cmdlet creates an iSCSI target with the specified name (`<TargetName>`), and associates it with a list of allowed initiators identified by their IP addresses (`<IP-VM1>`, `<IP-VM2>`). This defines which clients **can connect** to the target, ensuring secure and controlled access to the shared storage.

```powershell
New-IscsiServerTarget -TargetName "<TargetName>" -InitiatorIds @("IPAddress:<IP-VM1>", "IPAddress:<IP-VM2>", "...")
# For initiators, it is also possible to do the following to authorize all initiators: "IQN:*"
```

The following syntax is a PowerShell command used to associate a virtual disk with an existing iSCSI target. The **`Add-IscsiVirtualDiskTargetMapping`** cmdlet maps the specified [virtual disk](https://www.parallels.com/blogs/ras/virtual-storage/) file (`"C:\<FolderName>\<StorageDiskName>.vhdx"`) to the iSCSI target identified by `<TargetName>`. This allows the virtual disk to be accessed by initiators connected to the target, effectively exposing the storage for use over the network.

```powershell
Add-IscsiVirtualDiskTargetMapping -TargetName "<TargetName>" -Path "C:\<FolderName>\<StorageDiskName>.vhdx"
```

### Securing iSCSI with CHAP Authentication

To improve security, enable **CHAP (Challenge Handshake Authentication Protocol)** authentication for iSCSI targets. Configure the CHAP credentials using the following command :

```powershell
Set-IscsiServerTarget -TargetName "<TargetName>" -ChapUsername "<ChapUsername>" -ChapPassword "<ChapPassword>"
```

This ensures only authorized initiators can access the target.

### Enable Multipath I/O (MPIO)

Enable the **Multipath I/O (****[MPIO](https://www.dell.com/support/kbdoc/en-us/000131854/mpio-what-is-it-and-why-should-i-use-it?msockid=21582e1206786daa394a3b4307d66c24)****)** feature on the system. The **`Enable-WindowsOptionalFeature`** cmdlet activates the optional `MultiPathIO` feature, improving storage availability, fault tolerance, and performance.

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName MultiPathIO
```

## Configure VM1 and VM2 (Cluster Nodes)

Like **virtual machine 3**, configure a static IP address for both virtual machines :

```powershell
New-NetIPAddress -InterfaceAlias "<InterfaceName>" -IPAddress "<IP-VM>" -PrefixLength 24 -DefaultGateway "<IP-Gateway>"
Set-DnsClientServerAddress -InterfaceAlias (Get-NetAdapter -Name "<InterfaceName>" | Select-Object -ExpandProperty Name) -ServerAddresses "<IP-Gateway>"
# On both nodes
```

### Local Administrator Account Setup

Create a *local administrator* account on each node :

```powershell
net user "<AdminUser>" "<StrongPassword>" /add
net localgroup Administrators "<AdminUser>" /add
# On both nodes
```

### Connect Nodes to the iSCSI Target

Start the iSCSI Initiator Service and add a new iSCSI target portal :

```powershell
Start-Service -Name MSiSCSI
New-IscsiTargetPortal -TargetPortalAddress "IP-VM3"
Connect-IscsiTarget -NodeAddress "IQN-Target" -IsPersistent $true
# On both nodes
```

### Initialize and Format the Disk

Run the following commands to initialize, partition, and format the shared disk :

```powershell
Initialize-Disk -Number "<DiskNumber>"
New-Partition -DiskNumber "<DiskNumber>" -UseMaximumSize -AssignDriveLetter
Format-Volume -DriveLetter "<DriveLetter>" -FileSystem NTFS -NewFileSystemLabel "<StorageDiskName>"
# On only one
```

### Install Failover Clustering

Install the **Failover Clustering** feature on both nodes :

```powershell
Install-WindowsFeature -Name Failover-Clustering -IncludeManagementTools
```

### Test and Create the Cluster

Run the following commands to test and create the cluster :

```powershell
Test-Cluster -Node "VM1", "VM2"
New-Cluster -Name "ClusterName" -Node "VM1", "VM2" -StaticAddress "IP-Cluster"
# On only one
```

### Add the Shared Disk to the Cluster

Finally, add the shared disk to the cluster :

```powershell
Get-ClusterAvailableDisk | Add-ClusterDisk
```

## Conclusion

Setting up a Windows Server Core 2025 cluster with iSCSI on VMware provides a robust solution for high availability and shared storage in virtualized environments. By following this guide, you have successfully :

- Configured an iSCSI Target Server to host shared storage.
- Connected cluster nodes to the shared storage using secure and optimized iSCSI connections.
- Established a failover cluster with shared disk resources for redundancy.

This setup ensures that your environment can handle critical workloads with improved resilience, fault tolerance, and scalability. If you encounter any issues or need to customize further, consult Microsoft's documentation or reach out to your IT team for advanced configurations.
