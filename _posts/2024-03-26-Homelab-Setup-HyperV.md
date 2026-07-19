---
title: Home Lab Setup - HyperV
author: uzrg
date: 2024-03-26 00:34:00 +0800
categories: [Homelab, Virtualization]
tags: [Microsoft, HyperV, Windows, Windows Server, Ubuntu, CentOS, Linux]
pin: true
---

# Hyper-V Lab Infrastructure Documentation

## Overview

This page documents my Hyper-V homelab environment running on a Supermicro server hosting a complete Microsoft enterprise infrastructure stack.

## Physical Infrastructure

**SUPERLAB Host Server:**

- **Hardware:** Supermicro Super Server
- **Operating System:** Windows Server 2022 Datacenter
- **CPU:** AMD EPYC 3251 8-Core
- **Memory:** 255.89 GB RAM
- **Storage:** 8005.8 GB

## Architecture Diagrams

### Detailed System Architecture

```
SUPERLAB  ·  Supermicro Super Server
Windows Server 2022 Datacenter  ·  Hyper-V Role Enabled
AMD EPYC 3251 8-Core  ·  255.89 GB RAM  ·  8005.8 GB Storage
│
├── Network & Security Infrastructure
│   ├── pfSense      Firewall / Router
│   └── VMM01        Virtual Machine Manager
│
├── Active Directory Domain Services  (myhomelab.hv.lab)
│   ├── DC01         Primary Domain Controller
│   └── DC02         Secondary DC  ←→  AD Replication
│
├── SQL Server Always On Availability Group
│   ├── SQL01        Primary Replica   ─┐
│   ├── SQL02        Secondary Replica  ├─ Always On AG
│   ├── SQL03        Secondary Replica ─┘
│   ├── SQLAGL01     Availability Group Listener
│   ├── SQLCLU01     Windows Failover Cluster
│   ├── VLAN20       Cluster / Replication Network
│   └── Databases hosted on Always On AG
│       ├── AzureDevOps_Configuration  ·  AzureDevOps_myhomelab
│       ├── CM_MHL  (MECM site database)
│       ├── OperationsManager  ·  OperationsManagerDW  (SCOM databases)
│       ├── RDSCB_DB  (RDS Connection Broker database)
│       └── SUSDB  (WSUS Database — see MECM / SUP)
│
├── Remote Desktop Services Farm
│   ├── RDSCB        Connection Broker
│   ├── RDSGW        Gateway & Web Access
│   ├── RDSLC        License Server
│   ├── RDSSH01      Session Host 1
│   └── RDSSH02      Session Host 2
│
├── System Center Configuration Manager
│   ├── MECM01       Primary Site Server  (Active)
│   ├── MECM02       Primary Site Server  (Standby)
│   └── WSUS01       Software Update Point (SUP)
│
├── System Center Operations Manager
│   ├── OPSMGR01     Management Server 1
│   └── OPSMGR02     Management Server 2
│
├── DevOps & File Services
│   ├── DEVOPS01     Azure DevOps Server
│   └── FS01         File Server
│
└── Linux Systems
    ├── UBUNTU01     Ubuntu Server
    └── CENTOS01     CentOS Server
```

### Network Overview

```
Internet  (WAN)
│
└── SUPERLAB Physical Host
    │   Windows Server 2022 Datacenter  ·  Hyper-V
    │   AMD EPYC 3251 8-Core  ·  255.89 GB RAM  ·  8005.8 GB Storage
    │
    ├── WAN Interface  ──►  External Virtual Switch
    │                       └── pfSense  Firewall / Router  (internet perimeter)
    │
    └── LAN Interface  ──►  Internal Virtual Switch
        │
        ├── Active Directory Domain Services  (myhomelab.hv.lab)
        │   ├── DC01     Primary Domain Controller
        │   └── DC02     Secondary DC  ←→  AD Replication
        │
        ├── SQL Server Always On  (SQL01  ·  SQL02  ·  SQL03)
        │   └── Databases  →  DevOps  ·  MECM  ·  SCOM  ·  RDS Broker  ·  SUSDB (WSUS)
        │
        ├── Remote Desktop Services
        │   ├── RDSCB  ·  RDSGW  ·  RDSLC  Core services
        │   └── RDSSH01  ·  RDSSH02        Session hosts
        │
        ├── System Center Configuration Manager
        │   ├── MECM01  (Active)  ·  MECM02  (Standby)
        │   └── WSUS01                                   Software Update Point (SUP)
        │
        ├── System Center Operations Manager
        │   └── OPSMGR01  ·  OPSMGR02
        │
        ├── DevOps & File Services
        │   ├── DEVOPS01                         Azure DevOps Server
        │   └── FS01                             File Server
        │
        └── Infrastructure Services
            └── VMM01                            Virtual Machine Manager
```

### Virtual Switch Architecture

Two Hyper-V virtual switches carry all traffic. Inter-VLAN routing
happens entirely in software on pfSense — there's no physical VLAN
trunk anywhere in this design, only a virtual one:

```
Virtual Switch Manager
│
├── External-VSwitch-WAN         External  ·  bound to a physical NIC
│   └── pfSense (WAN NIC)        Untagged  ·  internet uplink
│
└── Internal-VSwitch-LAN-TRUNK   Internal  ·  no physical NIC — routing stays virtual
    │
    ├── pfSense (LAN NIC)        Trunk  ·  VLANs 10, 20, 30, 40
    │   └── Inter-VLAN routing handled entirely by pfSense
    │
    ├── VLAN 10 — Servers                 Access mode
    │   └── DC01 · DC02 · DHCP01 · FS01 · MECM01/02 · NPS01 · OPSMGR01
    │       RDSCB · RDSGW · RDSLC · RDSSH01/02 · VMM01 · WSUS01 · DEVOPS01
    │       SQL01 · SQL02 · SQL03  (first NIC)
    │
    ├── VLAN 20 — Cluster / Replication   Access mode
    │   └── SQL01 · SQL02 · SQL03  (second NIC)
    │
    └── VLAN 30 / VLAN 40 — Reserved      Trunked through, unassigned
```

Every VM gets a single access-mode NIC tagged to its VLAN, invisible
to the guest OS. pfSense is the only trunk exception, carrying all
four VLANs on one NIC. SQL01–03 are the only other dual-homed VMs,
with a second NIC on VLAN 20 for cluster/replication traffic.

For reference, here's how it is done with PowerShell:

```powershell
# Create the two virtual switches
New-VMSwitch -Name "External-VSwitch-WAN" -NetAdapterName "<physical NIC>" -AllowManagementOS $true
New-VMSwitch -Name "Internal-VSwitch-LAN-TRUNK" -SwitchType Internal

# pfSense: WAN NIC stays untagged, LAN NIC becomes an 802.1Q trunk
Set-VMNetworkAdapterVlan -VMName pfSense -VMNetworkAdapterName "WAN" -Untagged
Set-VMNetworkAdapterVlan -VMName pfSense -VMNetworkAdapterName "LAN" -Trunk -AllowedVlanIdList "10,20,30,40" -NativeVlanId 0

# Every other VM: one access-mode NIC per VLAN
Set-VMNetworkAdapterVlan -VMName <VMName> -VMNetworkAdapterName <NICName> -Access -VlanId 10

# SQL nodes: second NIC dedicated to cluster/replication traffic
Set-VMNetworkAdapterVlan -VMName SQL01 -VMNetworkAdapterName <ClusterNIC> -Access -VlanId 20
```

## Infrastructure Components

### Network & Security Infrastructure

- **pfSense:** Firewall/Router — inter-VLAN routing and perimeter security
- **VMM01:** Virtual Machine Manager Server for Hyper-V infrastructure management
- **DHCP01:** DHCP Server providing automated IP address assignment

### Active Directory Domain Services

- **Domain:** myhomelab.hv.lab
- **DC01:** Primary Domain Controller providing authentication and directory services
- **DC02:** Secondary Domain Controller with automatic replication and failover

### SQL Server Always On Availability Group

The SQL infrastructure provides high availability database services through a three-node cluster:

- **SQL01:** SQL Server Node 1 - Primary replica
- **SQL02:** SQL Server Node 2 - Secondary replica
- **SQL03:** SQL Server Node 3 - Secondary replica

### SQL Support Services

- **SQLAGL01:** Availability Group Listener for client connections
- **SQLCLU01:** Windows Failover Cluster providing cluster management

### Remote Desktop Services Farm

Complete RDS deployment providing virtual desktop and application services:

- **RDSCB:** Connection Broker managing user sessions and load balancing
- **RDSGW:** RDS Gateway & Web Access providing secure external access
- **RDSLC:** License Server managing RDS client access licenses
- **RDSSH01:** Session Host 1 providing virtual desktop sessions
- **RDSSH02:** Session Host 2 providing additional capacity and redundancy

### System Center Configuration Manager (MECM)

Enterprise system management and software deployment:

- **MECM01:** MECM Primary Site Server managing the entire infrastructure
- **MECM02:** MECM Primary Site Server (Standby) in Active/Standby HA with MECM01
- **WSUS01:** Windows Server Update Services serving as Software Update Point (SUP) for MECM

### System Center Operations Manager (SCOM)

Infrastructure monitoring and alerting:

- **OPSMGR01:** Management Server 1 providing monitoring services
- **OPSMGR02:** Management Server 2 providing redundancy and load balancing

### DevOps & File Services

- **DEVOPS01:** Azure DevOps Server providing source control, build, and deployment services
- **FS01:** File Server providing centralized file storage and sharing

### Linux Systems

- **UBUNTU01:** Ubuntu Server - available for Linux workloads
- **CENTOS01:** CentOS Server - available for Linux workloads

## High Availability Features

### SQL Server Always On

The three-node SQL Server Always On Availability Group provides:
- Automatic failover between nodes
- Read-only routing to secondary replicas
- Backup on secondary replicas
- Distributed database hosting for critical applications including:
  - Azure DevOps Server databases
  - System Center Configuration Manager database
  - System Center Operations Manager databases
  - RDS Connection Broker database

### Active Directory Replication

Dual domain controllers ensure:
- Automatic replication of Active Directory data
- Redundant DNS services
- High availability authentication services

### System Center Redundancy

Both Configuration Manager and Operations Manager feature:
- Multiple management servers for load balancing
- Automatic failover capabilities
- Distributed data processing
- High availability monitoring and management

## Service Dependency Hierarchy

```
Physical Host (SUPERLAB)
├── Network Services (pfSense, VMM01, DHCP01)
├── Core Services
│   ├── Active Directory Domain Services (DC01, DC02)
│   ├── Active Directory Certificate Services (DC01)
│   └── SQL Always On (SQL01, SQL02, SQL03)
├── Application Services (depend on Core Services)
│   ├── System Center Configuration Manager (MECM01, MECM02, WSUS01)
│   ├── System Center Operations Manager (OPSMGR01, OPSMGR02)
│   ├── RDS Farm (RDSCB, RDSGW, RDSLC, RDSSH01, RDSSH02)
│   └── DevOps & Files (DEVOPS01, FS01)
```

## Service Dependencies Summary

1. **SQL Server Always On** provides database services to:
   - Azure DevOps Server
   - System Center Configuration Manager
   - System Center Operations Manager
   - Remote Desktop Services Connection Broker

2. **Active Directory** provides authentication services to all domain-joined systems

3. **WSUS01** serves as the Software Update Point (SUP) for MECM

4. **RDS Farm** depends on the Connection Broker for session management and load balancing

## Network Architecture

The network is segmented into logical zones:

- **External Network:** Internet-facing services through pfSense firewall
- **DMZ:** pfSense firewall providing security boundary
- **Internal Network:** Protected internal services including:
  - Core infrastructure services (AD, DNS, DHCP)
  - Database services (SQL Always On)
  - Application services (RDS, SCOM, MECM)
  - File and DevOps services

## Resource Allocation

The Supermicro server with 255.89 GB RAM and 8TB storage is sufficient to hosts my homelab, allowing me to experiment with enterprise capabilities for learning and testing.
