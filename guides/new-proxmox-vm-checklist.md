# New Proxmox VM Checklist

> Tickable checklist for taking a new Linux VM on Proxmox VE from "created" to "in service": VM settings, network, access, firewall, updates, monitoring, backup and documentation.

**Status:** Active · **Updated:** 2026-10-08

## How to use

Copy this file into the change record for the VM, fill in the header, and tick each item only after verifying it with a live command. Items marked **(critical only)** apply to VMs whose loss would be an incident.

| Field | Value |
|---|---|
| VM ID | |
| Hostname | `vm-name.example.com` |
| Purpose | |
| Owner | |
| Node | |
| Date in service | |

## 1. VM settings (Proxmox host)

- [ ] `qemu-guest-agent` installed and enabled in the guest (`sudo apt install -y qemu-guest-agent && sudo systemctl enable --now qemu-guest-agent`); agent option enabled on the VM
- [ ] Disk controller is VirtIO SCSI (single) with `iothread=1`; not IDE or SATA
- [ ] NIC is VirtIO, not E1000
- [ ] `discard=on` on every disk so TRIM reaches thin storage; `sudo fstrim -v /` succeeds in the guest
- [ ] Start order and delay set relative to dependencies (`--startup order=N,up=30`)
- [ ] **(critical only)** `protection: 1` set so the VM cannot be destroyed by accident
- [ ] Pool assigned if automation ACLs are pool-scoped
- [ ] Description field filled in: purpose, owner, link to this record

```bash
qm set <vmid> --agent enabled=1,fstrim_cloned_disks=1
qm set <vmid> --scsihw virtio-scsi-single
qm set <vmid> --scsi0 local-lvm:vm-<vmid>-disk-0,discard=on,iothread=1,ssd=1
qm set <vmid> --net0 virtio,bridge=vmbr0
qm set <vmid> --startup order=20,up=30
qm set <vmid> --protection 1          # critical only
qm agent <vmid> ping                  # succeeds once the agent is running
```

## 2. Network

- [ ] Static DHCP reservation created for the NIC's MAC address
- [ ] Hostname set and matches the DNS record
- [ ] Forward and reverse DNS resolve from another host
- [ ] Time sync active

```bash
ip -br link
ip -br addr
getent hosts vm-name.example.com
dig -x 192.0.2.50 +short
timedatectl | grep synchronized
```

## 3. Access

- [ ] Dedicated admin user with sudo; no shared accounts
- [ ] Public key installed (`~/.ssh` 700, `authorized_keys` 600)
- [ ] `PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `PermitRootLogin no`
- [ ] Default cloud-image user disabled or removed
- [ ] Tailscale joined and tagged if remote access is needed (`sudo tailscale up --advertise-tags=tag:server`); MagicDNS name recorded
- [ ] Key-only login verified from a fresh terminal before the console session is closed

```bash
sudo adduser --disabled-password --gecos '' admin && sudo usermod -aG sudo admin
sudo install -d -m 700 -o admin -g admin /home/admin/.ssh
echo 'ssh-ed25519 AAAA...example admin@workstation' \
  | sudo install -m 600 -o admin -g admin /dev/stdin /home/admin/.ssh/authorized_keys

sudo tee /etc/ssh/sshd_config.d/10-hardening.conf >/dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
EOF
sudo sshd -t && sudo systemctl reload ssh
```

## 4. Firewall

- [ ] Rules staged: deny incoming, SSH rate-limited, service ports from the correct source ranges only
- [ ] Enabled with the dead-man procedure in [remote-firewall-change.md](https://github.com/Dstanfield-Creator/network/blob/main/firewall/remote-firewall-change.md), tested from a brand new SSH session
- [ ] Dead-man timer disarmed after the test (`systemctl list-timers fw-deadman.timer` is empty)

```bash
sudo ufw default deny incoming
sudo ufw limit 22/tcp comment 'ssh'
sudo ufw allow from 192.0.2.0/24 to any port 9100 proto tcp comment 'node_exporter'
sudo systemd-run --unit=fw-deadman --on-active=180 /usr/sbin/ufw disable
# enable (out-of-band if possible), test from a NEW session, then:
sudo systemctl stop fw-deadman.timer
sudo ufw status verbose
```

## 5. Updates

- [ ] Fully patched at handover; rebooted if the kernel changed
- [ ] `unattended-upgrades` installed and enabled for security updates
- [ ] Automatic reboot policy decided (`Unattended-Upgrade::Automatic-Reboot`, `Automatic-Reboot-Time`)
- [ ] `/var/run/reboot-required` covered by monitoring

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

## 6. Monitoring

- [ ] `node_exporter` installed, listening on 9100, reachable only from the monitoring host
- [ ] Scrape target added and showing `up == 1`
- [ ] Host added to the SSH reachability check inventory
- [ ] Disk, memory and reboot-required alerts cover the host

```bash
systemctl status node_exporter --no-pager
ss -tlnp | grep 9100
curl -s http://192.0.2.50:9100/metrics | head -3
```

## 7. Backup

- [ ] VM added to the vzdump / Proxmox Backup Server job with the right schedule and retention
- [ ] First backup completed and verified on the backup server
- [ ] Restore tested: restore to a spare VM ID, boot with the NIC link down, check the data, destroy the test VM
- [ ] Backup failure notifications reach a monitored address or channel

```bash
vzdump <vmid> --storage pbs-store --mode snapshot
proxmox-backup-client snapshot list --repository backup@pbs@pbs.example.com:store1
qmrestore pbs-store:backup/vm/<vmid>/<timestamp> 9999 --unique 1
qm set 9999 --net0 virtio,bridge=vmbr0,link_down=1
qm start 9999            # verify data, then:
qm stop 9999 && qm destroy 9999 --purge
```

## 8. Documentation

- [ ] Inventory row added: VM ID, hostname, IP, node, purpose, owner, backup job, monitoring
- [ ] Purpose and services written in the VM description and the inventory
- [ ] Owner and on-call contact recorded
- [ ] Decommission plan written: dependants, how to drain, what to delete, backup retention after removal
- [ ] This checklist attached to the change record with the completion date

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
