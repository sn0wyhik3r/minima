---
layout: post
title: "Windows Server Core 2025 Cluster with iSCSI on VMware (No AD)"
categories: windows
---

In this article, we'll explain **how to set up** a [storage server](https://www.broadberry.fr/storage-servers) using [Windows Server Core 2025](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025) with [iSCSI](https://www.techtarget.com/searchstorage/definition/iSCSI), designed to be shared by two [nodes](https://docs.vmware.com/en/VMware-Tanzu-Service-Mesh/services/concepts-guide/GUID-6BA4B828-C778-47BD-8159-37847260148E.html) ([virtual machines](https://www.vmware.com/topics/virtual-machine) running [Windows Server Core 2025](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025)) within a [cluster environment](https://www.techopedia.com/definition/31922/virtual-machine-cluster-vm-cluster#:~:text=Virtual%20machine%20clusters%20work%20by%20protecting%20the%20physical,virtual%20machine%20clustering%20provides%20a%20dynamic%20backup%20processes.).

Well, to begin with, let's take a look at the various prerequisites :

| Component              | Minimum Requirement                                                                 |
|------------------------|-------------------------------------------------------------------------------------|
| **Hypervisor**         | VMware Workstation, ESXi, or any other compatible hypervisor                       |
| **Virtual Machines**   | - 3 VMs total :                                                                      |
|                        |   - 2 for the cluster nodes                                                        |
|                        |   - 1 for the iSCSI Target Server                                                  |
|                        | - Each VM requires :                                                                |
|                        |   - 2 vCPU                                                                          |
|                        |   - 4 GB RAM (8 GB recommended)                                                     |
|                        |   - 50 GB system disk (plus additional storage for iSCSI VM)                       |
| **Operating System**   | Windows Server 2025 Standard or Datacenter                                         |
| **Networking**         | - Static IP addresses for all VMs                                                  |
|                        | - Network connectivity between VMs on the same subnet or routed network            |
|                        | - Optional : Dedicated interface for iSCSI traffic                                  |
| **Roles and Features** | - Failover Clustering on VM1 and VM2                                               |
|                        | - Multi-Path I/O (MPIO) on VM1, VM2, and VM3                                        |
|                        | - iSCSI Target Server role on VM3                                                  |
| **Shared Storage**     | - At least one shared iSCSI disk                                                   |
|                        | - Formatted with NTFS                                                              |
|                        | - Adequate size for applications (100 GB or more)                            |
| **Accounts and Security** | - Identical local administrator accounts (same username and password) on VM1 and VM2 |
|                        | - Firewall configuration to allow required ports :                                  |
|                        |   - iSCSI : 3260                                                                    |
|                        |   - Clustering : 3343, 135, 445, 137-139                                            |
| **Management Tools**   | - PowerShell (v5.1 or later)                                                       |
|                        | - Failover Clustering management tools (installed via `-IncludeManagementTools`)    |
| **Knowledge**          | - Understanding of clustering, iSCSI, and PowerShell                               |
|                        | - Basic networking and Windows Server management                                   |
