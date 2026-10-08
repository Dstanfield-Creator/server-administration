# Proxmox VE API Tokens with Least Privilege

> Create a scoped API token for automation on Proxmox VE: a custom role with only the privileges the job needs, a dedicated user, an ACL on the narrowest path, and a secret that never lives in a script.

**Status:** Active · **Updated:** 2026-10-08

## How Proxmox permissions fit together

Proxmox VE grants access through ACLs. An ACL binds a role (a set of privileges) to a user or token on a path (`/`, `/vms`, `/vms/211`, `/storage/local-lvm`, `/nodes/pve1`, `/pool/agents`). The least-privilege pattern for automation is:

1. A custom role holding only the privileges the job needs.
2. A dedicated user in the `pve` realm with no password and no login.
3. An API token for that user.
4. An ACL binding the role to the user on the narrowest path that works.

All commands run as root on a cluster node. The `pve` realm is used throughout; `pam` behaves the same.

## Privilege reference

| Need | Privileges |
|---|---|
| Read-only inventory (`qm list`, node status) | `Sys.Audit`, `VM.Audit`, `Datastore.Audit` |
| Start, stop, reboot VMs | `VM.PowerMgmt` plus `VM.Audit` |
| Change a VM's hardware or options | `VM.Config.*` |
| Create, clone, destroy VMs | `VM.Allocate`, `VM.Clone`, `Datastore.AllocateSpace` |
| Guest agent commands (`qm guest exec`) | `VM.Monitor` on older releases; the `VM.GuestAgent.*` family on current ones |
| Back up | `VM.Backup`, `Datastore.AllocateSpace` |

`pveum role list` prints every built-in role with its privileges, which is the quickest way to see the exact names on your version.

## Example 1: power-management-only role

For a scheduler or dashboard that lists VMs and starts or stops them, but cannot change configuration or touch disks.

```bash
pveum role add PowerOps --privs "Sys.Audit Sys.PowerMgmt VM.Audit VM.PowerMgmt"
pveum user add svc-power@pve --comment "Power scheduling automation"
pveum user token add svc-power@pve sched --privsep 0 --comment "cron on ops-host.example.com"
pveum acl modify /vms   --users svc-power@pve --roles PowerOps
pveum acl modify /nodes --users svc-power@pve --roles PowerOps
```

`token add` prints the secret exactly once. Put it in the vault at that moment (see Gotchas).

## Example 2: VM-admin role for an agent

For an orchestration or build agent that creates, modifies, snapshots and destroys VMs inside its own pool.

```bash
pveum role add VMAgent --privs "VM.Allocate VM.Audit VM.Clone VM.Config.CDROM VM.Config.CPU VM.Config.Cloudinit VM.Config.Disk VM.Config.HWType VM.Config.Memory VM.Config.Network VM.Config.Options VM.Console VM.Monitor VM.PowerMgmt VM.Snapshot VM.Snapshot.Rollback Datastore.Audit Datastore.AllocateSpace Pool.Audit SDN.Use"
pveum pool add agents --comment "VMs owned by automation"
pveum user add svc-agent@pve
pveum user token add svc-agent@pve ci --privsep 0
pveum acl modify /pool/agents                     --users svc-agent@pve --roles VMAgent
pveum acl modify /storage/local-lvm               --users svc-agent@pve --roles VMAgent
pveum acl modify /sdn/zones/localnetwork/vmbr0    --users svc-agent@pve --roles VMAgent
```

Granting on `/pool/agents` means the agent sees only VMs placed in that pool. Create new VMs with `--pool agents` so they inherit the grant.

## Path-specific overrides

A grant on a more specific path overrides a broader one. Use this to carve crown-jewel VMs out of a wide grant.

```bash
# Broad: the agent may manage everything under /vms
pveum acl modify /vms     --users svc-agent@pve --roles VMAgent

# Specific: on VM 211 the same user is read-only
pveum acl modify /vms/211 --users svc-agent@pve --roles PVEAuditor
```

Check the effective privileges at the specific path:

```bash
pveum user permissions svc-agent@pve --path /vms/211
pveum user token permissions svc-agent@pve ci --path /vms/211
```

Add the protection flag to VMs that must never be destroyed by automation regardless of role:

```bash
qm set 211 --protection 1
qm config 211 | grep protection
```

With `protection: 1`, `qm destroy` and disk removal fail until the flag is cleared, which is a deliberate second step requiring `VM.Config.Options`.

## Gotchas

**Token privileges are capped by the owning user.** A token can never do more than its user. With `--privsep 1` (the default) the token needs its own ACL entries and gets the intersection of those and the user's. With `--privsep 0` it inherits the user's permissions exactly. In both cases the user must hold the ACL. This looks correct but yields nothing if `svc-power@pve` itself has no grants:

```bash
pveum acl modify /vms --tokens 'svc-power@pve!sched' --roles PowerOps
```

**Do not reuse `root@pam` tokens.** A leaked root token is a leaked cluster.

**Rotate deliberately.** `pveum user token remove svc-power@pve sched`, then `token add` again, update the vault item, redeploy.

**Never write the secret into a script or commit it.** Read it at runtime from one of these:

```bash
# 1Password CLI
PVE_TOKEN="$(op read 'op://Infra/pve-svc-power/credential')"

# Environment variable supplied by the caller or a systemd EnvironmentFile (mode 600)
PVE_TOKEN="${PVE_TOKEN:?PVE_TOKEN not set}"

# Mode-600 file owned by the service user
PVE_TOKEN="$(cat /etc/pve-automation/token)"
```

## Using the token

```bash
PVE_HOST=pve1.example.com
PVE_TOKENID='svc-power@pve!sched'

curl -sS --fail \
  -H "Authorization: PVEAPIToken=${PVE_TOKENID}=${PVE_TOKEN}" \
  "https://${PVE_HOST}:8006/api2/json/cluster/resources?type=vm" \
  | jq '.data[] | {vmid, name, status}'
```

## Verification

```bash
pveum user token list svc-power@pve
pveum user token permissions svc-power@pve sched
pveum user token permissions svc-power@pve sched --path /vms/211
pveum acl list | grep -E 'svc-(power|agent)'
```

For `svc-power@pve!sched` the output should list only `Sys.Audit`, `Sys.PowerMgmt`, `VM.Audit` and `VM.PowerMgmt`. If `VM.Allocate` or any `VM.Config.*` appears, a broader grant is leaking in; look for ACLs on `/` or group memberships.

Finish with a negative test from the client and expect a 403:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' -X DELETE \
  -H "Authorization: PVEAPIToken=${PVE_TOKENID}=${PVE_TOKEN}" \
  "https://${PVE_HOST}:8006/api2/json/nodes/pve1/qemu/211"
```

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
