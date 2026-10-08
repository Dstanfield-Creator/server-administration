# Server Administration

> Hardening, operational guides and reference for Linux, Windows Server and Proxmox hosts. Part of the Infrastructure & Platform department, alongside [lab-ops](https://github.com/Dstanfield-Creator/lab-ops) which holds the lab itself.

**Status:** Active · **Updated:** 2026-10-08

## Structure

```
├── hardening/           # Security baselines for services and hosts
├── guides/              # Operational guides, checklists and troubleshooting
├── reference/           # Quick reference sheets
├── CONTRIBUTING.md      # Contribution guidelines
└── LICENSE
```

## Contents

### hardening/

- [OpenSSH server hardening](./hardening/openssh-server-hardening.md) - key-only sshd drop-in with modern crypto, host key cleanup, safe reload, restricted automation keys, and ssh-audit verification
- [systemd service sandboxing](./hardening/systemd-service-sandboxing.md) - confining long-running services with systemd directives, JIT runtime caveats, and scoring with systemd-analyze security

### guides/

- [Proxmox VE API tokens with least privilege](./guides/proxmox-api-token-least-privilege.md) - custom roles, dedicated users, path-scoped ACLs, protected VMs and safe secret handling for automation tokens
- [New Proxmox VM checklist](./guides/new-proxmox-vm-checklist.md) - tickable checklist from VM settings through network, access, firewall, updates, monitoring, backup and documentation
- [SSH key authentication failures](./guides/ssh-key-auth-failures.md) - classify DNS, DOWN, AUTH, HOSTKEY, HOSTKEY! and agent problems, with the command for each

### reference/

- [Proxmox VE CLI cheatsheet](./reference/proxmox-cli-cheatsheet.md) - qm, pct, pvesm, pveum, vzdump, pvesh, task logs, cluster status, and storage housekeeping

## Moved to other departments

- Host firewall baseline (UFW) and the remote firewall change runbook: [network](https://github.com/Dstanfield-Creator/network/tree/main/firewall)
- node_exporter setup and monitoring stacks: [monitoring](https://github.com/Dstanfield-Creator/monitoring)
- Windows AD logging baseline for detection: [detections](https://github.com/Dstanfield-Creator/detections/blob/main/docs/windows-ad-logging-baseline-for-detection.md)

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
