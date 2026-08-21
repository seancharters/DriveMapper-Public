# DriveMapper — Administrator Guide

Maps network drives for signed-in Windows users based on their **Entra ID group
membership**, driven by a Windows service rather than a logon script or scheduled task.

Works across a mixed fleet — hybrid-joined and cloud-only Entra-joined devices alike.

This guide covers everything needed to deploy and run DriveMapper from the released
`DriveMapper.msi` and a `drivemap.json`. No source code required.

---

## Contents

1. [How it works](#how-it-works)
2. [Before you start](#before-you-start)
3. [Entra ID setup](#entra-id-setup)
4. [Configuration reference](#configuration-reference)
5. [Installing](#installing)
6. [Deploying with Intune](#deploying-with-intune)
7. [Updating drive mappings](#updating-drive-mappings)
8. [Rotating the client secret](#rotating-the-client-secret)
9. [The tray icon](#the-tray-icon)
10. [Command reference](#command-reference)
11. [Troubleshooting](#troubleshooting)

---

## How it works

A Windows service cannot map a drive the user can see. Drive letters belong to a *logon
session*, and a service runs as SYSTEM in session 0 — a drive it maps is invisible to
everyone. DriveMapper is therefore three pieces:

```
┌─────────────────────────────────────────────────────────────────────┐
│  Session 0                                                          │
│                                                                     │
│   DriveMapper.Service.exe   (LocalSystem, Automatic, auto-restart)  │
│     • SCM session events: logon, unlock, RDP reconnect              │
│     • network availability, resume from sleep, periodic re-check    │
│     • resolves Entra group membership (app-only Graph)              │
│                                                                     │
│                 │ WTSQueryUserToken + CreateProcessAsUser           │
└─────────────────┼───────────────────────────────────────────────────┘
                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Session 1..n   (the user's own desktop, the user's own token)      │
│                                                                     │
│   DriveMapper.Agent.exe     (short-lived, one run per trigger)      │
│     • applies the plan the service produced, then exits             │
│                                                                     │
│   DriveMapper.Tray.exe      (long-lived, optional)                  │
│     • "Map drives now" and a personal sync interval                 │
└─────────────────────────────────────────────────────────────────────┘
```

### Why this is more reliable than a scheduled task

| Failure mode | How DriveMapper handles it |
| --- | --- |
| Task didn't fire at logon | The SCM delivers session notifications directly — no scheduler queue to miss. |
| Ran before the network was up | Transient errors (53, 54, 1203, 1222) retry with backoff, and a run is re-triggered when the network returns or the machine resumes. |
| Task engine disabled, or the task got deleted | Windows service recovery: restart after 5 s, 15 s, then 60 s, counter resetting daily. |
| Drive vanished mid-session | Periodic re-check re-verifies every managed letter. |
| Drives missing because the user is an admin | The agent runs with the user's **filtered** (non-elevated) token — the one Explorer uses. |
| Can't reach Entra (offline, remote) | Last-known-good group membership is cached per user for 14 days. |
| Group membership genuinely unknown | Nothing is unmapped. "Unknown" is never treated as "not a member". |
| Device has no Primary Refresh Token | App-only auth needs no user token at all. |

---

## Before you start

### ⚠️ Authentication to the file share

DriveMapper decides **which** drives to map. Windows still has to authenticate the user to
the SMB share itself, and on a **cloud-only Entra-joined** device that is usually the
sticking point:

| Scenario | Works? |
| --- | --- |
| Azure Files with Entra Kerberos | **Yes** — the supported path for Entra-joined devices. |
| On-premises file server, hybrid-joined device with line of sight to a DC | **Yes** — ordinary Kerberos. |
| On-premises file server, **cloud-only** Entra-joined device | **No** — there is no Kerberos path. Every mapping fails with error 1326 or 5. |

If you are in the third case, fix that first. No drive mapper of any kind can work around
it; the options are Azure Files with Entra Kerberos, Entra Domain Services, or a stored
credential in Credential Manager.

Confirm quickly on a test device:

```powershell
net use X: \\fileserver\share
```

If that fails, the problem is share authentication, not DriveMapper.

### Requirements

- Windows 10 1809 (build 17763) or later, x64
- Devices joined to Entra ID (hybrid or cloud-only)
- Local administrator rights to install
- An Entra ID app registration (below)

---

## Entra ID setup

DriveMapper's recommended mode, `graphApplication`, has the **service** look up group
membership using application permissions and a credential only it can read. No user token,
no broker, no per-user consent — so hybrid and cloud-only devices behave identically.

### 1. Create the app registration

Entra admin center → **Identity → Applications → App registrations → New registration**

| Field | Value |
| --- | --- |
| Name | `DriveMapper` |
| Supported account types | **Single tenant** |
| Redirect URI | leave blank — app-only never uses one |

Record the **Application (client) ID** and **Directory (tenant) ID**.

### 2. Grant application permissions

**API permissions → Add a permission → Microsoft Graph → _Application_ permissions**

| Permission | Why |
| --- | --- |
| `GroupMember.Read.All` | Read the user's group membership |
| `User.Read.All` | Find the Entra user from a hybrid device's on-premises SID |

Then **Grant admin consent**.

> `User.Read.All` is only strictly needed for **hybrid** accounts. A cloud-only account's
> Windows SID (`S-1-12-1-…`) already contains the Entra object id, which DriveMapper reads
> directly with no lookup.

Leave **Allow public client flows** set to **No** — correct for a confidential app.

### 3. Create a credential

**Certificate (preferred).** Upload a certificate under **Certificates & secrets**, install
the private key into `LocalMachine\My` on each endpoint readable by SYSTEM, and set
`certificateThumbprint` in the config. Nothing sensitive then sits on disk.

**Client secret.** Create one under **Certificates & secrets** and copy the *Value* (shown
once). Supply it at install time, or store it later with `--set-secret`.

#### How the secret is protected

- Encrypted with **DPAPI, machine scope**, with application-specific entropy
- File ACL replaced: inheritance off, **SYSTEM and Administrators only**
- Read **only by the service**, which runs as LocalSystem
- The agent and tray — which run as the signed-in user — never read it

> Be clear-eyed about what DPAPI machine scope gives you: any process that can *read the
> file* can decrypt it. The ACL is the real control. Do not loosen it, and do not copy
> `secret.dat` between machines — it will not decrypt elsewhere. Anyone with local
> administrator rights on the endpoint can recover the secret. If that matters in your
> threat model, use a certificate with a non-exportable key.

### 4. Collect your group object IDs

```powershell
Connect-MgGraph -Scopes 'Group.Read.All'
Get-MgGroup -Filter "startswith(displayName,'SG-Drive-')" |
    Select-Object DisplayName, Id | Format-Table -AutoSize
```

---

## Configuration reference

`drivemap.json` is read from the first of these that exists:

1. `C:\ProgramData\DriveMapper\drivemap.json` — deployed configuration; survives upgrades
2. `C:\Program Files\DriveMapper\drivemap.json` — whatever shipped inside the MSI

```jsonc
{
  "schemaVersion": 1,
  "configVersion": "2026.08.21.1",

  "entra": {
    "tenantId": "00000000-0000-0000-0000-000000000000",
    "clientId": "00000000-0000-0000-0000-000000000000",
    "groupSource": "graphApplication",
    "useTransitiveGroups": true
  },

  "options": {
    "removeUnmatchedDrives": true,
    "persistent": false,
    "recheckIntervalMinutes": 60,
    "groupCacheMaxAgeHours": 336,
    "logRetentionDays": 30,
    "trayIcon": true,
    "minimumUserIntervalMinutes": 5
  },

  "mappings": [
    { "driveLetter": "S", "uncPath": "\\\\fs01\\shared", "label": "Shared", "allUsers": true },
    { "driveLetter": "F", "uncPath": "\\\\fs01\\finance", "label": "Finance",
      "groupId": "11111111-1111-1111-1111-111111111111" }
  ]
}
```

Comments and trailing commas are allowed.

### Top level

| Field | Meaning |
| --- | --- |
| `schemaVersion` | Must be `1`. |
| `configVersion` | Free-form version for this set of mappings, e.g. `"2026.08.21.1"`. Stamped into the registry on import so Intune can detect which configuration a device has. **Bump it whenever you change mappings.** |

### `mappings`

| Field | Meaning |
| --- | --- |
| `driveLetter` | `"F"` or `"F:"`. `A:`, `B:` and `C:` are rejected. |
| `uncPath` | `\\server\share`. Must be UNC. |
| `label` | Optional friendly name shown in Explorer. |
| `groupId` | **Preferred.** Entra group object ID — survives group renames. |
| `groupName` | Display name. Convenient, but matched only when group names are available. |
| `groupSid` | Windows/AD group SID. Only for `groupSource: windowsToken`. |
| `allUsers` | Map for everyone; no group lookup. |

### `options`

| Option | Default | Notes |
| --- | --- | --- |
| `removeUnmatchedDrives` | `true` | Disconnects a managed letter when the user no longer qualifies. Only touches letters named in this config, only when the letter points at that config's own target, and never forced while files are open. |
| `persistent` | `false` | Off on purpose: the agent remaps at every logon, and persistent mappings are what produce stale red-X letters. |
| `recheckIntervalMinutes` | `60` | Safety-net re-run. `0` disables. Measured from the last trigger of any kind. |
| `groupCacheMaxAgeHours` | `336` (14 days) | How long an offline logon may rely on cached membership. |
| `logRetentionDays` | `30` | `0` disables log cleanup. |
| `trayIcon` | `true` | Show the notification-area icon. |
| `minimumUserIntervalMinutes` | `5` | Floor for the interval a user may choose in the tray. |

### `groupSource`

| Value | Resolved by | Needs | Sees cloud-only Entra groups |
| --- | --- | --- | --- |
| `graphApplication` **(recommended)** | Service | Client secret or certificate | Yes |
| `windowsToken` | Agent | Nothing | **No** — logon-token groups only |
| `auto` | Agent | Nothing (Entra optional) | Only if delegated auth succeeds |
| `claimsThenGraph` / `claimsOnly` / `graphOnly` | Agent | A device Primary Refresh Token | Yes |

If your groups are managed **in Entra** rather than synced from AD, `windowsToken` will not
see them — Windows only puts local and on-premises AD groups in a logon token.

The delegated modes require the signed-in user to be able to mint a token on the device.
That fails on a domain-joined machine with no PRT, which is the usual cause of
*"Silent token acquisition needs user interaction"*.

---

## Installing

From an **elevated** prompt:

```powershell
msiexec /i DriveMapper.msi /qn SECRET="<entra client secret>"
```

| Property | Purpose |
| --- | --- |
| `SECRET` | Client secret. Encrypted to DPAPI during install; the plaintext never touches disk, and it is marked hidden so it stays out of MSI logs. |
| `SECRETFILE` | Path to a file containing the secret, if you would rather not put it on a command line. |
| `CONFIGFILE` | Path to a `drivemap.json`. If omitted, a `drivemap.json` sitting **next to the MSI** is picked up automatically. |

The configuration is **validated during install**. A typo fails the install with the exact
problem rather than silently mapping nothing at every logon. If validation fails during an
upgrade, the previous working configuration is restored.

Verify:

```powershell
cd "C:\Program Files\DriveMapper"
.\DriveMapper.Service.exe --test-graph     # is the credential valid?
.\DriveMapper.Service.exe --diagnose       # sessions, SIDs, groups, MAP/skip per mapping
```

`--diagnose` lists every active session with its SID, resolves each user's Entra groups, and
prints MAP or skip per mapping. It is the fastest way to confirm a deployment end to end.

### Uninstalling

```powershell
msiexec /x "{PRODUCT-CODE}" /qn
```

The ProductCode is regenerated for every build (correct for major upgrades), so take it from
`ProductCode.txt` in the release bundle rather than reusing a previous one.

Removes the service and the stored secret; logs are kept. Non-persistent drive letters clear
at the next sign-out.

---

## Deploying with Intune

Package the MSI as a **Win32 app**. Intune derives detection and uninstall from the product
code, so the app definition is only a few fields.

### Wrap it

```powershell
.\IntuneWinAppUtil.exe -c "C:\staging\DriveMapper" -s "DriveMapper.msi" -o "C:\staging\out"
```

Put `DriveMapper.msi` in the staging folder, plus `drivemap.json` if you want it picked up
automatically.

### App settings

| Field | Value |
| --- | --- |
| Install command | `msiexec /i "DriveMapper.msi" /qn SECRET="<entra client secret>"` |
| Uninstall command | `msiexec /x "{PRODUCT-CODE}" /qn` — the build prints the ProductCode, and writes it to `ProductCode.txt` beside the MSI |
| Install behavior | **System** |
| Detection rule | **MSI** → paste the product code |
| OS architecture | x64 |
| Minimum OS | Windows 10 1809 |

Assign to a **device** group. The app is machine-scoped; per-user behaviour comes from group
membership at run time, not from who the app is assigned to.

> **The install command line is stored in Intune** and readable by anyone with Intune admin
> access, and is briefly visible in the local process list. If that is unacceptable, use a
> certificate instead and pass no secret at all.

---

## Updating drive mappings

Ship the configuration as its **own Intune Win32 app**, separate from DriveMapper. Mapping
changes then need no new MSI and no reinstall.

Every successful import stamps `HKLM\SOFTWARE\DriveMapper`:

| Value | Meaning |
| --- | --- |
| `ConfigVersion` | the `configVersion` field from your JSON |
| `ConfigHash` | SHA-256 of the installed file |
| `ConfigInstalledUtc` | when it was applied |
| `ConfigMappingCount` | how many mappings it contains |

So detection is a **plain registry rule, no scripting**:

| Field | Value |
| --- | --- |
| Key path | `HKEY_LOCAL_MACHINE\SOFTWARE\DriveMapper` |
| Value name | `ConfigVersion` |
| Detection method | String comparison |
| Operator | Equals |
| Value | `2026.08.21.1` |
| Associated 32-bit app on 64-bit clients | **No** |

That last row matters: the service writes to the 64-bit view, and a 32-bit rule looks in
`WOW6432Node` and finds nothing.

### The update loop

1. Edit the mappings in `drivemap.json`.
2. **Bump `configVersion`.** Forgetting is the failure that matters — Intune would consider
   every device compliant and never deliver the change.
3. Repackage and upload as a new version of the config app.
4. Set the detection value to the new version.

Install command for the config app:

```
powershell.exe -ExecutionPolicy Bypass -NoProfile -Command "& 'C:\Program Files\DriveMapper\DriveMapper.Service.exe' --import-config '.\drivemap.json'"
```

Uninstall command: `cmd /c exit 0`. Set the main DriveMapper app as a **dependency** so
ordering is handled for you.

An invalid configuration is **rejected**: the install fails and the previous working config
stays in place. No restart is needed — the service rereads the config each cycle and the
agent on every run, so users pick the change up at their next sign-in, unlock, or re-check.

### Checking a device

```powershell
& "C:\Program Files\DriveMapper\DriveMapper.Service.exe" --config-status
```

Shows the registry stamp, the hash of the file on disk, whether the two still agree, and
which configuration is actually in effect.

### Stronger detection

Custom script detection can additionally verify the file still matches the hash recorded at
import and still parses, so a hand-edited device is reported as *not installed* and repaired:

```powershell
$out = & "C:\Program Files\DriveMapper\DriveMapper.Service.exe" --detect-config=2026.08.21.1
if ($LASTEXITCODE -ne 0) { exit 1 }
Write-Output $out
exit 0
```

Pin exact content instead with `--detect-config=sha256:<hash>`.

---

## Rotating the client secret

Push to every device; **no restart needed**, because the service builds a fresh Graph client
for each resolution:

```powershell
& "C:\Program Files\DriveMapper\DriveMapper.Service.exe" --new-secret=<new client secret>
```

It is **atomic and self-verifying**: the new secret is proved against Graph before being
committed, and the previous one is restored if it fails. A rotation pushed to a thousand
devices therefore either works or reports failure — it never leaves a machine holding a
credential that cannot authenticate.

Exit code 0 means verified, 1 means rolled back, so Intune reports the truth. Add
`--no-verify` to skip the check.

Client secrets expire. When Graph starts returning 401, rotate. Certificates avoid the
scramble.

---

## The tray icon

The service starts one `DriveMapper.Tray.exe` per interactive session. It offers:

- **Map drives now** — asks the service to run for that session immediately
- **Sync interval** — how often *this user's* session asks for a run: off, 5, 15, 30 or 60
  minutes, stored in `HKCU\Software\DriveMapper`. It cannot go below
  `minimumUserIntervalMinutes`, so a user can ask for their drives more often but can never
  relax what you configured. The machine-wide `recheckIntervalMinutes` is unaffected.
- **Open log folder**, and the last run's outcome in the tooltip and menu header

The tray holds **no privilege of its own**. Everything goes to the service over the named
pipe `\\.\pipe\DriveMapper.Control`, and the service derives which session a request applies
to from the *calling process*, never from the message — so a user can only ever remap their
own drives.

Turn it off fleet-wide with `"trayIcon": false`.

---

## Command reference

All from `C:\Program Files\DriveMapper`.

### Service — elevated

| Command | Purpose |
| --- | --- |
| `--diagnose` | Sessions, SIDs, resolved groups, MAP/skip per mapping |
| `--test-graph` | Prove the app-only credential works |
| `--test-graph --sid <SID>` | Resolve one specific user's groups |
| `--set-secret` | Store the client secret interactively (hidden input) |
| `--new-secret=<value>` | Rotate the secret, verified, with rollback |
| `--remove-secret` | Delete the stored secret |
| `--import-config <path>` | Validate and install a configuration |
| `--config-status` | Which configuration is deployed, and whether it has drifted |
| `--detect-config[=<version>]` | Intune detection; exit 0 when present and valid |
| `--post-install` | Recreate ProgramData directories, permissions, event log source |
| `--console --verbose` | Run the service logic in the foreground |

### Agent — as the affected user, **not** elevated

| Command | Purpose |
| --- | --- |
| `--console --verbose --whatif` | Show every decision, change nothing |
| `--console --verbose` | Run for real |
| `--list-groups` | Windows logon-token groups for this user |

Run the agent **non-elevated**. Mapped drives belong to the token that created them, so an
elevated run puts them where Explorer cannot see them.

With `groupSource: graphApplication` the agent has no credential of its own — use
`--diagnose` on the service, or `Restart-Service DriveMapper`.

Add `--trace` to either for raw MSAL output when unpicking an authentication failure.

---

## Troubleshooting

### Where to look, in order

```powershell
Get-Service DriveMapper | Format-List Name, Status, StartType

Get-Content "$env:ProgramData\DriveMapper\logs\service-$(Get-Date -f yyyyMMdd).log" -Tail 50
Get-Content "$env:ProgramData\DriveMapper\logs\agent-$env:USERNAME-$(Get-Date -f yyyyMMdd).log" -Tail 80

Get-EventLog -LogName Application -Source DriveMapper -Newest 20 | Format-Table -AutoSize
```

A healthy agent run reads roughly:

```
--- Agent run starting for CONTOSO\jbloggs (session 1) ---
Loaded 3 mapping(s) from C:\ProgramData\DriveMapper\drivemap.json; groupSource = GraphApplication
Applying plan from the service: 2 to map, 1 to remove (membership via graph-application, 14 group(s)).
Mapped S: to \\fs01\shared (Company Shared).
F: already points at \\fs01\finance; left as is.
--- Agent run complete: 2 matched, 2 mapped/verified, 0 removed, 0 skipped, 0 failed. ---
```

### Nothing happens at all

| Log line | Cause |
| --- | --- |
| *(service log empty)* | The service never started. Check `Get-Service DriveMapper`. |
| `Agent executable not found` | Partial install. Repair the MSI. |
| `WTSQueryUserToken failed … (error 1314)` | The service is not running as LocalSystem. Check `sc.exe qc DriveMapper`. |
| `Trigger (…) but no active user sessions` | Fired at the sign-in screen. Expected; a real logon follows. |

### Authentication failures

| Error | Meaning |
| --- | --- |
| `AADSTS7000218` | The app registration has **Allow public client flows = No** and you are using a *delegated* `groupSource`. Switch to `graphApplication` — no registration change needed. |
| `failed_to_acquire_token_silently_from_broker` | The Windows broker has no Entra account for this logon (no PRT). Check `dsregcmd /status`. Use `graphApplication`. |
| Graph `401` | The client secret expired or is wrong. Rotate it. |
| Graph `403` | Missing application permissions `GroupMember.Read.All` + `User.Read.All`, or admin consent was never granted. |
| `No Entra user has onPremisesSecurityIdentifier 'S-1-5-21-…'` | A hybrid account not synced to Entra ID, or `User.Read.All` is missing. |
| `No encrypted secret at …` | Run `--set-secret` elevated, or reinstall with `SECRET=`. |
| `could not be decrypted … different machine` | DPAPI machine keys do not travel. Re-run `--set-secret` locally; never copy `secret.dat` between endpoints. |

### Groups resolve but the drive does not map

| Error | Meaning | Usual cause |
| --- | --- | --- |
| 53 | Network path not found | Server unreachable, DNS, or off-VPN. Retried automatically. |
| 67 | Share name not found | Typo in `uncPath`, or the share was renamed. |
| 5 | Access denied | No NTFS/share rights. Group membership for *mapping* and rights on the *share* are two different things. |
| 85 | Drive letter already in use | Another tool or a local volume owns the letter. |
| 1219 | Conflicting credentials | An existing SMB session to the same server uses a different account. `net use \\server /delete`. |
| 1326 | Logon failure | See [Authentication to the file share](#️-authentication-to-the-file-share). |

### Drives are mapped but invisible in Explorer

Almost always an elevation split. DriveMapper already launches the agent with the user's
filtered token, so normal Explorer sees them. To make them visible from *elevated*
applications too:

```powershell
New-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' `
    -Name 'EnableLinkedConnections' -Value 1 -PropertyType DWord -Force
# reboot required
```

### Drives disappear unexpectedly

Look for `Removed X: (…) — user no longer qualifies.` That means resolution **succeeded**
and the user genuinely was not in the group. Check the `Membership source:` line — `cache:`
means Entra was unreachable.

To stop DriveMapper removing drives at all, set `"removeUnmatchedDrives": false`.

### Collecting logs

```powershell
$stamp = Get-Date -Format yyyyMMdd-HHmm
Compress-Archive -Path "$env:ProgramData\DriveMapper\logs\*" `
                 -DestinationPath "$env:TEMP\DriveMapper-logs-$stamp.zip"
```

Logs contain usernames, group object IDs and UNC paths. No tokens or secrets are ever
written, and MSAL PII logging is off unless you deliberately enable `entra.enablePiiLogging`
— remember to turn it back off.
