---
title: Home Lab Setup - HyperV
author: uzrg
date: 2024-03-26 00:34:00 +0800
categories: [Blogging, Tutorial, Homelab, Virtualization, Microsoft, HyperV, Windows Server, Linux]
tags: [Microsoft, HyperV, Windows, Windows Server, Ubuntu]
pin: true
mermaid: true
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

> ** Tip:** The SVG diagrams below can be copied and viewed in external tools like <a href="https://www.svgviewer.dev/" target="_blank">SVGViewer.dev</a> for better viewing, editing, or saving capabilities.

### Detailed System Architecture

<div style="text-align: center; margin: 20px 0;">
  <button onclick="copySVG('system-architecture-svg')" style="margin-bottom: 10px; padding: 8px 16px; background: #667eea; color: white; border: none; border-radius: 4px; cursor: pointer;">Copy SVG Code</button>

  <svg id="system-architecture-svg" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1400 1000" style="width: 100%; height: auto; max-width: 1400px; background: white; border: 2px solid #333; border-radius: 10px;">
    <defs>
      <linearGradient id="headerGradient" x1="0%" y1="0%" x2="100%" y2="0%">
        <stop offset="0%" stop-color="#667eea"/>
        <stop offset="100%" stop-color="#764ba2"/>
      </linearGradient>
      <marker id="arrow" markerWidth="10" markerHeight="10" refX="9" refY="3" orient="auto">
        <path d="M0,0 L0,6 L9,3 z" fill="#333"/>
      </marker>
    </defs>

    <!-- Main Container -->
    <rect x="20" y="20" width="1360" height="960" rx="15" fill="url(#headerGradient)" stroke="#333" stroke-width="3"/>
    <text x="700" y="60" text-anchor="middle" font-family="Arial, sans-serif" font-size="36" fill="white">Physical Host - SUPERLAB</text>

    <!-- Network Infrastructure -->
    <rect x="100" y="100" width="550" height="120" rx="10" fill="#e3f2fd" stroke="#64b5f6" stroke-width="2"/>
    <text x="375" y="130" text-anchor="middle" font-family="Arial, sans-serif" font-size="20" font-weight="bold">Network Infrastructure</text>

    <rect x="130" y="150" width="140" height="50" rx="5" fill="#bbdefb"/>
    <text x="200" y="180" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">OpSense</text>

    <rect x="290" y="150" width="140" height="50" rx="5" fill="#bbdefb"/>
    <text x="360" y="180" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">VMM01</text>

    <rect x="450" y="150" width="140" height="50" rx="5" fill="#bbdefb"/>
    <text x="520" y="180" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">DHCP01</text>

    <!-- Active Directory -->
    <rect x="750" y="100" width="550" height="120" rx="10" fill="#e8f5e9" stroke="#66bb6a" stroke-width="2"/>
    <text x="1025" y="130" text-anchor="middle" font-family="Arial, sans-serif" font-size="20" font-weight="bold">Active Directory</text>

    <rect x="800" y="150" width="220" height="50" rx="5" fill="#c8e6c9"/>
    <text x="910" y="180" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">DC01 Primary DC</text>

    <rect x="1040" y="150" width="220" height="50" rx="5" fill="#c8e6c9"/>
    <text x="1150" y="180" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">DC02 Secondary DC</text>

    <!-- SQL Cluster -->
    <rect x="100" y="250" width="550" height="280" rx="10" fill="#f3e5f5" stroke="#ab47bc" stroke-width="2"/>
    <text x="375" y="280" text-anchor="middle" font-family="Arial, sans-serif" font-size="20" font-weight="bold">SQL Always On Cluster</text>

    <rect x="130" y="310" width="140" height="50" rx="5" fill="#e1bee7"/>
    <text x="200" y="340" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">SQL01</text>

    <rect x="290" y="310" width="140" height="50" rx="5" fill="#e1bee7"/>
    <text x="360" y="340" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">SQL02</text>

    <rect x="450" y="310" width="140" height="50" rx="5" fill="#e1bee7"/>
    <text x="520" y="340" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">SQL03</text>

    <rect x="180" y="390" width="140" height="50" rx="5" fill="#ce93d8"/>
    <text x="250" y="420" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">SQLCLU01</text>

    <rect x="360" y="390" width="140" height="50" rx="5" fill="#ce93d8"/>
    <text x="430" y="420" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">SQLAGL01</text>

    <!-- Connection lines for SQL Cluster -->
    <line x1="200" y1="360" x2="250" y2="390" stroke="#333" stroke-width="2"/>
    <line x1="360" y1="360" x2="320" y2="390" stroke="#333" stroke-width="2"/>
    <line x1="360" y1="360" x2="400" y2="390" stroke="#333" stroke-width="2"/>
    <line x1="520" y1="360" x2="470" y2="390" stroke="#333" stroke-width="2"/>

    <!-- System Center -->
    <rect x="750" y="250" width="550" height="280" rx="10" fill="#fbe9e7" stroke="#ff8a65" stroke-width="2"/>
    <text x="1025" y="280" text-anchor="middle" font-family="Arial, sans-serif" font-size="20" font-weight="bold">System Center</text>

    <rect x="780" y="310" width="200" height="50" rx="5" fill="#ffccbc"/>
    <text x="880" y="340" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">MECM01</text>

    <rect x="1020" y="310" width="200" height="50" rx="5" fill="#ffccbc"/>
    <text x="1120" y="340" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">MECM02</text>

    <rect x="780" y="380" width="200" height="50" rx="5" fill="#ffab91"/>
    <text x="880" y="410" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">OPSMGR01</text>

    <rect x="1020" y="380" width="200" height="50" rx="5" fill="#ffab91"/>
    <text x="1120" y="410" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">OPSMGR02</text>

    <rect x="900" y="460" width="200" height="50" rx="5" fill="#ff8a65"/>
    <text x="1000" y="490" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">WSUS01</text>

    <!-- RDS Farm -->
    <rect x="100" y="560" width="550" height="200" rx="10" fill="#fff3e0" stroke="#ffb74d" stroke-width="2"/>
    <text x="375" y="590" text-anchor="middle" font-family="Arial, sans-serif" font-size="20" font-weight="bold">Remote Desktop Services</text>

    <rect x="130" y="620" width="140" height="50" rx="5" fill="#ffe0b2"/>
    <text x="200" y="650" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">RDSCB</text>

    <rect x="290" y="620" width="140" height="50" rx="5" fill="#ffe0b2"/>
    <text x="360" y="650" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">RDSGW</text>

    <rect x="450" y="620" width="140" height="50" rx="5" fill="#ffe0b2"/>
    <text x="520" y="650" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">RDSLC</text>

    <rect x="210" y="690" width="140" height="50" rx="5" fill="#ffcc80"/>
    <text x="280" y="720" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">RDSSH01</text>

    <rect x="370" y="690" width="140" height="50" rx="5" fill="#ffcc80"/>
    <text x="440" y="720" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">RDSSH02</text>

    <!-- DevOps & File Services -->
    <rect x="750" y="560" width="550" height="150" rx="10" fill="#e0f7fa" stroke="#4dd0e1" stroke-width="2"/>
    <text x="1025" y="590" text-anchor="middle" font-family="Arial, sans-serif" font-size="20" font-weight="bold">DevOps & File Services</text>

    <rect x="825" y="630" width="180" height="50" rx="5" fill="#b2ebf2"/>
    <text x="915" y="660" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">DEVOPS01</text>

    <rect x="1045" y="630" width="180" height="50" rx="5" fill="#b2ebf2"/>
    <text x="1135" y="660" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">FS01</text>

    <!-- Linux Systems -->
    <rect x="100" y="790" width="550" height="150" rx="10" fill="#e8eaf6" stroke="#7986cb" stroke-width="2"/>
    <text x="375" y="820" text-anchor="middle" font-family="Arial, sans-serif" font-size="20" font-weight="bold">Linux Systems</text>

    <rect x="170" y="860" width="180" height="60" rx="5" fill="#c5cae9"/>
    <text x="260" y="885" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">UBUNTU</text>
    <text x="260" y="905" text-anchor="middle" font-family="Arial, sans-serif" font-size="12">Status: Off</text>

    <rect x="410" y="860" width="180" height="60" rx="5" fill="#c5cae9"/>
    <text x="500" y="885" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">CENTOS01</text>
    <text x="500" y="905" text-anchor="middle" font-family="Arial, sans-serif" font-size="12">Status: Off</text>

    <!-- Replication line between DCs -->
    <line x1="910" y1="200" x2="1150" y2="200" stroke="#333" stroke-width="2" stroke-dasharray="5,5" marker-end="url(#arrow)"/>
    <text x="1030" y="195" text-anchor="middle" font-family="Arial, sans-serif" font-size="12">Replication</text>

    <!-- Cluster connections -->
    <line x1="200" y1="360" x2="360" y2="360" stroke="#333" stroke-width="2" stroke-dasharray="5,5"/>
    <line x1="360" y1="360" x2="520" y2="360" stroke="#333" stroke-width="2" stroke-dasharray="5,5"/>
    <text x="360" y="355" text-anchor="middle" font-family="Arial, sans-serif" font-size="12">Cluster</text>
  </svg>
</div>

### Network Overview

<div style="text-align: center; margin: 20px 0;">
  <button onclick="copySVG('network-overview-svg')" style="margin-bottom: 10px; padding: 8px 16px; background: #667eea; color: white; border: none; border-radius: 4px; cursor: pointer;">Copy SVG Code</button>

  <svg id="network-overview-svg" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1400 800" style="width: 100%; height: auto; max-width: 1400px; background: white; border: 2px solid #333; border-radius: 10px;">
    <defs>
      <linearGradient id="headerGradient2" x1="0%" y1="0%" x2="100%" y2="0%">
        <stop offset="0%" stop-color="#667eea"/>
        <stop offset="100%" stop-color="#764ba2"/>
      </linearGradient>
      <marker id="arrow2" markerWidth="10" markerHeight="10" refX="9" refY="3" orient="auto">
        <path d="M0,0 L0,6 L9,3 z" fill="#333"/>
      </marker>
    </defs>

    <!-- Main Container -->
    <rect x="20" y="20" width="1360" height="760" rx="15" fill="url(#headerGradient2)" stroke="#333" stroke-width="3"/>
    <text x="700" y="60" text-anchor="middle" font-family="Arial, sans-serif" font-size="36" fill="white">Network Overview</text>

    <!-- External Network -->
    <rect x="50" y="100" width="250" height="80" rx="10" fill="#ffebee" stroke="#ef9a9a" stroke-width="2"/>
    <text x="175" y="150" text-anchor="middle" font-family="Arial, sans-serif" font-size="18">WAN Interface</text>

    <!-- SUPERLAB Host -->
    <rect x="350" y="80" width="1000" height="650" rx="10" fill="#e3f2fd" stroke="#64b5f6" stroke-width="2"/>
    <text x="850" y="120" text-anchor="middle" font-family="Arial, sans-serif" font-size="24" font-weight="bold">SUPERLAB Physical Host</text>

    <!-- External Switch -->
    <rect x="400" y="150" width="200" height="70" rx="5" fill="#bbdefb"/>
    <text x="500" y="190" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">External Virtual Switch</text>

    <!-- Internal Switch -->
    <rect x="400" y="240" width="200" height="70" rx="5" fill="#bbdefb"/>
    <text x="500" y="280" text-anchor="middle" font-family="Arial, sans-serif" font-size="14">Internal Virtual Switch</text>

    <!-- DMZ Services -->
    <rect x="650" y="140" width="250" height="90" rx="10" fill="#ffebee" stroke="#ef9a9a" stroke-width="2"/>
    <text x="775" y="170" text-anchor="middle" font-family="Arial, sans-serif" font-size="16">DMZ Services</text>
    <rect x="675" y="190" width="200" height="30" fill="#ffcdd2"/>
    <text x="775" y="210" text-anchor="middle" font-family="Arial, sans-serif" font-size="12">OpSense Firewall</text>

    <!-- Internal Network -->
    <rect x="400" y="350" width="900" height="350" rx="10" fill="#e8f5e9" stroke="#66bb6a" stroke-width="2"/>
    <text x="850" y="380" text-anchor="middle" font-family="Arial, sans-serif" font-size="20" font-weight="bold">Internal Network</text>

    <!-- Core Services -->
    <rect x="430" y="410" width="400" height="250" rx="10" fill="#f3e5f5" stroke="#ab47bc" stroke-width="1"/>
    <text x="630" y="440" text-anchor="middle" font-family="Arial, sans-serif" font-size="16">Core Services</text>

    <rect x="450" y="460" width="360" height="50" rx="5" fill="#e1bee7"/>
    <text x="630" y="485" text-anchor="middle" font-family="Arial, sans-serif" font-size="12">Domain Controllers</text>
    <text x="630" y="500" text-anchor="middle" font-family="Arial, sans-serif" font-size="10">DC01 & DC02</text>

    <rect x="450" y="530" width="360" height="50" rx="5" fill="#e1bee7"/>
    <text x="630" y="555" text-anchor="middle" font-family="Arial, sans-serif" font-size="12">SQL Always On</text>
    <text x="630" y="570" text-anchor="middle" font-family="Arial, sans-serif" font-size="10">3-Node Cluster</text>

    <rect x="450" y="600" width="360" height="50" rx="5" fill="#e1bee7"/>
    <text x="630" y="630" text-anchor="middle" font-family="Arial, sans-serif" font-size="12">WSUS Server</text>

    <!-- Application Services -->
    <rect x="870" y="410" width="400" height="250" rx="10" fill="#fff3e0" stroke="#ffb74d" stroke-width="1"/>
    <text x="1070" y="440" text-anchor="middle" font-family="Arial, sans-serif" font-size="16">Application Services</text>

    <rect x="890" y="460" width="360" height="50" rx="5" fill="#ffe0b2"/>
    <text x="1070" y="485" text-anchor="middle" font-family="Arial, sans-serif" font-size="12">RDS Farm</text>
    <text x="1070" y="500" text-anchor="middle" font-family="Arial, sans-serif" font-size="10">5 Servers</text>

    <rect x="890" y="530" width="360" height="50" rx="5" fill="#ffe0b2"/>
    <text x="1070" y="555" text-anchor="middle" font-family="Arial, sans-serif" font-size="12">System Center</text>
    <text x="1070" y="570" text-anchor="middle" font-family="Arial, sans-serif" font-size="10">MECM & OPSMGR</text>

    <rect x="890" y="600" width="360" height="50" rx="5" fill="#ffe0b2"/>
    <text x="1070" y="630" text-anchor="middle" font-family="Arial, sans-serif" font-size="12">Azure DevOps</text>

    <!-- Connection lines -->
    <line x1="175" y1="180" x2="400" y2="180" stroke="#333" stroke-width="3" marker-end="url(#arrow2)"/>
    <text x="287" y="175" text-anchor="middle" font-family="Arial, sans-serif" font-size="12">Internet</text>

    <line x1="600" y1="180" x2="650" y2="180" stroke="#333" stroke-width="3" marker-end="url(#arrow2)"/>
    <text x="625" y="175" text-anchor="middle" font-family="Arial, sans-serif" font-size="12">External</text>

    <line x1="775" y1="230" x2="775" y2="350" stroke="#333" stroke-width="3" marker-end="url(#arrow2)"/>
    <text x="725" y="290" text-anchor="middle" font-family="Arial, sans-serif" font-size="12">Filtered</text>

    <line x1="500" y1="310" x2="630" y2="460" stroke="#333" stroke-width="2" marker-end="url(#arrow2)"/>
    <line x1="500" y1="310" x2="630" y2="530" stroke="#333" stroke-width="2" marker-end="url(#arrow2)"/>
    <line x1="500" y1="310" x2="630" y2="600" stroke="#333" stroke-width="2" marker-end="url(#arrow2)"/>
    <line x1="500" y1="310" x2="1070" y2="460" stroke="#333" stroke-width="2" marker-end="url(#arrow2)"/>
    <line x1="500" y1="310" x2="1070" y2="530" stroke="#333" stroke-width="2" marker-end="url(#arrow2)"/>
    <line x1="500" y1="310" x2="1070" y2="600" stroke="#333" stroke-width="2" marker-end="url(#arrow2)"/>
  </svg>
</div>

<script>
function copySVG(elementId) {
  const svgElement = document.getElementById(elementId);
  const svgData = svgElement.outerHTML;

  if (navigator.clipboard) {
    navigator.clipboard.writeText(svgData).then(() => {
      // Show success message
      const button = event.target;
      const originalText = button.textContent;
      button.textContent = 'âœ… Copied!';
      button.style.background = '#4caf50';

      setTimeout(() => {
        button.textContent = originalText;
        button.style.background = '#667eea';
      }, 2000);
    }).catch(err => {
      console.error('Failed to copy SVG: ', err);
      fallbackCopy(svgData);
    });
  } else {
    fallbackCopy(svgData);
  }
}

function fallbackCopy(text) {
  const textArea = document.createElement('textarea');
  textArea.value = text;
  document.body.appendChild(textArea);
  textArea.select();
  try {
    document.execCommand('copy');
    const button = event.target;
    const originalText = button.textContent;
    button.textContent = 'âœ… Copied!';
    button.style.background = '#4caf50';

    setTimeout(() => {
      button.textContent = originalText;
      button.style.background = '#667eea';
    }, 2000);
  } catch (err) {
    console.error('Fallback copy failed: ', err);
  }
  document.body.removeChild(textArea);
}
</script>

## Infrastructure Components

### Network & Security Infrastructure

- **OpSense:** Firewall/Router providing network security and traffic filtering
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

### System Center Configuration Manager

Enterprise system management and software deployment:

- **MECM01:** MECM Primary Site Server managing the entire infrastructure
- **MECM02:** MECM Secondary Site Server providing distributed management

### System Center Operations Manager

Infrastructure monitoring and alerting:

- **OPSMGR01:** Management Server 1 providing monitoring services
- **OPSMGR02:** Management Server 2 providing redundancy and load balancing

### Windows Server Update Services

- **WSUS01:** Update Server managing Windows updates and serving as Software Update Point (SUP) for MECM

### DevOps & File Services

- **DEVOPS01:** Azure DevOps Server providing source control, build, and deployment services
- **FS01:** File Server providing centralized file storage and sharing

### Offline Systems

- **UBUNTU:** Ubuntu Server (Status: Off) - Available for Linux workloads
- **CENTOS01:** CentOS Server (Status: Off) - Available for Linux workloads

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
- Geographic distribution of directory services

### System Center Redundancy

Both Configuration Manager and Operations Manager feature:
- Multiple management servers for load balancing
- Automatic failover capabilities
- Distributed data processing
- High availability monitoring and management

## Service Dependencies

The infrastructure is designed with clear service dependencies:

1. **SQL Server Always On** provides database services to:
   - Azure DevOps Server
   - System Center Configuration Manager
   - System Center Operations Manager
   - Remote Desktop Services Connection Broker
   - Windows Server Update Services

2. **Active Directory** provides authentication services to all domain-joined systems

3. **WSUS01** serves as the Software Update Point for MECM, creating an integrated patch management solution

4. **RDS Farm** depends on the Connection Broker for session management and load balancing

## Network Architecture

The network is segmented into logical zones:

- **External Network:** Internet-facing services through OpSense firewall
- **DMZ:** OpSense firewall providing security boundary
- **Internal Network:** Protected internal services including:
  - Core infrastructure services (AD, DNS, DHCP)
  - Database services (SQL Always On)
  - Application services (RDS, System Center)
  - File and DevOps services

## Resource Allocation

The Supermicro server with 255.89 GB RAM and 8TB storage efficiently hosts my homelab infrastructure, allowing me to experiment with enterprise capabilities for continuous learning.
