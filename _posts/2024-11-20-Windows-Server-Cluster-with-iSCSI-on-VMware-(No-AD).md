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
