# OpenSSH Server Hardening

> Key-only, no-root sshd configuration delivered as a drop-in file, with modern crypto, host key cleanup, a safe reload procedure, locked-down automation keys, and external verification with ssh-audit.

**Status:** Active · **Updated:** 2026-10-08

## Scope

OpenSSH 8.x/9.x on Debian 12 and Ubuntu 22.04/24.04. Both ship `Include /etc/ssh/sshd_config.d/*.conf` at the top of `sshd_config`. sshd uses the first value it sees for most keywords, so a drop-in wins over the stock defaults below the Include line. Leave `/etc/ssh/sshd_config` untouched; package upgrades then apply cleanly.

## Prerequisites

Confirm a working key login for the admin account from the management host before disabling passwords, and create a group that is permitted to log in:

```bash
ssh-keygen -t ed25519 -a 64 -C "admin@example.com"
ssh-copy-id -i ~/.ssh/id_ed25519.pub admin@host.example.com
ssh -o PasswordAuthentication=no -o KbdInteractiveAuthentication=no admin@host.example.com true && echo OK
sudo groupadd --system ssh-users && sudo usermod -aG ssh-users admin    # on the server
```

## Drop-in Configuration

`/etc/ssh/sshd_config.d/10-hardening.conf` (mode 0644, owned by root):

```text
# Authentication
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AuthenticationMethods publickey
PermitRootLogin no
MaxAuthTries 3
LoginGraceTime 30
AllowGroups ssh-users
# AllowUsers admin deploy@192.0.2.0/24     # alternative to AllowGroups; user@host patterns allowed

X11Forwarding no
AllowTcpForwarding no                      # yes if this host is a jump host

# Drop dead clients after 3 x 300 s of silence
ClientAliveInterval 300
ClientAliveCountMax 3

# Host keys: ed25519 first, RSA for older clients
HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key

# Crypto
KexAlgorithms sntrup761x25519-sha512@openssh.com,curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com

LogLevel VERBOSE                           # records the key fingerprint for each login
```

Notes:

- `sntrup761x25519-sha512@openssh.com` needs OpenSSH 8.5+ on both ends. Remove it on older servers; `sshd -t` reports an unknown algorithm.
- `KbdInteractiveAuthentication` replaced `ChallengeResponseAuthentication` in 8.7. On older builds set both.

## Host Keys

Remove DSA and ECDSA keys and regenerate the two that the config lists:

```bash
cd /etc/ssh
sudo rm -f ssh_host_dsa_key* ssh_host_ecdsa_key* ssh_host_rsa_key* ssh_host_ed25519_key*
sudo ssh-keygen -t ed25519 -f ssh_host_ed25519_key -N ""
sudo ssh-keygen -t rsa -b 4096 -f ssh_host_rsa_key -N ""
ssh-keygen -lf ssh_host_ed25519_key.pub        # publish this fingerprint out of band
```

Regenerating changes the host fingerprint; every client sees a host key warning and must update `known_hosts` (`ssh-keygen -R host.example.com`, then reconnect).

## Test, Then Reload Without Losing the Session

```bash
sudo sshd -t                                   # syntax check, silent on success
sudo sshd -T | grep -Ei '^(passwordauth|permitroot|allowgroups|kexalg|ciphers|macs)'
sudo systemctl reload ssh                      # 'sshd' on RHEL-family hosts
```

Reload applies the new config to new connections only; the current session stays up. Keep it open and test from a NEW terminal before closing it:

```bash
ssh admin@host.example.com true && echo "key login OK"
ssh -o PubkeyAuthentication=no admin@host.example.com    # expect: Permission denied (publickey)
ssh root@host.example.com                                 # expect: Permission denied (publickey)
```

## Restricting Automation Keys

Keys used by backup jobs, CI, or monitoring belong in `~/.ssh/authorized_keys` on the server with options in front of the key:

```text
from="192.0.2.10,192.0.2.11",command="/usr/local/bin/backup-shell",no-port-forwarding,no-agent-forwarding,no-X11-forwarding,no-pty ssh-ed25519 AAAAC3... backup@example.com
restrict,from="192.0.2.0/24",command="rrsync -ro /srv/data" ssh-ed25519 AAAAC3... mirror@example.com
```

- `from=` limits source addresses or hostnames; comma separated, `!` negates, CIDR accepted.
- `command=` forces that command regardless of what the client asked for. The original request is available to the script in `$SSH_ORIGINAL_COMMAND`, which lets one key serve a small allow-list of subcommands.
- `restrict` (OpenSSH 7.2+) disables all forwarding, pty allocation, and `~/.ssh/rc` in one word; add back only what is needed with `pty` or `port-forwarding`.

## Brute-Force Throttling

If UFW is in use, `sudo ufw limit 22/tcp` rate-limits the port. Otherwise, or in addition, fail2ban bans on log evidence rather than connection rate:

```bash
sudo apt install -y fail2ban
sudo tee /etc/fail2ban/jail.d/sshd.local >/dev/null <<'EOF'
[sshd]
enabled  = true
backend  = systemd
maxretry = 3
findtime = 10m
bantime  = 1h
ignoreip = 127.0.0.1/8 192.0.2.0/24
EOF
sudo systemctl restart fail2ban && sudo fail2ban-client status sshd
```

## Verification

Catch typos in the algorithm lists by asking the local build what it supports:

```bash
ssh -Q kex
ssh -Q cipher
```

Run `ssh-audit` from another host. It connects as a client and grades every algorithm the server offers:

```bash
pipx install ssh-audit                 # or: pip install --user ssh-audit
ssh-audit host.example.com
```

A clean result lists only the algorithms configured above with no `(fail)` or `(warn)` lines. Finish with `ss -tlnp | grep sshd` on the host and a review of the UFW rule covering port 22.

## Related

- [UFW baseline](https://github.com/Dstanfield-Creator/network/blob/main/firewall/ufw-baseline-linux.md)

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
