# Architecture

## Overview

This infrastructure is a centralized, secure, virtualized environment built on **Proxmox VE**, with **OPNsense** providing network security and segmentation, **FreeIPA** as the authoritative identity store, **Authentik** providing SSO/federation on top of FreeIPA, **Wazuh** providing security monitoring/SIEM, and **PatchMon** providing centralized patch management.

## Identity & Authentication Architecture

FreeIPA is the primary source of identities, users, groups, and authentication. Linux VMs are joined to the FreeIPA domain for centralized account and access management.

Authentik integrates with FreeIPA and acts as an authentication/federation layer for applications that can't speak LDAP directly, or that need modern protocols (OIDC/OAuth2, SAML).

```
User → Authentik → FreeIPA
```

This lets services like PatchMon, Wazuh, and OPNsense use centralized authentication while FreeIPA remains the single source of truth for identity.

```mermaid
flowchart LR
    U[User] --> A[Authentik<br/>OIDC / SAML]
    A -->|LDAP| F[FreeIPA<br/>Identity Provider]
    A -.SSO.-> P[PatchMon]
    A -.SSO.-> W[Wazuh]
    A -.SSO.-> O[OPNsense]
    F -->|domain join| L1[Linux VM]
    F -->|domain join| L2[Linux VM]
    F -->|domain join| L3[Linux VM]
```

## Infrastructure Layers

```mermaid
flowchart TB
    subgraph Virtualization
        PVE[Proxmox VE]
    end
    subgraph Network
        OPN[OPNsense<br/>Firewall / Segmentation]
    end
    subgraph Identity
        IPA[FreeIPA]
        AUTH[Authentik]
    end
    subgraph Security
        WAZ[Wazuh SIEM]
        PM[PatchMon]
    end

    PVE --> OPN
    OPN --> IPA
    OPN --> AUTH
    OPN --> WAZ
    OPN --> PM
    AUTH --> IPA
    WAZ -.monitors.-> PVE
    WAZ -.monitors.-> IPA
    WAZ -.monitors.-> AUTH
    PM -.patches.-> PVE
```

## Network Segmentation

> TODO: Document VLANs / subnets once OPNsense segmentation is finalized (e.g. management, identity, services, DMZ).

| Segment | Purpose | VLAN | Subnet |
|---|---|---|---|
| Management | Proxmox, OPNsense admin | | |
| Identity | FreeIPA, Authentik | | |
| Services | PatchMon, Wazuh, apps | | |
| DMZ | Externally-facing services (if any) | | |

## Component Responsibilities

- **Proxmox VE** — hosts and manages all VMs
- **OPNsense** — firewall, routing, inter-segment traffic control
- **FreeIPA** — identity, groups, DNS, Kerberos, domain join for Linux VMs
- **Authentik** — SSO/federation for apps needing OIDC/SAML instead of raw LDAP
- **Wazuh** — log collection, correlation, alerting, SIEM
- **PatchMon** — OS patch visibility and management across all hosts
