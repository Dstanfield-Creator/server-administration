# systemd Service Sandboxing

> Confining a long-running service with systemd's sandboxing directives, scoring it with `systemd-analyze security`, and the caveats for JIT runtimes such as Java and Node.

**Status:** Active · **Updated:** 2026-10-08

A service running as root with the whole filesystem writable has nothing between a bug and the host. systemd can confine a unit with mount namespaces, seccomp filters, and capability bounding without changes to the application; the kernel enforces each directive at start.

## Baseline Score

```bash
systemd-analyze security                        # all units, exposure 0.0 (safe) to 10.0 (unsafe)
systemd-analyze security gameserver.service     # one unit, every check listed
```

A unit with only `ExecStart=` and no `User=` scores 9.6 UNSAFE. The output lists each check, whether it passed, and the exposure it contributes. Work down that list.

## Directive Reference

| Directive | Effect | Watch out for |
|---|---|---|
| `User=` / `Group=` | Run as an unprivileged account | Data dirs must be owned by that account |
| `NoNewPrivileges=yes` | setuid/setgid/file caps cannot raise privileges | Anything that calls `sudo` or `ping` |
| `ProtectSystem=strict` | Whole filesystem read-only except `/dev`, `/proc`, `/sys` | Needs `ReadWritePaths=` or `StateDirectory=` |
| `ReadWritePaths=` | Carve out writable paths | Paths must exist before start |
| `ProtectHome=yes` | `/home`, `/root`, `/run/user` empty and inaccessible | Software installed under `/home` breaks; use `/opt` or `/srv` |
| `PrivateTmp=yes` | Private `/tmp` and `/var/tmp` | Unix sockets in `/tmp` invisible to other units |
| `PrivateDevices=yes` | Minimal `/dev` (null, zero, random, ...) | GPU or serial access needs `DeviceAllow=` instead |
| `ProtectKernelTunables=yes`, `ProtectKernelModules=yes`, `ProtectControlGroups=yes` | `/proc/sys` and `/sys` read-only, no module loading, `/sys/fs/cgroup` read-only | Apps that set sysctls at start; container runtimes |
| `RestrictAddressFamilies=` | Allow-list socket families | Omit `AF_UNIX` and journald logging breaks |
| `RestrictNamespaces=yes`, `RestrictRealtime=yes`, `RestrictSUIDSGID=yes`, `LockPersonality=yes` | Block namespace creation, RT scheduling, setuid file creation, personality changes | Low risk |
| `CapabilityBoundingSet=` / `AmbientCapabilities=` | Drop every capability not listed (empty = none) / grant one to a non-root user | Ports below 1024 need `CAP_NET_BIND_SERVICE` in both |
| `SystemCallFilter=@system-service` | Seccomp allow-list of ordinary service syscalls | Add `~@privileged @mount` to deny more |
| `MemoryDenyWriteExecute=yes` | No writable-and-executable memory mappings | Breaks JIT runtimes (next section) |

## MemoryDenyWriteExecute and JIT Runtimes

`MemoryDenyWriteExecute=yes` refuses `mmap` and `mprotect` calls that make a page both writable and executable. Runtimes that emit machine code at run time need exactly that:

- Java (HotSpot, OpenJ9) fails to start or falls back to interpreted mode with `-Xint`, which is many times slower.
- Node.js and anything on V8 fails unless run with `--jitless`, which disables WebAssembly and slows JavaScript considerably.
- .NET, LuaJIT, PyPy, and Go programs using `plugin` are affected similarly.

Leave the directive off for these runtimes and accept the one exposure line it costs. Keep `SystemCallFilter=@system-service` rather than a tighter set; JVMs call `setrlimit`, `mincore`, and `sched_*`, which are all inside `@system-service`.

## Graceful Stops and Restart Policy

```ini
KillSignal=SIGTERM           # default; use ExecStop= when the app has a clean shutdown command
KillMode=mixed               # SIGTERM to the main PID, SIGKILL to leftovers after the timeout
TimeoutStopSec=90            # time for a world save or connection drain before SIGKILL
Restart=on-failure           # non-zero exit, signal, timeout, or watchdog
RestartSec=10
# StartLimitIntervalSec=300 and StartLimitBurst=5 go in [Unit] and stop a crash loop
```

## Worked Example: Java Game Server

```bash
sudo useradd --system --home /var/lib/gameserver --shell /usr/sbin/nologin gameserver
sudo install -d -o gameserver -g gameserver -m 0750 /var/lib/gameserver
```

`/etc/systemd/system/gameserver.service`:

```ini
[Unit]
Description=Java game server
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=300
StartLimitBurst=5

[Service]
User=gameserver
Group=gameserver
ExecStart=/usr/bin/java -Xms2G -Xmx4G -jar /opt/gameserver/server.jar nogui
ExecStop=/bin/kill -s SIGINT $MAINPID
KillMode=mixed
TimeoutStopSec=90
Restart=on-failure
RestartSec=10

# Filesystem
ProtectSystem=strict
ReadWritePaths=/var/lib/gameserver
ProtectHome=yes
PrivateTmp=yes
PrivateDevices=yes

# Kernel, process, and network
NoNewPrivileges=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
RestrictNamespaces=yes
RestrictRealtime=yes
RestrictSUIDSGID=yes
LockPersonality=yes
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
CapabilityBoundingSet=
SystemCallArchitectures=native
SystemCallFilter=@system-service
SystemCallFilter=~@privileged @mount @cpu-emulation @obsolete
# MemoryDenyWriteExecute=yes   # intentionally off: the HotSpot JIT needs W+X pages

[Install]
WantedBy=multi-user.target
```

For a Node application, change `ExecStart` to `/usr/bin/node /opt/app/server.js`, keep `MemoryDenyWriteExecute` off, and if it must bind 80 or 443 replace the empty bounding set with `CapabilityBoundingSet=CAP_NET_BIND_SERVICE` plus `AmbientCapabilities=CAP_NET_BIND_SERVICE`.

## Apply and Score Again

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now gameserver.service
journalctl -u gameserver.service -f
systemd-analyze security gameserver.service
```

| Stage | Exposure | Rating |
|---|---|---|
| `ExecStart=` only, running as root | 9.6 | UNSAFE |
| Plus `User=`, `NoNewPrivileges`, `ProtectSystem=strict`, `PrivateTmp` | about 6.5 | MEDIUM |
| Full unit above, MDWE off | about 1.8 | OK |
| Full unit with `MemoryDenyWriteExecute=yes` (non-JIT binary) | about 1.3 | OK |

The exposure figure is a weighted sum of failed checks, not a count of vulnerabilities. A score near 2 with a documented reason for each remaining line (MDWE off for the JVM, `AF_INET` because it is a network service) is a good end state. Chasing 0.0 usually means breaking the application.

## Debugging a Confined Service

```bash
journalctl -u gameserver.service -b --no-pager | tail -50     # seccomp kills: "status=31/SYS"; EROFS = read-only fs
systemd-analyze syscall-filter @system-service                 # what a filter group contains
sudo systemd-run -p ProtectSystem=strict -p User=gameserver --wait --pty /opt/gameserver/run.sh   # test properties without editing the unit
```

Bisect by commenting out directives in blocks (filesystem, then kernel, then seccomp), running `daemon-reload` and `restart` each time.

## Related

- [OpenSSH server hardening](./openssh-server-hardening.md)

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
