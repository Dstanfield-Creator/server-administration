# UFW Baseline for Headless Debian/Ubuntu Servers

> Default-deny host firewall with UFW: management-subnet SSH, per-service rules, a trusted Tailscale interface, SSH rate limiting, logging, and a rollback timer so a remote enable cannot lock you out.

**Status:** Active · **Updated:** 2026-10-08

## Scope

Headless Debian 12 or Ubuntu 22.04/24.04 administered over SSH. UFW is a front end for nftables/iptables; this baseline covers the UFW layer only.

| Placeholder | Meaning |
|---|---|
| `192.0.2.0/24` | Management subnet allowed to reach SSH |
| `192.0.2.10` | A single monitoring host (Prometheus) |
| `192.0.2.50` | The server being firewalled |
| `tailscale0` | Tailscale interface (check with `ip -br link`) |

## Before Enabling: Arm a Rollback Timer

Enabling UFW over an existing SSH session can freeze that session. UFW's default ruleset drops packets in conntrack state INVALID, and an established flow that the freshly loaded state table has not tracked can be classified exactly that way. On a headless box there is no console to fall back to. Arm a transient systemd timer that disables UFW if you do not come back and cancel it:

```bash
sudo systemd-run --unit=fw-deadman --on-active=180 /usr/sbin/ufw disable
```

Workflow:

1. Arm the timer (180 seconds is enough to enable and test). `systemctl list-timers fw-deadman.timer` confirms it.
2. Enable UFW.
3. Open a NEW SSH session from the management subnet. Do not trust the old one.
4. If the new session works: `sudo systemctl stop fw-deadman.timer`.
5. If it does not: wait, UFW disables itself, fix the rules, start again from step 1.

If step 4 is forgotten, UFW is disabled after 180 seconds and the host is back to its pre-change state. That is the intended failure mode. Re-arm the timer for every later change that touches port 22.

## Baseline Rules

```bash
sudo apt update && sudo apt install -y ufw

# Defaults
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH only from the management subnet, plus rate limiting
sudo ufw allow from 192.0.2.0/24 to any port 22 proto tcp comment 'SSH from mgmt'
sudo ufw limit 22/tcp comment 'SSH rate limit'

# Tailscale: trust the overlay interface; access control lives in the tailnet ACLs
sudo ufw allow in on tailscale0 comment 'Tailscale'

# Logging: low logs blocked packets; medium adds allowed matches and is noisy
sudo ufw logging low
```

`ufw limit` denies a source that opens 6 or more connections in 30 seconds. It is a brute-force speed bump, not a substitute for key-only authentication. The allow rule and the limit rule coexist: the first permits the subnet, the second throttles anything that still matches.

## Per-Service Rules

Add only what the host serves, and restrict by source wherever the client set is known. Packages may ship profiles under `/etc/ufw/applications.d` (`ufw app list`, then `ufw allow 'Nginx Full'`).

```bash
# Web server, public
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# node_exporter, only from the Prometheus server
sudo ufw allow from 192.0.2.10 to any port 9100 proto tcp comment 'node_exporter'
```

## Enable, Test From a New Session, Disarm

```bash
sudo systemd-run --unit=fw-deadman --on-active=180 /usr/sbin/ufw disable
sudo ufw enable
```

From a NEW terminal on the management subnet:

```bash
ssh admin@192.0.2.50 'sudo systemctl stop fw-deadman.timer && sudo ufw status | head -1'
```

## Managing Rules by Number

```bash
sudo ufw status numbered
```

```text
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    192.0.2.0/24    # SSH from mgmt
[ 2] 22/tcp                     LIMIT IN    Anywhere        # SSH rate limit
[ 3] Anywhere on tailscale0     ALLOW IN    Anywhere        # Tailscale
[ 4] 80/tcp                     ALLOW IN    Anywhere        # HTTP
```

Delete by number and re-list after every deletion, because the numbering shifts:

```bash
sudo ufw delete 4
sudo ufw status numbered
```

UFW evaluates rules top to bottom and the first match wins. `sudo ufw insert 1 deny from 192.0.2.99 to any` places a deny above the existing allows.

## IPv6

UFW manages IPv6 when `IPV6=yes` is set in `/etc/default/ufw`, which is the Debian and Ubuntu default. Every rule without an explicit address family is added for both v4 and v6, which is why `ufw status` shows `(v6)` twins. If the host has no global IPv6 address, leaving it enabled is harmless and avoids an unfiltered `ip6tables`. Changing `IPV6=` requires `ufw disable && ufw enable` to take effect.

Rules with a v4 source such as `from 192.0.2.0/24` are v4-only. Add a matching v6 rule if the management network has a v6 prefix:

```bash
sudo ufw allow from 2001:db8:100::/64 to any port 22 proto tcp comment 'SSH from mgmt v6'
```

## Verification

On the host, confirm the listener set matches the rule set:

```bash
ss -tlnp                       # TCP listeners with owning process (ss -ulnp for UDP)
sudo ufw show listening        # UFW's view, with the rule that covers each port
```

From a host on the management subnet, then from a host outside it:

```bash
nmap -Pn -p 22,80,443,9100 192.0.2.50      # 22 and 9100 open from mgmt only
sudo nmap -Pn -sS -p- 192.0.2.50            # full TCP sweep
```

Review recent blocks with `sudo journalctl -k --since "10 min ago" | grep 'UFW BLOCK'`. To start again from nothing, `sudo ufw reset` disables UFW and moves the old rules to `/etc/ufw/*.rules.<timestamp>`.

## Related

- [OpenSSH server hardening](./openssh-server-hardening.md)
- [Prometheus node_exporter setup](../monitoring/prometheus-node-exporter-setup.md)

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
