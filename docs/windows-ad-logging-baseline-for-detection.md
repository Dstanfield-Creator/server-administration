# Windows AD Logging Baseline for Detection Engineering

> Advanced Audit Policy, PowerShell logging, Sysmon with a community config, and event forwarding for an Active Directory lab, with a map from common AD attacks to the events that reveal them.

**Status:** Active · **Updated:** 2026-10-08

## Scope

A lab domain (`lab.example.com`) with one or two domain controllers, a few member servers and workstations, and a SIEM receiving events. Default Windows auditing records very little that detection content expects. Enabling this baseline raises log volume noticeably, mainly from 4662, 4688, and Sysmon 3, 7, and 10; size the Security log accordingly (last section).

## Advanced Audit Policy

Force subcategory settings to override the legacy categories so an old setting cannot silently win:

GPO: `Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > Security Options > Audit: Force audit policy subcategory settings (Windows Vista or later) to override audit policy category settings = Enabled`

Subcategories live under `Computer Configuration > Policies > Windows Settings > Security Settings > Advanced Audit Policy Configuration > Audit Policies`. Apply on DCs via a GPO linked to the Domain Controllers OU and on members via a baseline GPO:

| Category > Subcategory | Setting | Key Event IDs |
|---|---|---|
| Logon/Logoff > Logon | Success, Failure | 4624, 4625, 4648 (explicit credentials) |
| Account Management > User Account Management | Success, Failure | 4720, 4722, 4723, 4724, 4725, 4726, 4738, 4740 |
| Account Management > Security Group Management | Success, Failure | 4727-4737, 4756 (member added to universal group) |
| Account Logon > Kerberos Authentication Service | Success, Failure | 4768 (TGT issued), 4771 (pre-auth failed) |
| Account Logon > Kerberos Service Ticket Operations | Success, Failure | 4769 (TGS issued) |
| Account Logon > Credential Validation | Success, Failure | 4776 (NTLM) |
| DS Access > Directory Service Access | Success, Failure | 4662 (needs SACLs, below) |
| Detailed Tracking > Process Creation | Success | 4688 (with command line, below) |
| Object Access > Other Object Access Events | Success, Failure | 4698, 4699, 4702 (scheduled tasks) |
| System > Security System Extension | Success | 4697 (service installed); 7045 is its System-log twin, written by the SCM regardless of policy |

Equivalent `auditpol` commands, run elevated on a standalone lab box or to confirm a GPO applied:

```bat
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable
auditpol /set /subcategory:"Security Group Management" /success:enable /failure:enable
auditpol /set /subcategory:"Kerberos Authentication Service" /success:enable /failure:enable
auditpol /set /subcategory:"Kerberos Service Ticket Operations" /success:enable /failure:enable
auditpol /set /subcategory:"Credential Validation" /success:enable /failure:enable
auditpol /set /subcategory:"Directory Service Access" /success:enable /failure:enable
auditpol /set /subcategory:"Process Creation" /success:enable
auditpol /set /subcategory:"Other Object Access Events" /success:enable /failure:enable
auditpol /set /subcategory:"Security System Extension" /success:enable
```

### Command line in 4688

4688 without the command line is nearly useless. GPO: `Computer Configuration > Policies > Administrative Templates > System > Audit Process Creation > Include command line in process creation events = Enabled`, or:

```powershell
Set-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit' -Name ProcessCreationIncludeCmdLine_Enabled -Value 1 -Type DWord
```

### PowerShell script block logging (4104)

GPO: `Computer Configuration > Policies > Administrative Templates > Windows Components > Windows PowerShell > Turn on PowerShell Script Block Logging = Enabled`. Leave "Log script block invocation start / stop events" off; it doubles volume for little value. Also enable `Turn on Module Logging` with module name `*` for 4103. Both land in `Microsoft-Windows-PowerShell/Operational`.

```powershell
$k = 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging'
New-Item -Force $k | Out-Null
Set-ItemProperty $k -Name EnableScriptBlockLogging -Value 1 -Type DWord
```

### SACL for 4662 (DCSync visibility)

4662 fires only for objects carrying a matching SACL. Audit the three replication extended rights on the domain head so DCSync attempts are logged:

```powershell
$dn = (Get-ADDomain).DistinguishedName
$acl = Get-Acl "AD:\$dn"
$everyone = [System.Security.Principal.SecurityIdentifier]'S-1-1-0'
# DS-Replication-Get-Changes, -Get-Changes-All, -Get-Changes-In-Filtered-Set
foreach ($g in '1131f6aa-9c07-11d1-f79f-00c04fc2dcd2','1131f6ad-9c07-11d1-f79f-00c04fc2dcd2','89e95b76-444d-4c62-991a-0facbeda640c') {
  $acl.AddAuditRule((New-Object System.DirectoryServices.ActiveDirectoryAuditRule($everyone, 'ExtendedRight', 'Success', [guid]$g)))
}
Set-Acl "AD:\$dn" $acl
```

## Sysmon

Sysmon adds process, network, and registry telemetry the Security log lacks. Install with a maintained community configuration and review it before deploying: SwiftOnSecurity `sysmon-config` (single file, conservative defaults) or Olaf Hartong `sysmon-modular` (ATT&CK-tagged modules, build a config with `Merge-AllSysmonXml`).

```powershell
Invoke-WebRequest https://download.sysinternals.com/files/Sysmon.zip -OutFile $env:TEMP\Sysmon.zip
Expand-Archive $env:TEMP\Sysmon.zip $env:TEMP\Sysmon -Force
Invoke-WebRequest https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml -OutFile C:\baseline\sysmonconfig.xml
& $env:TEMP\Sysmon\Sysmon64.exe -accepteula -i C:\baseline\sysmonconfig.xml
```

Key event IDs in `Microsoft-Windows-Sysmon/Operational`:

| ID | Event | Why it matters |
|---|---|---|
| 1 | Process creation | Full command line, hashes, parent process; richer than 4688 |
| 3 | Network connection | Process-to-destination mapping; beacons and lateral movement |
| 7 | Image loaded | DLL side-loading, unsigned modules loaded into LSASS |
| 10 | Process access | Handles to `lsass.exe` with `0x1010` or `0x1fffff` access (credential dumping) |
| 11 | File create | Dropped executables and scripts in user-writable paths |
| 13 | Registry value set | Run keys, service `ImagePath`, WDigest `UseLogonCredential` |
| 22 | DNS query | Process-attributed DNS; DGA and tunnelling |

## Forwarding

**Windows Event Forwarding (agentless).** A collector runs `wecutil qc`; a GPO points sources at it (`Administrative Templates > Windows Components > Event Forwarding > Configure target Subscription Manager = Server=http://wec.lab.example.com:5985/wsman/SubscriptionManager/WEC,Refresh=60`); `NETWORK SERVICE` is added to the Event Log Readers group on sources. Load a subscription with `wecutil cs subscription.xml` and check active sources with `wecutil gr <name>`. Forwarded events land in `ForwardedEvents` on the collector, which one agent then ships. The Palantir `windows-event-forwarding` repository has ready-made subscription XML.

**Agent per host.** Winlogbeat (Elastic) or Fluent Bit (`winevtlog` input; outputs for Loki, OpenSearch, Splunk HEC) reads channels directly. Minimal `winlogbeat.yml`:

```yaml
winlogbeat.event_logs:
  - name: Security
    event_id: 4624, 4625, 4648, 4662, 4672, 4688, 4697, 4698, 4720-4740, 4756, 4768, 4769, 4771, 4776
  - name: System
    event_id: 7045
  - name: Microsoft-Windows-PowerShell/Operational
    event_id: 4103, 4104
  - name: Microsoft-Windows-Sysmon/Operational
output.elasticsearch:
  hosts: ["https://siem.lab.example.com:9200"]
```

## Attack to Event Map

| Technique | Primary events | What to look for |
|---|---|---|
| Kerberoasting | 4769 | Encryption type `0x17` (RC4) for service accounts; one account requesting many TGS in seconds |
| AS-REP roasting | 4768 | Pre-auth type `0` for accounts flagged "Do not require Kerberos preauthentication"; 4738 setting that flag |
| DCSync | 4662 | Access mask `0x100` with the three replication GUIDs above, from an account that is not a DC computer account |
| Pass-the-hash | 4624, 4776, 4648 | 4624 Logon Type 3 or 9, `Logon Process NtLmSsp`/`seclogo`, package NTLM, key length 0; 4648 on the source host |
| Password spraying | 4625, 4771, 4776 | Many accounts, one source; status `0xC000006A`; 4771 failure code `0x18` |
| Persistence via task or service | 4698, 7045, 4697, Sysmon 13 | Command lines pointing at user-writable paths or encoded PowerShell |
| Credential dumping | Sysmon 10, 7, 1 | Non-system process opening LSASS; unsigned DLL in LSASS; `procdump`, `comsvcs.dll MiniDump` |

## Log Sizes and Verification

```powershell
wevtutil sl Security /ms:1073741824                                 # 1 GB
auditpol /get /category:*                                           # every subcategory shows the expected setting
Get-WinEvent -LogName Security -FilterXPath "*[System[EventID=4688]]" -MaxEvents 3 | Format-List
Get-WinEvent -LogName 'Microsoft-Windows-Sysmon/Operational' -MaxEvents 3
```

Exercise the pipeline end to end: request a service ticket for a lab service account with RC4 and confirm a 4769 with `0x17` reaches the SIEM within the expected delay. A baseline that has not been tested against a known signal is a hypothesis.

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
