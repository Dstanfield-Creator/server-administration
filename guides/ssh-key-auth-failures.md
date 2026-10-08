# SSH Key Authentication Failures

> Classify an SSH failure into one of six buckets (DNS, DOWN, AUTH, HOSTKEY, HOSTKEY!, AGENT) and run the command that confirms each bucket before changing anything.

**Status:** Active · **Updated:** 2026-10-08

## Classify before fixing

Fixing the wrong layer wastes time (regenerating keys when sshd is stopped) or opens a hole (deleting a `known_hosts` line when the host key really did change). Work down this table; each check rules out the layer above it.

| Class | Symptom | Layer | First command |
|---|---|---|---|
| DNS | `Could not resolve hostname` | Name resolution | `ssh -G`, `getent hosts` |
| DOWN | `Connection refused`, `Connection timed out`, `No route to host` | TCP | `/dev/tcp` probe |
| AUTH | Banner received, then `Permission denied (publickey)` | Authentication | `ssh -vvv`, `ssh-add -L` |
| HOSTKEY | `The authenticity of host ... can't be established` | Host key, first contact | Verify fingerprint out-of-band |
| HOSTKEY! | `REMOTE HOST IDENTIFICATION HAS CHANGED!` | Host key, mismatch | Investigate; do not auto-fix |
| AGENT | AUTH symptoms, but the key exists on disk | Agent socket | `echo $SSH_AUTH_SOCK`, `ssh-add -L` |

## Step 0: see what ssh will actually do

`ssh -G` prints the effective configuration after every `Host` and `Match` block, without connecting. If `hostname`, `port` or `user` is not what you expected, an alias in `~/.ssh/config` is redirecting you; fix that first.

```bash
ssh -G host.example.com | grep -E '^(hostname|port|user|identityfile|identityagent|identitiesonly|proxyjump|proxycommand) '
```

## DNS

```bash
getent hosts host.example.com
resolvectl status | grep -A3 'DNS Servers'
tailscale status | head        # MagicDNS names (host.example.ts.net) need Tailscale up on the client
```

## DOWN

Test TCP reachability without involving SSH. This separates "nothing is listening" from "SSH rejected me".

```bash
timeout 4 bash -c 'exec 3<>/dev/tcp/192.0.2.10/22' && echo open || echo closed-or-filtered
```

| Result | Meaning |
|---|---|
| `open` at once | TCP is fine; go to AUTH |
| `Connection refused` at once | Host is up, nothing on that port (sshd stopped, wrong port) |
| Times out after 4 s | Host down, firewall dropping, or wrong address |
| `No route to host` | Local routing or ARP; check `ip route get 192.0.2.10` |

A firewall enabled minutes earlier is the usual cause of a sudden timeout on a host that was fine; see the remote firewall change runbook. For a VM you control, check from the hypervisor with `qm status <vmid>` and `qm guest exec <vmid> -- systemctl status ssh`.

## AUTH

The server answered the handshake and rejected every key.

```bash
ssh -vvv -o BatchMode=yes admin@host.example.com true 2>&1 \
  | grep -E 'Offering|Server accepts|Authentications that can continue|Permission denied|no mutual'
```

`BatchMode=yes` stops a password prompt from hiding the real failure and makes the command safe in scripts.

| Line in `-vvv` output | Meaning |
|---|---|
| `Offering public key:` | Key sent to the server |
| `Server accepts key:` | Server matched it; a failure after this is server-side |
| No `Offering` lines at all | Client has no usable key; see AGENT |
| `no mutual signature algorithm` | Legacy RSA-SHA1 mismatch; upgrade or set `PubkeyAcceptedAlgorithms` |

Compare fingerprints: `ssh-add -L | ssh-keygen -lf -` on the client, `ssh-keygen -lf ~/.ssh/authorized_keys` on the server.

Server-side checks, via console or another working account:

```bash
ls -ld ~ ~/.ssh ~/.ssh/authorized_keys     # ~/.ssh 700, authorized_keys 600, home not group-writable
chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys
sudo journalctl -u ssh  --since '-15 min' | tail -40    # Debian/Ubuntu
sudo journalctl -u sshd --since '-15 min' | tail -40    # RHEL/Fedora
sudo sshd -T | grep -E '^(pubkeyauthentication|passwordauthentication|authorizedkeysfile|allowusers|allowgroups)'
```

## HOSTKEY (first contact)

```text
The authenticity of host 'host.example.com (192.0.2.10)' can't be established.
ED25519 key fingerprint is SHA256:...
```

Verify the fingerprint out-of-band before accepting. On the console, `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`; from a hypervisor, `qm guest exec <vmid> -- cat /etc/ssh/ssh_host_ed25519_key.pub`. Then pre-seed `known_hosts`:

```bash
ssh-keyscan -t ed25519 host.example.com >> ~/.ssh/known_hosts
```

## HOSTKEY! (key changed)

```text
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
```

This is not a nuisance to click past. Either the host was legitimately rebuilt, or something between you and it is answering in its place. Treat it as an incident until you know which.

1. Was this host rebuilt or reinstalled? Check the change record and the decommission log.
2. Get the current fingerprint out-of-band (console, hypervisor, cloud serial console) and compare it to the one in the warning.
3. Only when the fingerprint matches an out-of-band source, remove the stale entries:

```bash
ssh-keygen -R host.example.com
ssh-keygen -R 192.0.2.10
```

Never set `StrictHostKeyChecking=no` to silence the warning.

## AGENT

An agent that is not being picked up looks exactly like a missing key: `Permission denied (publickey)` with no `Offering public key` lines.

```bash
echo "$SSH_AUTH_SOCK"
ssh-add -L
ssh -G host.example.com | grep identityagent
```

| Output | Fix |
|---|---|
| `SSH_AUTH_SOCK` empty | No agent in this shell; start one or re-source the session environment |
| Socket set but `Could not open a connection to your authentication agent` | Agent process died; restart it or log in again |
| `The agent has no identities` | Agent is up but empty; `ssh-add ~/.ssh/id_ed25519` |
| Config sets `IdentityAgent` to a 1Password socket | 1Password must be unlocked with its SSH agent enabled; the socket is `~/.1password/agent.sock` on Linux |
| Works in a terminal but not from cron or a systemd unit | Those contexts do not inherit `SSH_AUTH_SOCK`; use a dedicated key file with `IdentitiesOnly yes` |

A typical 1Password agent stanza; `ssh -G` must show this path as `identityagent` and the socket must exist:

```text
Host *.example.com
    IdentityAgent ~/.1password/agent.sock
    IdentitiesOnly yes
```

## Many hosts failing together

When several hosts fail at once the cause is usually shared: DNS, a jump host, Tailscale, or the agent. A parallel checker that probes every host in an inventory and reports each as DNS, DOWN, AUTH, HOSTKEY or OK is at
https://github.com/Dstanfield-Creator/lab-ops/tree/main/scripts/lab-ssh-check

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
