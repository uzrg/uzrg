---
title: Home Lab Setup - HyperV
author: uzrg
date: 2024-03-26 00:34:00 +0800
categories:
  [
    Blogging,
    Tutorial,
    Homelab,
    Virtualization,
    Microsoft,
    HyperV,
    Windows Server,
    Linux
  ]
tags: [Microsoft, HyperV, Windows, Windows Server, Ubuntu]
pin: true
img_path: /posts/20230326
---

# Hyper-V Lab Infrastructure Documentation
## Overview
This documentation describes the comprehensive Hyper-V lab environment running on SUPERLAB, a Supermicro server hosting a complete Microsoft enterprise infrastructure stack.

## Physical Infrastructure
**SUPERLAB Host Server:**
- **Hardware:** Supermicro Super Server
- **Operating System:** Windows Server 2022 Datacenter
- **CPU:** AMD EPYC 3251 8-Core
- **Memory:** 255.89 GB RAM
- **Storage:** 8005.8 GB

## Architecture Diagrams
### Detailed System Architecture
The detailed architecture shows all virtual machines, their roles, and interconnections:

```mermaid
graph TB
    subgraph "Physical Host - SUPERLAB"
        subgraph "Network Infrastructure"
            OPS[OpSense Firewall<br/>2048 MB]
            VMM[VMM01 Server<br/>1102 MB]
            WSUS[WSUS01 Update Server<br/>1148 MB]
        end

        subgraph "Active Directory"
            DC01[DC01 Primary DC<br/>1504 MB]
            DC02[DC02 Secondary DC<br/>1326 MB]
        end

        subgraph "SQL Always On Cluster"
            SQL01[SQL01 Node 1<br/>2530 MB]
            SQL02[SQL02 Node 2<br/>2564 MB]
            SQL03[SQL03 Node 3<br/>2548 MB]
            SQLAGL[SQLAGL01 AG Listener<br/>1148 MB]
            SQLCLU[SQLCLU01 Cluster<br/>1104 MB]
        end

        subgraph "Remote Desktop Services"
            RDSCB[RDSCB Connection Broker<br/>1412 MB]
            RDSGW[RDSGW Gateway<br/>1094 MB]
            RDSLC[RDSLC License Server<br/>1064 MB]
            RDSSH01[RDSSH01 Session Host<br/>1086 MB]
            RDSSH02[RDSSH02 Session Host<br/>1048 MB]
            RDSWEB[RDSWEB Web Access<br/>1058 MB]
        end

        subgraph "System Center"
            MECM01[MECM01 Primary Site<br/>2004 MB]
            MECM02[MECM02 Secondary Site<br/>1126 MB]
            OPSMGR01[OPSMGR01 Mgmt Server<br/>2498 MB]
            OPSMGR02[OPSMGR02 Mgmt Server<br/>1302 MB]
        end

        subgraph "DevOps & File Services"
            DEVOPS[DEVOPS01 Azure DevOps<br/>8152 MB]
            FS01[FS01 File Server<br/>1108 MB]
        end

        subgraph "Linux Systems (Offline)"
            UBUNTU[Ubuntu Server<br/>Status: Off]
            CENTOS[CENTOS01<br/>Status: Off]
        end
    end

    DC01 -.-> DC02
    SQL01 -.-> SQL02
    SQL02 -.-> SQL03
    SQL01 -.-> SQLAGL
    SQL02 -.-> SQLAGL
    SQL03 -.-> SQLAGL
    SQLAGL --> DEVOPS
    SQLAGL --> MECM01
    SQLAGL --> OPSMGR01
    SQLAGL --> RDSCB
    SQLAGL --> WSUS

    RDSCB --> RDSSH01
    RDSCB --> RDSSH02
    RDSCB --> RDSWEB
    RDSCB --> RDSGW
    RDSCB --> RDSLC

    MECM01 --> MECM02
    OPSMGR01 --> OPSMGR02
```

### Network Overview
The network overview diagram illustrates the network topology and service dependencies:

```mermaid
graph LR
    subgraph "External Network"
        WAN[WAN Interface]
    end

    subgraph "SUPERLAB Physical Host"
        EXT_SW[External Virtual Switch]
        INT_SW[Internal Virtual Switch]

        subgraph "DMZ Services"
            OPS_FW[OpSense Firewall]
            RDSGW_DMZ[RDS Gateway]
        end

        subgraph "Internal Network"
            subgraph "Core Services"
                DC_SERVICES[Domain Controllers<br/>DC01 & DC02]
                SQL_CLUSTER[SQL Always On<br/>3-Node Cluster]
                WSUS_SRV[WSUS Server]
            end

            subgraph "Application Services"
                RDS_FARM[RDS Farm<br/>6 Servers]
                SCCM_INFRA[System Center<br/>MECM & OPSMGR]
                DEVOPS_SRV[Azure DevOps Server]
                FILE_SRV[File Server]
            end
        end
    end

    WAN --> EXT_SW
    EXT_SW --> OPS_FW
    EXT_SW --> RDSGW_DMZ
    OPS_FW --> INT_SW
    INT_SW --> DC_SERVICES
    INT_SW --> SQL_CLUSTER
    INT_SW --> WSUS_SRV
    INT_SW --> RDS_FARM
    INT_SW --> SCCM_INFRA
    INT_SW --> DEVOPS_SRV
    INT_SW --> FILE_SRV

    SQL_CLUSTER -.-> DEVOPS_SRV
    SQL_CLUSTER -.-> SCCM_INFRA
    SQL_CLUSTER -.-> RDS_FARM
    SQL_CLUSTER -.-> WSUS_SRV
    DC_SERVICES -.-> SQL_CLUSTER
    DC_SERVICES -.-> RDS_FARM
    DC_SERVICES -.-> SCCM_INFRA
    DC_SERVICES -.-> DEVOPS_SRV
```

## Infrastructure Components
### Network & Security Infrastructure
- **OpSense:** Firewall/Router (2048 MB)
- **VMM01:** VMM Server (1102 MB)
- **WSUS01:** Update Server (1148 MB)

### Active Directory Domain Services
- **Domain:** myhomelab.hv.lab
- **DC01:** Primary Domain Controller (1504 MB)
- **DC02:** Secondary Domain Controller (1326 MB)

### SQL Server Always On Availability Group
- **SQL01:** SQL Server Node 1 (2530 MB)
- **SQL02:** SQL Server Node 2 (2564 MB)
- **SQL03:** SQL Server Node 3 (2548 MB)

### SQL Support Services
- **SQLAGL01:** AG Listener (1148 MB)
- **SQLCLU01:** Failover Cluster (1104 MB)
- **CAUSQLCL4bu:** Cluster Virtual Network

### Hosted Databases
- **Configuration Management:** AzureDevOps_Configuration, AzureDevOps_myhomelab, CM_MHL
- **Operations Management:** OperationsManager, OperationsManagerDW, RDSCB_DB
- **Update Services:** SUSDB (Windows Update Services)

### Remote Desktop Services Farm
- **RDSCB:** Connection Broker (1412 MB)
- **RDSGW:** RDS Gateway (1094 MB)
- **RDSLC:** License Server (1064 MB)

### RDS Session Hosts
- **RDSSH01:** Session Host 1 (1086 MB)
- **RDSSH02:** Session Host 2 (1048 MB)
- **RDSWEB:** Web Access (1058 MB)

### System Center Configuration Manager
- **MECM01:** MECM Primary Site (2004 MB)
- **MECM02:** MECM Secondary Site (1126 MB)

### System Center Operations Manager
- **OPSMGR01:** Management Server 1 (2498 MB)
- **OPSMGR02:** Management Server 2 (1302 MB)

### DevOps & File Services
- **DEVOPS01:** Azure DevOps Server (8152 MB)
- **FS01:** File Server (1108 MB)

### Offline Systems
- **UBUNTU:** Ubuntu Server (Status: Off)
- **CENTOS01:** CentOS Server (Status: Off)

## Network Architecture
### Physical Network Interfaces
- **WAN Interface:** External connectivity
- **LAN Interface:** Internal network access

### Virtual Network Infrastructure
- **External Virtual Switch:** WAN access for external-facing services
- **Internal Virtual Switch:** LAN trunk for internal communication

### Service Dependencies
SQL Server Always On Availability Group provides database services to:
- Azure DevOps Server
- System Center Configuration Manager
- System Center Operations Manager
- Remote Desktop Services Connection Broker
- Windows Server Update Services

## High Availability Features
### SQL Server Always On
- Three-node availability group with automatic failover
- Cluster support services for enhanced reliability
- Distributed database hosting for critical applications

### Active Directory Replication
- Dual domain controllers with automatic replication
- Redundant DNS and authentication services

### System Center Redundancy
- Multiple management servers for both Configuration Manager and Operations Manager
- Load balancing and failover capabilities

## Management and Monitoring
The infrastructure includes comprehensive management through:
- **System Center Operations Manager:** Infrastructure monitoring
- **System Center Configuration Manager:** System deployment and updates
- **Windows Server Update Services:** Centralized patching
- **Azure DevOps Server:** Development lifecycle management

## Notes
The environment demonstrates enterprise-grade Microsoft technologies with proper redundancy and high availability configurations.
