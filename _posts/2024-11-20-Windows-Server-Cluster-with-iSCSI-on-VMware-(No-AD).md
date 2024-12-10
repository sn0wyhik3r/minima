---
layout: post
title: "Windows Server Core 2025 Cluster with iSCSI on VMware (No AD)"
categories: windows
---

## Introduction

In this article, we'll explain **how to set up** a [storage server](https://www.broadberry.fr/storage-servers) using [Windows Server Core 2025](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025) with [iSCSI](https://www.techtarget.com/searchstorage/definition/iSCSI), designed to be shared by two [nodes](https://docs.vmware.com/en/VMware-Tanzu-Service-Mesh/services/concepts-guide/GUID-6BA4B828-C778-47BD-8159-37847260148E.html) ([virtual machines](https://www.vmware.com/topics/virtual-machine) running [Windows Server Core 2025](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025)) within a [cluster environment](https://www.techopedia.com/definition/31922/virtual-machine-cluster-vm-cluster#:~:text=Virtual%20machine%20clusters%20work%20by%20protecting%20the%20physical,virtual%20machine%20clustering%20provides%20a%20dynamic%20backup%20processes.).

## Basic VM Hardware Prerequisites

| Component         | Minimum Requirement             |
|-------------------|----------------------------------|
| **CPU**           | 2 vCPUs per VM                 |
| **Memory (RAM)**  | 4 GB per VM (8 GB recommended)  |
| **Storage**       | - 50 GB system disk per VM     |
|                   | - Additional storage for iSCSI VM as needed |
| **Network**       | - 1 network interface per VM   |
|                   | - Optional : second interface for iSCSI traffic |
| **VM Count**      | 3 VMs :                         |
|                   | - 2 for cluster nodes          |
|                   | - 1 for iSCSI Target Server    |

## Configure VM3 (iSCSI Target Server)

Once the [**3 virtual machines**](https://www.vmware.com/topics/virtual-machine) have been set up correctly, we can get started. We'll start by configuring the [storage server](https://www.broadberry.fr/storage-servers) (the [iSCSI](https://www.techtarget.com/searchstorage/definition/iSCSI) Target Server). First, we assign a static `IP address`, `subnet mask`, and `default gateway` to a specific **network interface**, ensuring consistent network configuration for environments like servers or devices that require fixed IP addresses for reliable communication.

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
# For initiators, it is also possible to do the following to authorize all initiators : "IQN:*"
```

The following syntax is a PowerShell command used to associate a virtual disk with an existing iSCSI target. The **`Add-IscsiVirtualDiskTargetMapping`** cmdlet maps the specified [virtual disk](https://www.parallels.com/blogs/ras/virtual-storage/) file (`"C:\<FolderName>\<StorageDiskName>.vhdx"`) to the iSCSI target identified by `<TargetName>`. This allows the virtual disk to be accessed by initiators connected to the target, effectively exposing the storage for use over the network.

```powershell
Add-IscsiVirtualDiskTargetMapping -TargetName "<TargetName>" -Path "C:\<FolderName>\<StorageDiskName>.vhdx"
```

After that, we enable the **Multipath I/O ([MPIO](https://www.dell.com/support/kbdoc/en-us/000131854/mpio-what-is-it-and-why-should-i-use-it?msockid=21582e1206786daa394a3b4307d66c24))** feature on a Windows system. The **`Enable-WindowsOptionalFeature`** cmdlet activates the optional `MultiPathIO` feature, which provides support for multiple physical paths between a server and a storage device. This feature is essential for improving storage availability, fault tolerance, and performance by allowing load balancing and failover between paths in environments using [SAN](https://www.ibm.com/topics/storage-area-network) or [iSCSI](https://www.techtarget.com/searchstorage/definition/iSCSI)-based storage.

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName MultiPathIO
```

## Configure VM1 and VM2 (Cluster Nodes)

Like **virtual machine 3**, we're going to configure a static IP address for both virtual machines (which don't have a *DHCP server*).

```powershell
New-NetIPAddress -InterfaceAlias "<InterfaceName>" -IPAddress "<IP-VM>" -PrefixLength 24 -DefaultGateway "<IP-Gateway>"
Set-DnsClientServerAddress -InterfaceAlias (Get-NetAdapter -Name "<InterfaceName>" | Select-Object -ExpandProperty Name) -ServerAddresses "<IP-Gateway>"
# On both nodes
```

Creating a *local administrator* account on each node is essential to ensure consistent administrative access. It allows for uniform and secure management of the nodes, simplifying tasks such as configuring [iSCSI shares](https://www.techtarget.com/searchstorage/definition/iSCSI) or cluster setups. A dedicated account enhances security by separating roles and avoiding reliance on the built-in Administrator account, while also ensuring streamlined permission management across the nodes.

```powershell
net user "<AdminUser>" "<StrongPassword>" /add
net localgroup Administrators "<AdminUser>" /add
# On both nodes
```

This step, starts the Microsoft iSCSI Initiator Service ([MSiSCSI](https://techcommunity.microsoft.com/blog/filecab/iscsi-target-cmdlet-reference/424419)), which enables the server to connect to **iSCSI targets** and access shared storage over the network. It is a crucial step in configuring **iSCSI connectivity** on a Windows system.

```powershell
Start-Service -Name MSiSCSI
# On both nodes
```

This PowerShell command above, adds a new **iSCSI target portal** to the initiator configuration. The **`New-IscsiTargetPortal`** cmdlet specifies the IP address of the iSCSI target server (`"IP_du_serveur_iSCSI"`), allowing the client (initiator) to communicate with the server and discover available iSCSI targets for connection.

```powershell
New-IscsiTargetPortal -TargetPortalAddress "IP-VM3"
# On both nodes
```

The, we can connect the initiator to a specified iSCSI target. The **`Connect-IscsiTarget`** cmdlet uses the `NodeAddress` (the target's IQN identifier, `"IQN-Target"`) to establish the connection and ensures that the connection persists across system reboots with the `-IsPersistent $true` parameter. This is essential for maintaining consistent access to shared storage.

```powershell
Connect-IscsiTarget -NodeAddress "IQN-Target" -IsPersistent $true
# On both nodes
```

These commands **initialize** a specified disk, create a partition that uses the maximum `available space`, assign it a drive letter, and format it with the *NTFS* file system while labeling it as "ClusterDisk," making the disk ready for use in a storage or **cluster** environment. This configuration ensures that the disk is usable for the cluster. Initialization is performed only once, to avoid conflicts.

```powershell
Initialize-Disk -Number <DiskNumber>
New-Partition -DiskNumber <DiskNumber> -UseMaximumSize -AssignDriveLetter
Format-Volume -DriveLetter <DriveLetter> -FileSystem NTFS -NewFileSystemLabel "<StorageDiskName>"
# On only one
```

Then, install the **Failover Clustering** feature on a Windows Server. The **`-IncludeManagementTools`** parameter ensures that the associated management tools, such as the Failover Cluster Manager GUI and PowerShell cmdlets for managing clusters, are also installed. This is essential for configuring and managing high-availability clusters.

```powershell
Install-WindowsFeature -Name Failover-Clustering -IncludeManagementTools
# On both nodes
```

Run the following command on a **single VM**, for example `VM1`. The test automatically includes the two nodes specified in the parameters. This analyzes the compatibility of the **two nodes** for clustering and identifies any problems before creating the *cluster*.

```powershell
Test-Cluster -Node "VM1", "VM2"
```

Run the following command on a **single VM** (*the same VM where you tested the configuration*). Once the cluster has been created, it will be active on **both nodes**. This command **initializes** the `cluster` with a name and a virtual IP. The two nodes specified in the parameters become cluster members.

```powershell
New-Cluster -Name "ClusterName" -Node "VM1", "VM2" -StaticAddress "IP-Cluster"
```

And finally, run the following command on a **single VM**, *as for the previous commands*. Once added, the disk is available to `all nodes` in the cluster. It associates the [iSCSI](https://techcommunity.microsoft.com/blog/filecab/iscsi-target-cmdlet-reference/424419) shared disk with the cluster so that it serves as a quorum, ensuring consistent decisions across the cluster.

```powershell
Get-ClusterAvailableDisk | Add-ClusterDisk
```
