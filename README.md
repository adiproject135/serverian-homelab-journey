# Serverian Homelab Journey

A hands-on homelab project documenting the setup, administration, security hardening, and ongoing development of a personal Debian server.

The goal is to build a reliable, remotely manageable Linux environment while learning practical system administration, networking, security, and infrastructure automation.

## Current environment

| Component | Current implementation |
|---|---|
| Server OS | Debian GNU/Linux 13 (Trixie) |
| Hostname | `adrian-server` |
| Desktop environment | KDE Plasma |
| Network management | NetworkManager |
| Remote administration | OpenSSH with Ed25519 public-key authentication |
| Private networking | Tailscale with MagicDNS |
| Host firewall | nftables |
| Privilege management | Restricted administrative helper with a narrow sudoers rule |
| Availability | Persistent systemd sleep/idle inhibitor |

## Implemented work

- Recovered network connectivity after a fresh Debian installation by diagnosing interface state, DHCP, IP reachability, DNS, APT sources, and subnet reachability.
- Configured Wi-Fi through NetworkManager.
- Enabled SSH key-based administration and disabled SSH password and root login.
- Added Tailscale for stable private remote access without router port forwarding.
- Configured nftables with a default-deny inbound policy and explicit SSH access paths.
- Added a restricted administrative helper instead of granting unrestricted passwordless root access.
- Resolved an unnecessary `systemd-networkd-wait-online` boot dependency while retaining NetworkManager.
- Migrated selected KDE preferences from Fedora and retained a pre-migration backup.
- Added a persistent systemd inhibitor to prevent automatic sleep and idle suspension.

Implementation details, decisions, troubleshooting history, verification notes, and known limitations are documented in [the server setup report](docs/2026-11-24-server-setup-report.md).

## Architecture overview

The server is administered remotely from a Fedora workstation. OpenSSH provides shell access using public-key authentication. Tailscale supplies private connectivity and MagicDNS naming, while nftables controls unsolicited inbound traffic on the Debian host.

```text
Fedora workstation
       |
       | Tailscale private network
       | SSH / Ed25519 key
       v
Debian server (adrian-server)
       |
       +-- NetworkManager / Wi-Fi
       +-- nftables host firewall
       +-- Restricted administrative helper
       +-- systemd keep-awake inhibitor
```

## Roadmap

The next development stages are:

1. **Reliability and recovery:** automated security updates, off-server backups, restore testing, and reboot recovery.
2. **Security validation:** sudoers validation, listening-service inventory, Tailscale access review, and log-review procedures.
3. **Monitoring:** resource metrics, disk-space alerts, and service-health visibility.
4. **Configuration management:** version-controlled configuration, reproducible provisioning, and evaluation of Ansible.

These are planned improvements, not claims about the current implementation. See the roadmap in the [full report](docs/2026-11-24-server-setup-report.md).

## Repository structure

```text
.
├── README.md
├── .gitignore
└── docs/
    └── 2026-11-24-server-setup-report.md
```

## Security and publication notes

This repository is intended for public documentation. It deliberately excludes private keys, credentials, authentication tokens, personal backups, and machine-specific network addresses. Configuration examples should be reviewed and sanitized before publication.

The report describes the state observed during the documented setup period. A planned item should only be marked implemented after it has been configured and verified.

## Status

Active learning and infrastructure project. The environment and documentation will evolve as new services, monitoring, backup procedures, and automation are introduced.
