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

Once the [**3 virtual machines**](https://www.vmware.com/topics/virtual-machine) have been set up correctly, we can get started. We'll start by configuring the [storage server](https://www.broadberry.fr/storage-servers) (the [iSCSI](https://www.techtarget.com/searchstorage/definition/iSCSI) Target Server). First, knowing that there's no DHCP server configured, we'll assign an **IP address** to the machine statically with the following command.

```powershell
New-NetIPAddress -InterfaceAlias "<InterfaceName>" -IPAddress "<IP-VM3>" -PrefixLength 24 -DefaultGateway "<IP-Gateway>"
Set-DnsClientServerAddress -InterfaceAlias (Get-NetAdapter -Name "<InterfaceName>" | Select-Object -ExpandProperty Name) -ServerAddresses "<IP-Gateway>"
```

The following syntax is a PowerShell command used to install a specific feature. [FS-iSCSITarget-Server](https://techdirectarchive.com/2021/07/14/how-to-install-and-configure-iscsi-target-server-and-iscsi-initiator-on-a-windows-server/), is the name of the feature that installs the tools and services needed to configure an [iSCSI](https://www.techtarget.com/searchstorage/definition/iSCSI) Target Server.

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

## Configure VM1 and VM2 (Cluster Nodes)

Like **virtual machine 3**, we're going to configure a static IP address for both virtual machines (which don't have a *DHCP server*).

```powershell
New-NetIPAddress -InterfaceAlias "<InterfaceName>" -IPAddress "<IP-VM1>" -PrefixLength 24 -DefaultGateway "<IP-Gateway>"
Set-DnsClientServerAddress -InterfaceAlias (Get-NetAdapter -Name "<InterfaceName>" | Select-Object -ExpandProperty Name) -ServerAddresses "<IP-Gateway>"
# On both nodes
```

In an environment **without Active Directory**, creating a *local administrator* account on each node is essential to ensure consistent administrative access. It allows for uniform and secure management of the nodes, simplifying tasks such as configuring [iSCSI shares](https://www.techtarget.com/searchstorage/definition/iSCSI) or cluster setups. A dedicated account enhances security by separating roles and avoiding reliance on the built-in Administrator account, while also ensuring streamlined permission management across the nodes.

```powershell
net user AdminCluster StrongPassword123! /add
net localgroup Administrators AdminCluster /add
# On both nodes
```

In the next step, we'll set up **[Failover Clustering](https://learn.microsoft.com/en-us/windows-server/failover-clustering/failover-clustering-overview)**. It allows servers to *work together* as a **cluster**, providing high availability for applications and services.

```powershell
Install-WindowsFeature -Name Failover-Clustering -IncludeManagementTools
# On both nodes
```
