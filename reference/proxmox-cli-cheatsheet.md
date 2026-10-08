# Proxmox VE CLI Cheatsheet

> Quick reference for day-to-day Proxmox VE 8.x administration from the shell: qm, pct, pvesm, pveum, vzdump, pvesh, task logs, cluster status, and storage housekeeping.

**Status:** Active · **Updated:** 2026-10-08

Placeholders: `<node>` is the node name as shown by `pvecm nodes`, `<vmid>` is the numeric guest ID, `<newid>` is an unused ID, `<storage>` is a storage ID from `pvesm status`. All commands run as root on a PVE node. Most accept `--output-format json` for scripting.

## qm: QEMU Virtual Machines

```bash
qm list                                    # VMs on this node: status, memory, PID
qm config <vmid>                           # /etc/pve/qemu-server/<vmid>.conf, resolved
qm status <vmid> --verbose                 # state plus live stats
qm start <vmid>
qm shutdown <vmid> --timeout 120           # ACPI shutdown, wait up to 120 s
qm shutdown <vmid> --forceStop 1           # ACPI, then hard stop when the timeout expires
qm stop <vmid>                             # pull the plug
qm unlock <vmid>                           # clear a stale lock after a failed task
```

Settings and hardware:

```bash
qm set <vmid> --protection 1               # blocks destroy and disk removal until set to 0
qm set <vmid> --agent enabled=1,fstrim_cloned_disks=1
qm resize <vmid> scsi0 +20G                # grow only; shrinking is unsupported
```

Guest agent (needs `qemu-guest-agent` running inside the VM):

```bash
qm guest exec <vmid> -- /bin/sh -c 'ufw disable'           # out-of-band: no network path to the guest needed
qm guest exec <vmid> --timeout 60 -- systemctl status ssh
```

Snapshots, clones, templates:

```bash
qm snapshot <vmid> pre-upgrade --description 'before kernel update' --vmstate 1
qm rollback <vmid> pre-upgrade ; qm delsnapshot <vmid> pre-upgrade
qm clone <vmid> <newid> --name web02 --full 1 --storage <storage>    # omit --full for a linked clone of a template
qm template <vmid>                         # convert to template (irreversible)
qm destroy <vmid> --purge 1                # also removes backup-job and HA references; refused if protection=1
```

## pct: LXC Containers

```bash
pct list ; pct config <vmid>
pct start <vmid> ; pct shutdown <vmid> ; pct stop <vmid>
pct enter <vmid>                           # root shell inside the container
pct exec <vmid> -- apt update
pct create <vmid> local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
  --hostname ct01 --memory 512 --net0 name=eth0,bridge=vmbr0,ip=dhcp --unprivileged 1
```

## pvesm: Storage

```bash
pvesm status                               # every storage: type, active, total/used/avail
pvesm list <storage>                       # volumes on a storage
pvesm alloc <storage> <vmid> vm-<vmid>-disk-1 32G     # allocate a raw/qcow2/zvol volume
pvesm free <storage>:vm-<vmid>-disk-1      # delete a volume
```

## pveum: Users, Roles, Tokens, ACLs

```bash
pveum user list ; pveum role list ; pveum acl list
pveum user add svc-backup@pve --comment 'backup automation'
pveum role add BackupOperator --privs 'VM.Backup VM.Audit Datastore.AllocateSpace Datastore.Audit'
pveum acl modify /vms --users svc-backup@pve --roles BackupOperator
pveum user token add svc-backup@pve ci --privsep 1 --expire 0    # secret is printed once
pveum acl modify /vms --tokens 'svc-backup@pve!ci' --roles BackupOperator
pveum user token list svc-backup@pve ; pveum user token remove svc-backup@pve ci
pveum user permissions svc-backup@pve      # effective permissions by path
```

## vzdump: One-off Backups and Restores

```bash
vzdump <vmid> --storage <storage> --mode snapshot --compress zstd --notes-template '{{guestname}} manual'
vzdump <vmid> --storage pbs-main --mode snapshot              # to a Proxmox Backup Server datastore
qmrestore /var/lib/vz/dump/vzdump-qemu-<vmid>-*.vma.zst <newid> --storage <storage>
```

Modes: `snapshot` (live; needs snapshot-capable storage, guest agent fsfreeze recommended), `suspend` (brief pause), `stop` (shut down, back up, start).

## pvesh: The API From the Shell

`pvesh` calls the same REST API the web UI uses; `pvesh usage <path> --verbose` prints an endpoint's schema.

```bash
pvesh get /nodes/<node>/qemu                                     # all VMs on a node with status
pvesh get /nodes/<node>/qemu/<vmid>/config
pvesh get /cluster/resources --type vm --output-format json | jq -r '.[] | [.vmid,.name,.status,.node] | @tsv'
pvesh create /nodes/<node>/qemu/<vmid>/status/shutdown
pvesh set /nodes/<node>/qemu/<vmid>/config --protection 1
```

## Tasks and Logs

```bash
pvesh get /nodes/<node>/tasks --limit 20 --source all
pvesh get /nodes/<node>/tasks/<UPID>/log                         # UPID from the task list
tail -f /var/log/pve/tasks/active
```

## Cluster and Node Status

```bash
pveversion -v                              # every PVE package version; attach to bug reports
pvecm status                               # quorum, votes, member addresses
systemctl status pve-cluster corosync pvedaemon pveproxy pvestatd --no-pager
```

## Storage Housekeeping: What Fills `local`

`local` is `/var/lib/vz` on the root filesystem. When it fills, the node itself starts failing.

```bash
df -h / /var/lib/vz
du -sh /var/lib/vz/*                       # dump/, images/, template/, private/
du -sh /var/lib/vz/dump/* | sort -h | tail
pvesh get /nodes/<node>/storage/local/content --content backup --output-format json | jq -r '.[] | [.ctime,.size,.volid] | @tsv' | sort
pvesm free local:backup/vzdump-qemu-<vmid>-2026_01_01-00_00_00.vma.zst
journalctl --vacuum-size=500M ; apt clean
```

Set `prune-backups` on the storage in `/etc/pve/storage.cfg` (for example `keep-last=3,keep-weekly=4`) so old dumps are pruned automatically rather than by hand.

## Common Status Fields

| Field | Seen in | Meaning |
|---|---|---|
| `status` | `qm list`, `pct list`, `/cluster/resources` | `running`, `stopped`, `paused`, `suspended` |
| `lock` | `qm config`, `qm status` | `backup`, `snapshot`, `migrate`, `clone`, `rollback`; clear with `qm unlock` only once the task is dead |
| `pid` | `qm list` | Host PID of the QEMU process; 0 when stopped |
| `maxmem` / `mem` | `/cluster/resources` | Configured vs. currently used memory in bytes |
| `maxdisk` / `disk` | `/cluster/resources` | Disk size vs. used; used is 0 for VMs without the guest agent |
| `quorate` | `pvecm status` | `Yes` means the cluster can make changes; `No` means `/etc/pve` is read-only |
| `active` / `enabled` | `pvesm status` | Storage is reachable / configured for use |

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
