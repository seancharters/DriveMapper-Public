# DriveMapper

Maps network drives for signed-in Windows users based on their **Entra ID group
membership**, driven by a Windows service rather than a logon script or scheduled task.

Works across a mixed fleet — hybrid-joined and cloud-only Entra-joined devices alike.

> **Downloads are on the [Releases](../../releases) page.** This repository holds the
> installer and documentation only; the source is maintained privately.

---

## Why a service

A Windows service cannot map a drive the user can see — drive letters belong to a *logon
session*, and a service runs as SYSTEM in session 0. DriveMapper is therefore split: a
service that watches for session events and resolves group membership, and a short-lived
agent it launches inside each user's session to do the actual mapping.

That structure is what makes it more dependable than the usual approaches:

| Failure mode | How DriveMapper handles it |
| --- | --- |
| Logon script or task didn't fire | The Service Control Manager delivers session notifications directly — no scheduler queue to miss. |
| Ran before the network was up | Transient errors retry with backoff, and a run is re-triggered when the network returns or the machine resumes. |
| Task engine disabled, or the task got deleted | Windows service recovery restarts it: 5 s, 15 s, then 60 s. |
| Drive vanished mid-session | A periodic re-check re-verifies every managed letter. |
| Drives missing because the user is an admin | The agent runs with the user's filtered token — the one Explorer uses. |
| Can't reach Entra ID (offline, remote) | Last-known-good group membership is cached per user. |
| Group membership genuinely unknown | Nothing is unmapped. "Unknown" is never treated as "not a member". |
| Device has no Primary Refresh Token | App-only authentication needs no user token at all. |

Users also get an optional notification-area icon with **Map drives now** and a personal
sync interval.

---

## Getting started

1. Download `DriveMapper.msi` and `drivemap.sample.json` from the
   [latest release](../../releases/latest).
2. Create an Entra ID app registration with the application permissions
   `GroupMember.Read.All` and `User.Read.All`, and grant admin consent.
3. Rename the sample to `drivemap.json`, set `tenantId`, `clientId` and your mappings.
4. Put it next to the MSI and install, elevated:

   ```powershell
   msiexec /i DriveMapper.msi /qn SECRET="<entra client secret>"
   ```

5. Verify:

   ```powershell
   cd "C:\Program Files\DriveMapper"
   .\DriveMapper.Service.exe --diagnose
   ```

Full instructions are in **[admin-guide.md](admin-guide.md)**, including Intune deployment,
updating drive mappings without reinstalling, and secret rotation.

---

## ⚠️ Read this before deploying

DriveMapper decides *which* drives to map. Windows still has to authenticate the user to the
SMB share, and that part depends on how the device is joined — not on this tool.

| Scenario | Works? |
| --- | --- |
| Azure Files with Entra Kerberos | **Yes** |
| On-premises file server, hybrid-joined device with line of sight to a DC | **Yes** |
| On-premises file server, cloud-only Entra-joined device | **Yes\*** — only when Cloud Kerberos Trust is configured |

\* Microsoft Entra Kerberos (cloud Kerberos trust) lets an Entra-joined device obtain a
Kerberos ticket for an on-premises file server. It requires the user to have a hybrid
identity synced from on-premises AD, line of sight to a domain controller, and the
`AzureADKerberos` object provisioned in AD. Without it, mappings to an on-premises
server fail with error 1326 or 5.

Confirm on a test device before deploying widely:

```powershell
net use X: \\fileserver\share
```

If that fails, the problem is share authentication, not DriveMapper.

---

## Requirements

- Windows 10 1809 (build 17763) or later, x64
- Devices joined to Entra ID (hybrid or cloud-only)
- An Entra ID app registration, admin-consented
- Local administrator rights to install

---

## Contents

| File | Purpose |
| --- | --- |
| [`admin-guide.md`](admin-guide.md) | Full deployment and administration guide |
| [`drivemap.sample.json`](drivemap.sample.json) | Annotated sample configuration |
| [Releases](../../releases) | `DriveMapper.msi` and checksums |
