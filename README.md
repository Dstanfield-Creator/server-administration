# Server Administration

Server deployment, configuration, monitoring, hardening, and maintenance for Linux, Windows Server, and hybrid environments.

## Structure

```
├── docs/                # Deployment guides and walkthroughs
├── configuration/       # Config file examples and templates
├── monitoring/          # Monitoring setup, alerting, dashboards
├── hardening/           # Security hardening and baselines
├── reference/           # Quick reference sheets and checklists
├── CONTRIBUTING.md      # Contribution guidelines
└── LICENSE
```

## Topics

- **Linux Server Admin** — Ubuntu, CentOS, RHEL, Debian
- **Windows Server** — Active Directory, Group Policy, replication
- **Web Servers** — Apache, Nginx, IIS configuration and tuning
- **Database Servers** — MySQL, PostgreSQL, MSSQL installation and admin
- **Application Servers** — Tomcat, Node.js, .NET deployment
- **Virtualization** — Proxmox, VMware, Hyper-V host management
- **Storage & Backups** — NAS, SAN, backup strategies, snapshots
- **Performance Tuning** — CPU, memory, disk optimization
- **Security Hardening** — SSH keys, firewall rules, SELinux, AppArmor
- **Patch Management** — Automated updates, CVE tracking

## Contents

### docs/

- [Windows AD logging baseline for detection](./docs/windows-ad-logging-baseline-for-detection.md) - Advanced Audit Policy, PowerShell and Sysmon logging, event forwarding, and an attack-to-event map for an Active Directory lab

### monitoring/

- [Prometheus node_exporter setup](./monitoring/prometheus-node-exporter-setup.md) - sandboxed node_exporter bound to the management interface, textfile collector for custom metrics, scrape config, and useful PromQL

### hardening/

- [UFW baseline for headless Debian/Ubuntu servers](./hardening/ufw-baseline-linux.md) - default-deny UFW with management-subnet SSH, Tailscale interface allow, SSH rate limiting, and a rollback timer for remote enables
- [OpenSSH server hardening](./hardening/openssh-server-hardening.md) - key-only sshd drop-in with modern crypto, host key cleanup, safe reload, restricted automation keys, and ssh-audit verification
- [systemd service sandboxing](./hardening/systemd-service-sandboxing.md) - confining long-running services with systemd directives, JIT runtime caveats, and scoring with systemd-analyze security

### reference/

- [Proxmox VE CLI cheatsheet](./reference/proxmox-cli-cheatsheet.md) - qm, pct, pvesm, pveum, vzdump, pvesh, task logs, cluster status, and storage housekeeping

## Getting Started

Start with [docs/](./docs/) for your platform, or [hardening/](./hardening/) for security baselines.

---

**Author:** Danny Stanfield · Perth, WA
**License:** MIT
