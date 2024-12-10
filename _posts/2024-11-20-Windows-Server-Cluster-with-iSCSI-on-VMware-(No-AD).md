---
layout: post
title: "Windows Server Core 2025 Cluster with iSCSI on VMware (No AD)"
categories: windows
---

In this article, we'll explain **how to set up** a [storage server](https://www.broadberry.fr/storage-servers) using [Windows Server Core 2025](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025) with [iSCSI](https://www.techtarget.com/searchstorage/definition/iSCSI), designed to be shared by two [nodes](https://docs.vmware.com/en/VMware-Tanzu-Service-Mesh/services/concepts-guide/GUID-6BA4B828-C778-47BD-8159-37847260148E.html) ([virtual machines](https://www.vmware.com/topics/virtual-machine) running [Windows Server Core 2025](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025)) within a [cluster environment](https://www.techopedia.com/definition/31922/virtual-machine-cluster-vm-cluster#:~:text=Virtual%20machine%20clusters%20work%20by%20protecting%20the%20physical,virtual%20machine%20clustering%20provides%20a%20dynamic%20backup%20processes.).

Well, to begin with, let's take a look at the various prerequisites :

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

Once the [**3 virtual machines**](https://www.vmware.com/topics/virtual-machine) have been set up correctly, we can get started. We'll start by configuring the [storage server](https://www.broadberry.fr/storage-servers) (the [iSCSI](https://www.techtarget.com/searchstorage/definition/iSCSI) Target Server). The following syntax is a PowerShell command used to install a specific feature. [FS-iSCSITarget-Server](https://techdirectarchive.com/2021/07/14/how-to-install-and-configure-iscsi-target-server-and-iscsi-initiator-on-a-windows-server/), is the name of the feature that installs the tools and services needed to configure an [iSCSI](https://www.techtarget.com/searchstorage/definition/iSCSI) Target Server.

```powershell
Install-WindowsFeature -Name FS-iSCSITarget-Server
```

Once we've done that, we'll create a [virtual storage](https://www.parallels.com/blogs/ras/virtual-storage/) disk for the [storage share](https://www.vmware.com/docs/introduction-to-storage-virtualization). To do this, we'll use a [cmdlet](https://learn.microsoft.com/en-us/powershell/scripting/developer/cmdlet/cmdlet-overview?view=powershell-7.4) provided by theallity function we added earlier.

```powershell
New-IscsiVirtualDisk -Path "C:\<FolderName>\<StorageDiskName>.vhdx" -Size 100GB
# 100GB is our example, but you can size it as you wish.
```

Now we're going to create a new "**iSCSI target**", which acts as an access point for iSCSI initiators (the systems that will consume the storage). The name is used to identify this target when it is connected by **initiators**.

```powershell
New-IscsiServerTarget -TargetName "<TargetName>" -InitiatorIds @("IPAddress:<IP-VM1>", "IPAddress:<IP-VM2>", "...")
```

We're also going to **associate** the [virtual disk](https://www.parallels.com/blogs/ras/virtual-storage/) we created earlier with the targets we've just configured.

```powershell
Add-IscsiVirtualDiskTargetMapping -TargetName "<TargetName>" -Path "C:\<FolderName>\<StorageDiskName>.vhdx"
```

Next, we'll activate the **Multipath I/O** ([MPIO](https://www.dell.com/support/kbdoc/en-us/000131854/mpio-what-is-it-and-why-should-i-use-it?msockid=21582e1206786daa394a3b4307d66c24)) feature in Windows Server or Windows 10/11, enabling multipath management for [SAN](https://www.ibm.com/topics/storage-area-network) (Storage Area Network) or [iSCSI](https://www.techtarget.com/searchstorage/definition/iSCSI) storage. It offers **[fault tolerance](https://www.geeksforgeeks.org/fault-tolerance-in-distributed-system/)** by redirecting **I/O** operations through an alternative path if the main path fails, improves performance by load balencing between several available paths, and is commonly used with [iSCSI](https://www.techtarget.com/searchstorage/definition/iSCSI) or Fibre Channel storage environments.

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName MultiPathIO
```
