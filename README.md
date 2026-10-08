# Home SOC Analyst Lab

A self-built home lab simulating a SOC analyst environment: a Wazuh SIEM ingesting Sysmon telemetry from a Windows endpoint, with Atomic Red Team-generated attacks for real, MITRE ATT&CK-mapped alert triage practice.

**Goal:** hands-on experience with the tools and workflows a SOC analyst uses day to day — log ingestion, detection rules, and alert triage — built from scratch on consumer hardware (16GB RAM laptop).

---

## Architecture

```
Atomic Red Team  →  Sysmon  →  Wazuh Agent  →  Wazuh Manager  →  Dashboard / Alerts  →  Analyst Triage
 (simulates an       (logs        (forwards        (applies          (MITRE-mapped        (pivot to raw
  attack technique)   activity)    to manager)       rules)            alerts surface)      logs, verdict)
```

**Environment:**
- **Host:** 16GB RAM laptop, VirtualBox
- **Wazuh manager:** Ubuntu Server 22.04 VM (indexer + manager + dashboard, all-in-one install)
- **Victim endpoint:** Windows 10 VM running Sysmon (SwiftOnSecurity config) + Wazuh agent
- **Attack simulation:** Atomic Red Team (Red Canary), run locally on the Windows VM
- **Networking:** VirtualBox Host-Only Adapter (VM-to-VM/manager traffic) + NAT (internet access) — chosen after Bridged mode proved unreliable over Wi-Fi

---

## What This Demonstrates

- Deploying and configuring a SIEM (Wazuh) from scratch, including indexer/manager/dashboard components
- Configuring endpoint telemetry (Sysmon) with an industry-standard ruleset
- Wiring an agent's log collection to a specific Windows Event Log channel
- Generating realistic attack activity safely with an industry-standard tool (Atomic Red Team)
- Reading raw Sysmon/Wazuh event data and applying a structured triage framework:
  1. What fired, and why (rule description + severity)
  2. What actually happened (process, parent process, command line, target file)
  3. Is the source legitimate (signed binary, expected location)
  4. Does it match known/expected activity
  5. Does the MITRE ATT&CK mapping actually fit the evidence
  6. Verdict: true positive / false positive / benign-but-noteworthy / needs more data
- Real infrastructure troubleshooting under pressure (see below) — networking, OS-level config, and resource management, not just following a tutorial

---

## Build Notes & Troubleshooting

A few of the more instructive problems hit along the way — included because working through them was as valuable as the end result:

**Wi-Fi + VirtualBox Bridged networking don't mix.** VMs on Bridged mode couldn't obtain an IPv4 address over Wi-Fi (a common VirtualBox limitation — the virtual MAC address often can't complete DHCP over a Wi-Fi adapter). Solution: Host-Only Adapter (VM-to-VM/host traffic) + NAT (internet access) on both VMs instead.

**Wazuh indexer failed to initialize (`vm.max_map_count` too low).** OpenSearch, which powers the Wazuh indexer, requires a higher OS-level limit on memory-mapped regions than Ubuntu ships with by default. Fixed with `sudo sysctl -w vm.max_map_count=262144`, made persistent via `/etc/sysctl.conf`.

**Sysmon events weren't reaching Wazuh.** The agent's `ossec.conf` pointed at `Microsoft-Windows-Sysmon\Operational` (backslash) instead of the actual channel name, `Microsoft-Windows-Sysmon/Operational` (forward slash). Diagnosed by confirming Sysmon was logging locally (`Get-WinEvent`) while Wazuh showed nothing — isolating the break to the agent's channel config specifically.

**Windows VM account lacked Administrator rights**, discovered via `net user <username>` showing empty group membership. Traced to the account-creation flow during a Windows evaluation ISO setup; resolved with a clean reinstall using the offline/personal-use account path.

**Disk filled to 100%, crashing the indexer and manager.** Root cause: `/var/ossec/queue/vd_updater/tmp/contents` (Wazuh's vulnerability-detection feed updater) had ballooned to 16GB of uncleaned temporary data, likely from an interrupted update. Diagnosed with `du -h --max-depth=N`, drilling down level by level from `/` to the actual offending folder. Resolved by clearing the temp directory; Wazuh regenerates it safely on the next update cycle.

---

## Sample Triage Write-Up

**Alert:** `Executable file dropped in folder commonly used by malware` (Level 15)

**What fired:** `cleanmgr.exe` (Windows' built-in Disk Cleanup utility) created `WimProvider.dll` inside a randomly-named GUID folder under `%TEMP%`.

**Investigation:** This exact process/file/path combination is well-documented as routine Disk Cleanup behavior. However, the same pattern — `cleanmgr.exe` dropping a DLL into a predictable temp path — is also a documented historical UAC-bypass technique, which is why the rule fires at high severity regardless of intent.

**Verdict:** False positive in this instance (known, deliberate lab activity; legitimate Microsoft binary; expected path). Noted as **benign-but-noteworthy** rather than dismissed outright — the underlying technique is real, so the rule's severity is appropriate to keep, even though this specific trigger wasn't malicious.

*(This is the kind of ambiguous, evidence-based call that makes up most real SOC triage — not every alert resolves to a clean yes/no.)*

---

## What's Next

- [ ] Expand Atomic Red Team coverage beyond Discovery/Execution (Credential Access, Persistence, Defense Evasion)
- [ ] Add a Kali Linux attacker VM for manual, non-scripted attack practice
- [ ] Tune detection rules based on triage findings
- [ ] Practice triage on "quiet" baseline activity (no deliberate attacks) to build a sense of normal vs. anomalous

---

## Tools Used

| Tool | Purpose |
|---|---|
| [Wazuh](https://wazuh.com/) | SIEM — log indexing, detection rules, dashboard |
| [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) | Endpoint telemetry (process, file, network events) |
| [SwiftOnSecurity's Sysmon config](https://github.com/SwiftOnSecurity/sysmon-config) | Community-standard Sysmon ruleset |
| [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team) | Safe, realistic attack technique simulation |
| VirtualBox | Hypervisor |
