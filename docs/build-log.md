# SOC Home Lab — Build Log

A running log of building a home SOC lab from scratch: a Windows 11 victim/endpoint VM generating telemetry, and an Ubuntu VM standing up as the SIEM/analyst machine. Entries are dated and written as things actually happened, including the mistakes — troubleshooting is part of the skill being demonstrated here, not just the finished result.

---

## 2026-09-12 — Environment setup, Sysmon install & verification

### Goal for this session
Get both VMs running and networked, then install Sysmon on the Windows endpoint so it produces rich, SOC-grade telemetry instead of default (sparse) Windows Event Logs.

### Environment
- Host: 32 GB RAM, VirtualBox
- **SOC-Windows** — Windows 11, TPM 2.0 + Secure Boot enabled, 8 GB RAM allocated
- **SOC-Ubuntu** — Ubuntu Server, 8 GB RAM allocated
- Both VMs connected on the same internal network (confirmed reachable via `ping`)

### What I did
1. Confirmed VM-to-VM networking with `ping` between SOC-Windows and SOC-Ubuntu.
2. Updated SOC-Ubuntu and installed base tooling:
3. Downloaded Sysmon (Microsoft Sysinternals) on SOC-Windows.
4. Installed Sysmon using the community-standard **SwiftOnSecurity config** rather than the bare default one:
5. Verified Sysmon was logging via:
6. Generated real activity (`calc.exe`) and confirmed Sysmon captured it, including the full parent-child process chain (`Image`, `ParentImage`, `CommandLine`, `Hashes`).

### Issues hit & how they were resolved
- **`sudo apt install ... && ...` chain threw "Unable to locate package install."** Fixed by running each `apt` command separately instead of chaining them.
- **`Get-WinEvent -LogName "Microsoft-Windows-Sysmon./Operational"` failed with `ObjectNotFound`.** A stray period before the slash in the log name — corrected to `Microsoft-Windows-Sysmon/Operational`.
- **Windows VM had no internet access** after setting up the internal VM-to-VM network. Fixed by adding a second NIC in VirtualBox set to NAT, leaving the original adapter on the internal network for VM-to-VM traffic.
- **Searching the most recent 20 Sysmon events didn't surface a `notepad.exe` launch.** Switched to filtering by process name directly instead of relying on recency:
### Status / next steps
- ✅ Windows endpoint generating and logging Sysmon telemetry
- ✅ VM-to-VM networking confirmed
- **Next:** install a SIEM on SOC-Ubuntu and forward Sysmon logs into it.

---

## 2026-09-12/13 — Wazuh SIEM install and agent deployment

### What I did
1. Switched SIEM choice from Splunk to Wazuh after the Splunk account signup got stuck in manual verification.
2. Installed Wazuh all-in-one (server + indexer + dashboard) on SOC-Ubuntu:
3. Logged into the Wazuh dashboard and deployed the Windows agent to SOC-Windows via the dashboard's generated install command.
4. Started the agent service (`WazuhSvc`) and confirmed it showed **active** in the dashboard's Endpoints page.
5. Edited `ossec.conf` to add a `<localfile>` block forwarding the Sysmon/Operational event channel specifically, since the agent doesn't forward it by default.

### Issues hit & how they were resolved
- **Ubuntu 25.04 isn't on Wazuh's officially supported OS list** (16.04–24.04 LTS only) — install proceeded anyway with a warning, no issues hit as a result, but noted as a risk for future troubleshooting.
- **`curl -sO` silently failed to save the file** — root cause was a typo (`-s0` instead of `-sO`, zero vs. capital O) combined with a hyphen mistyped as a period in the filename. Fixed by typing the command carefully and verifying with `ls -la` after every attempt rather than assuming success.
- **Dashboard login rejected the correct-looking password** — root cause was pulling the wrong user's password from `wazuh-passwords.txt` (there are multiple: `admin`, `wazuh-wui`, `kibanaserver`, etc.). Fixed by explicitly checking which line was labeled `admin`.
- **Agent showed "pending" instead of "active" after install** — the `WazuhSvc` service hadn't actually been started; running `Start-Service -Name WazuhSvc` resolved it.
- **After adding the Sysmon `<localfile>` block, the agent service wouldn't start at all** (`Start-Service` returned no error, but status stayed "Stopped"). Diagnosed by reading the agent's own log file directly:
which showed `ERROR: (1226): Error reading XML file 'ossec.conf': (line 0)`. Inspected the full config file with `Get-Content` and found the closing tag was missing its slash (`<localfile>` instead of `</localfile>`). Fixed the single character and the service started cleanly.

### Status
- ✅ Full pipeline working end-to-end: Sysmon → Wazuh agent → Wazuh manager → indexer → dashboard
- ✅ Agent shows active, Sysmon-sourced events confirmed searchable in the dashboard

---

## Earlier — Investigation reasoning practice (pre-lab)

Before the VMs were stood up, practiced the analyst reasoning process against a hypothetical scenario:

**Scenario:** User `SOCAnalyst` — 2 failed logons (4625), then a successful logon (4624) from the same internal source IP, Logon Type 3, followed 10 minutes later by `Get-Process` and `Get-ChildItem` via PowerShell. No privilege changes, no new accounts.

**Verdict:** Benign — reasoning: low failed-logon count, consistent internal source IP, a normal network-logon type, and only benign, read-only commands executed afterward with no privilege escalation or account creation.

This exercise established the core method used throughout the lab: **command + context + surrounding activity = verdict**, rather than judging any single event in isolation.

