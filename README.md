# Centralized Secure Infrastructure

A centralized, secure, and virtualized IT infrastructure built on Proxmox VE, with network security, centralized identity management, security monitoring, and automated patch management.

## Components

| Component | Role |
|---|---|
| **[Proxmox VE](./proxmox/)** | Virtualization platform — hosts all VMs and infrastructure services |
| **[OPNsense](./opnsense/)** | Firewall and network gateway — traffic filtering, routing, segmentation |
| **[FreeIPA](./freeipa/)** | Centralized identity and access management — primary Identity Provider and domain for Linux VMs |
| **[Authentik](./authentik/)** | Identity federation / SSO layer for apps that don't speak LDAP natively |
| **[Wazuh](./wazuh/)** | Security monitoring and SIEM — log collection, correlation, threat detection |
| **[PatchMon](./patchmon/)** | Centralized OS patch management and update tracking |
| **[NetBox](./netbox/)** | IPAM today (IPv4/IPv6 prefixes, subnets, VRFs) — room to grow into full DCIM/network source-of-truth |

## Identity & Authentication Flow

```
User → Authentik → FreeIPA
```

FreeIPA is the single authoritative source of identities, users, and groups. Authentik retrieves identities from FreeIPA via LDAP and exposes modern authentication protocols (OIDC/OAuth2, SAML) to services that need them — including PatchMon, Wazuh, and OPNsense — so nothing needs its own separate credential store.

See [`docs/architecture.md`](./docs/architecture.md) for the full architecture writeup and diagram.

## Repository Structure

```
.
├── docs/                 # Architecture docs, diagrams, decisions (ADRs)
├── proxmox/              # VM templates, cloud-init configs
├── opnsense/             # Firewall rules, network config exports
├── freeipa/              # Domain config, DNS
├── authentik/            # Blueprints, provider configs
├── wazuh/                # Detection rules, agent configs
├── patchmon/             # Patch policies
├── netbox/               # IPAM config, VRF/VLAN layout (future: DCIM)
└── ansible/              # Configuration management playbooks
```

## Goals

- Centralized virtualization through Proxmox VE
- Network security and segmentation through OPNsense
- Centralized identity and domain management through FreeIPA
- Centralized authentication and SSO through Authentik
- Integration of Linux VMs into the FreeIPA domain
- Centralized security monitoring / SIEM through Wazuh
- Centralized OS patch management through PatchMon
- Centralized IP address management (and future network source-of-truth) through NetBox
- Reduced reliance on separate per-service credentials
- Improved visibility and traceability of user activity and security events
- A modular architecture extensible with future services

## Status

🚧 Work in progress.
