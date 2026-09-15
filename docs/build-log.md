# Investigation 02 — Possible DLL Search Order Hijack

**Date:** 2026-09-15
**Source:** Wazuh SIEM, agent `DESKTOP-BIMNT3G` (SOC-Windows lab VM)
**Rule triggered:** Wazuh rule 92219 — "Possible DLL search order hijack"
**MITRE ATT&CK:** T1574.001 (DLL Search Order Hijacking), T1574.002 (DLL Side-Loading) — Persistence, Privilege Escalation, Defense Evasion
**Severity:** Level 6

## Alert

Wazuh's Threat Hunting view flagged a level 6 alert:

> Possible DLL search order hijack by `C:\Windows\SystemTemp\{92D70DCE-ED02-4F6B-B983-800AFAEDE1C3}\ssshim.dll` created in Windows root folder

This was the first real (non-manually-generated) alert produced by the lab's rule engine, sourced from a Sysmon Event ID 11 (File Create) event forwarded through the Wazuh agent.

## Evidence gathered

- **Creating process:** `C:\Windows\system32\svchost.exe`
- **File created:** `ssshim.dll`, inside a randomly GUID-named folder under `C:\Windows\SystemTemp`
- **Rule fire count:** `rule.firedtimes: 50` — this exact rule had already fired 50 times on this single machine
- **Available fields:** this was a File Create (Sysmon EID 11) record, not a Process Create (EID 1) record — no `commandLine` or `parentImage` fields were present, which is expected for this event type and confirmed by checking the raw JSON directly rather than assuming

## Reasoning

1. **Frequency is the strongest signal.** A single rule firing 50 times on a freshly built lab machine that hasn't had any deliberate attack activity run against it strongly points toward a repeating, benign OS behavior rather than 50 discrete intrusion attempts.
2. **Process identity supports this.** `svchost.exe` is a legitimate, heavily-used Windows system process — not inherently suspicious on its own, though it is a process attackers do sometimes abuse specifically because it's so trusted.
3. **Filename is a strong contextual clue.** `ssshim.dll` corresponds to Windows' Application Compatibility ("shim") subsystem, a known legitimate OS component that creates files in exactly this kind of temp/GUID-named pattern as part of normal operation.
4. **No corroborating evidence of an actual attack.** No unusual command line, no unexpected parent process, no follow-on network activity or new scheduled task — nothing found or expected that would corroborate malicious intent.
5. **Why the rule still fired despite being benign:** the underlying detection logic is intentionally broad — it looks for a *pattern* (a DLL created in a temp/GUID-named folder) that legitimate Windows subsystems and real attackers can both produce. This is a known and reasonable tradeoff in detection engineering: favoring recall (catching real hijacking attempts) at the cost of some false positives, which is exactly why triage by an analyst — rather than blind auto-escalation — matters.

## Verdict

**Benign.** Consistent with routine Windows Application Compatibility subsystem behavior, not an active DLL hijacking attempt.

## What would change this verdict

- The same pattern occurring on a process *other than* a core Windows binary
- Correlation with unusual outbound network connections around the same timestamp
- A newly created scheduled task, registry Run key, or service tied to the same file
- The DLL being created by a process the machine doesn't normally run (e.g. a browser, an Office app, or an unrecognized executable)

## What this demonstrates

- Reading and correlating Sysmon-derived SIEM alerts rather than raw logs
- Distinguishing a detection rule *firing* from an actual *confirmed incident*
- Using MITRE ATT&CK mapping as context, not as an automatic verdict
- Recognizing event-type limitations (File Create vs. Process Create) and confirming rather than assuming what evidence is actually available
