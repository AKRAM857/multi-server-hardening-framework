# Multi-Server Hardening Framework: Threat Modeling, Role-Based Firewall Policy, and Ansible-Based Automation

> **Securing a multi-server Linux infrastructure through threat-driven design, role-based nftables firewall policy, and Ansible automation.**

A production-inspired infrastructure project that translates a per-server threat model into enforced, role-specific firewall policy — deployed, validated, and reproduced automatically across a full server fleet using Ansible.

The objective is not simply to write firewall rules, but to demonstrate how secure infrastructure is *engineered*: every rule in this project is derived from a documented threat, applied consistently through automation, and verified against the live system rather than assumed correct.

---

## Overview

Manually configuring security on each server individually does not scale, and it does not stay correct — configuration drifts, exceptions accumulate, and no two servers end up secured the same way.

This project implements a threat-driven security engineering methodology across three server roles — a Management Server acting as an Ansible control node, a Web Server, and a Database Server — where every firewall rule is traceable back to a specific, documented threat, and every server's configuration is enforced automatically rather than maintained by hand.

The environment combines Linux networking, nftables, Ansible, and role-based access design to demonstrate how a small, threat-modeled infrastructure can be hardened and kept consistent through automation, rather than one-off manual configuration.

---

## Security Objectives

The project was built with the following objectives:

- Centralize infrastructure administration through a single, hardened Ansible control node.
- Derive every firewall rule from an explicit, per-server threat model — not convention or default configuration.
- Enforce role-specific, deny-by-default network policy using nftables.
- Automate configuration deployment and validation using Ansible, eliminating manual drift.
- Contain the impact of a single server's compromise through defense-in-depth layering.
- Document every engineering decision, trade-off, and scope boundary throughout the implementation.

---

## System Architecture

The infrastructure is designed around a single principle:

> **No server trusts a connection it cannot justify against a documented threat.**

The Management Server is the only system authorized to alter the Web Server and Database Server; all configuration flows outward from this single control point, never in reverse. Each managed server's firewall policy is derived independently from its own operational role, rather than a shared, generic ruleset.

<p align="center">
  <img src="diagrams/architecture.png" alt="Infrastructure Overview" width="900">
</p>

*(Place the architecture diagram — Administrator → Management Server (Ansible Control Node) → Web Server / Database Server, with trust zones and interface labeling — at `diagrams/architecture.png`.)*

---

## Key Features

| Category | Implementation |
|-----------|----------------|
| Infrastructure Automation | Ansible (roles, templates, variables) |
| Firewall Policy | nftables, role-based and per-interface scoped |
| Security Methodology | Per-server threat modeling (asset-centric, STRIDE-informed) |
| Configuration Management | Jinja2 templates, group-scoped variables |
| Access Control | SSH key-based authentication, dedicated service account |
| Privilege Management | Scoped `sudo` via `sudoers.d`, no direct root login |
| Network Segmentation | Per-interface firewall scoping (`iifname`/`oifname`) |
| Operating System | Ubuntu Server |

---

## Defense Layers

The infrastructure follows a layered security model where multiple mechanisms cooperate instead of relying on a single control.

| Layer | Technology |
|--------|------------|
| Perimeter / Access | SSH Public Keys, key-only authentication |
| Network Filtering | nftables, deny-by-default, per-role and per-interface |
| Host Hardening | Minimal package footprint, no direct root login |
| Automation | Ansible roles, templates, and idempotent deployment |

---

## Implemented Components

The current implementation includes the following engineering components:

| Component | Status |
|-----------|:------:|
| Threat Modeling Methodology | ✅ |
| Management Server (Ansible Control Node) | ✅ |
| Role-Based nftables Firewall Policy | ✅ |
| Web Server Hardening | ✅ |
| Database Server Hardening | ✅ |
| Ansible Automation (Roles, Templates, Variables) | ✅ |
| Per-Interface Firewall Scoping | ✅ |
| Deployment & Access-Control Validation (`nft`, `nmap`) | ✅ |
| Technical Documentation | ✅ |
| Architecture Diagrams | 🚧 |

---

## Repository Structure

```text
.
├── inventory/       # Ansible inventory and group_vars (connection + policy variables)
├── roles/           # Ansible role: firewalls (tasks, templates, handlers)
├── scripts/         # Setup, deployment, and validation command reference
├── diagrams/        # Architecture diagrams
├── docs/            # Project documentation and technical report
├── troubleshooting.md  # Real errors encountered, root cause, and fix
├── site.yml         # Top-level playbook
└── README.md
```

---

## Documentation

This repository focuses on documenting both the implementation and the reasoning behind the infrastructure.

Current documentation covers:

- Security engineering methodology and threat modeling process
- Infrastructure and trust-zone architecture
- Per-server firewall policy and its derivation from threat analysis
- Ansible project structure, variable scoping, and deployment
- Deployment and validation results per server
- Known limitations and explicitly out-of-scope decisions

The documentation will continue to evolve as the project grows.

---

## Future Enhancements

The current implementation provides the foundation of the infrastructure. Future improvements may include:

- VPN-gated administrative access (WireGuard), removing the Management Server's direct internet exposure
- Network-layer segmentation (NAT gateway, DMZ) as a built and defended subsystem
- Centralized logging and audit accountability
- Automated deployment pipeline for application code, mediated through the same control node

---

## Project Status

Current Stage: Core Infrastructure Complete

Threat modeling, firewall policy, and Ansible automation have been fully implemented and validated across all three servers.

The remaining work focuses on architecture diagrams and continued documentation refinement.

---

## License

This project is released under the MIT License.

See the [LICENSE](LICENSE) file for more information.
