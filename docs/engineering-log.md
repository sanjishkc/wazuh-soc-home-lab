# Wazuh SOC Home Lab — Engineering & Troubleshooting Log

A chronological record of building a self-hosted Wazuh SIEM home lab: what I did, the problems I hit, how I diagnosed them, and how I fixed them. Written as I went.

**Author:** Sanjish
**Goal:** Stand up Wazuh (SIEM) in a VM, monitor a Windows endpoint, generate and detect attacks, map to MITRE ATT&CK, and publish it as a portfolio project.
**Hardware I started with:** laptop, 8 GB RAM, dual-core (later upgraded to 16 GB).

---

## Entry 0 — Design decision: single-VM architecture

**Situation:** Wazuh's all-in-one (indexer + manager + dashboard) really wants ~8 GB for itself, and my whole laptop was only 8 GB.

**Decision:** Run only the Wazuh **server** in a VM, and use my **Windows host itself** as the monitored endpoint via a lightweight agent — instead of a second VM I couldn't afford. Rule: only boot the VM when actively working the lab.

**Lesson:** Right-size the architecture to the hardware. A constrained lab is still a lab.

---

## Entry 1 — Version choices

- **Ubuntu:** the download page defaulted to 26.04 LTS, but Wazuh 4.14 officially supports **24.04 LTS**, which also has far more community troubleshooting online. Grabbed `ubuntu-24.04.4-live-server-amd64.iso` from `releases.ubuntu.com/24.04.4/` (had to go to the releases page since the main page pushes the newest release). Server, not Desktop, to save RAM.
- **Wazuh:** confirmed current stable was **4.14.6**. The install URL `packages.wazuh.com/4.14/...` pulls the latest 4.14.x automatically.

**Lesson:** newest isn't always right — match the OS to what the tool officially supports.

---

## Entry 2 — Phase 0 prep

- Verified VT-x (hardware virtualization) — since VirtualBox already ran 64-bit VMs, it was already enabled.
- Decided the VM lives on the **D: drive** (folder `D:\VMs\Wazuh`) to keep it off the C: OS drive. ~250 GB free on D:.
- VirtualBox itself is installed on C: — that's fine, it's only the app; the VM's disk is what matters and that goes on D:.

---

## Entry 3 — Phase 1: building the VM (VirtualBox 7.2)

**Did:** Created the `Wazuh-Server` VM with the 24.04.4 ISO attached, 2 vCPU, 50 GB dynamic disk.

**Problem:** RAM was set to 2 GB by default. **Fix:** bumped it to 4 GB (the indexer floor).

**Problem:** In the VM's Network settings, the "Attached to" dropdown only showed **NAT** and **Bridged Adapter** — no Host-only option. I needed a second, contained network between the host and VM.

**Diagnosis:** Two VirtualBox 7.2 quirks:
1. The global Network Manager had moved out of the sidebar menus, making it hard to find.
2. The VM's Network settings only show **Adapter 1** in "Basic" mode — you have to switch to **Expert** mode to see Adapters 2–4.

**Fix:** Switched the settings dialog to **Expert** mode, opened **Adapter 2**, and set it to **Host-only Adapter** — the default `VirtualBox Host-Only Ethernet Adapter` already existed, so nothing extra to create. Final: Adapter 1 = NAT (internet for downloads), Adapter 2 = Host-only (contained host↔VM link).

**Lesson:** VirtualBox 7.x hides multi-adapter config behind "Expert" mode. If an option seems missing, check for a Basic/Expert toggle.

---

## Entry 4 — Installing Ubuntu Server

**Did:** Booted the ISO and ran the installer. Chose Ubuntu Server (not minimized), left networking on DHCP, used the whole disk (default LVM), created my user, **enabled OpenSSH server** during install (so I could SSH in later instead of typing in the console).

**Problem:** Got confused at the **Proxy configuration** screen and a Help menu overlay got stuck.

**Fix:** Proxy is left **blank** for a home lab (no proxy). Cleared the stuck menu and continued. Finished install, ran `sudo apt update && sudo apt upgrade -y`, noted the host-only IP (`192.168.56.102`), and took a snapshot named **"Clean Ubuntu"** as a rollback point.

**Lesson:** snapshots after a clean OS install are worth the 30 seconds — I leaned on this one hard later.

---

## Entry 5 — Swap file (low-RAM safety net)

**Did:** Added a swap file so the kernel wouldn't hard-kill processes under memory spikes:
```
sudo fallocate -l 3G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```
Verified with `free -h` and `swapon --show`.

**Lesson:** swap = disk used as overflow RAM. On a low-memory box it prevents out-of-memory crashes (at the cost of speed). The `/etc/fstab` line makes it survive reboots.

---

## Entry 6 — Phase 2: the Wazuh dashboard that wouldn't install (the big one)

This took 4 failed installs to crack. Documenting it properly because the diagnostic path is the whole point.

**What I ran each time:**
```
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

**Symptom:** Every run installed the **indexer, manager, and filebeat fine**, then **failed at the dashboard step** and auto-rolled-back the entire install. Same spot every time.

### Wrong theory #1 — memory
I assumed the dashboard's memory-heavy "optimize" step was getting OOM-killed. So I:
- added swap,
- bumped the VM 4 GB → 5 GB,
- eventually **upgraded my laptop to 16 GB** and gave the VM **8 GB**.

**The dashboard still failed at 8 GB.** That ruled memory out — if 8 GB doesn't fix it, it isn't memory.

### Reading the actual log (the turning point)
Instead of guessing again, I filtered the install log:
```
sudo grep -iE "error|fail|timeout|refused|cannot|unable|killed|not running" /var/log/wazuh-install.log | tail -40
```
The real error was right there:
```
E: Write error - write (28: No space left on device)
```
**It was never RAM or CPU — the disk was full.**

### Root cause — the Ubuntu LVM "half the disk" trap
Checked the storage:
```
df -h /      # root was only 27G, and filled during install
sudo vgs     # VSize 54G, VFree 27G  <-- half the disk was unallocated!
sudo lvs     # ubuntu-lv was only ~27G
```
Ubuntu Server's guided LVM install only allocated **~27 GB of the 50 GB disk** to the root filesystem and left the other ~27 GB unused in the volume group. Wazuh filled the 27 GB mid-install and died.

### The fix — grow the volume, then the filesystem
```
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv   # grow the logical volume into the free space
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv               # grow the ext4 filesystem to fill it
df -h /                                               # now 54G, 43G free
```
Two layers: the LV is the container, the filesystem sits inside it — you grow both.

**Lesson (my best interview story):** Don't chase the *assumed* cause. I lost a lot of time on RAM (even bought more) when the real problem was disk space from Ubuntu's default LVM only using half the disk. **Reading the log is what solved it.**

---

## Entry 7 — Cleaning up broken leftovers before the final install

**Problem:** After all the failed/rolled-back installs, the reinstall was blocked by two things:
1. `ERROR: The Wazuh manager package could not be removed` — a half-installed `wazuh-manager` package whose uninstall script failed (exit 127) because its files were already gone.
2. `ERROR: Port 1515 / 55000 is being used` — leftover manager processes still running and holding the enrollment/API ports.

**Fix (manual cleanup over SSH):**
```
sudo pkill -9 -f wazuh                                  # free ports 1515/55000
sudo ss -tlnp | grep -E ':1515|:55000'                  # confirm ports free (no output)
sudo rm -f /var/lib/dpkg/info/wazuh-manager.prerm /var/lib/dpkg/info/wazuh-manager.postrm  # blank the broken removal scripts
sudo dpkg --purge --force-all wazuh-manager             # force-remove the package
sudo rm -rf /var/ossec                                  # remove leftover data
dpkg -l | grep wazuh || echo clean                      # verify: clean
```

**Lesson:** a broken dpkg package won't remove if its own removal script errors. Blank the `prerm`/`postrm` info scripts, then `dpkg --purge --force-all`. And "port in use" errors usually mean a leftover process — kill it first.

---

## Entry 8 — Successful install

**Did:** Re-ran `sudo bash ./wazuh-install.sh -a` on the clean box with the resized disk. **It finished.**
- Dashboard installed and initialized.
- Saved the admin password from the summary.
- After a reboot, confirmed all three services `active (running)`:
  ```
  sudo systemctl status wazuh-indexer wazuh-manager wazuh-dashboard
  ```
  (indexer ~1.5 G, manager ~1.3 G, dashboard ~0.3 G — ~3 G total, comfortable on 8 GB.)

**Minor note:** saw two one-off `I/O error, dev sda` messages on the console during the heavy install. They never recurred and the guest disk is healthy — a transient hiccup under load, not a fault.

---

## Entry 9 — Reaching the dashboard (http vs https)

**Problem:** `http://192.168.56.102:443` gave "page not found," even though the host pinged fine.

**Diagnosis:** Port 443 is **HTTPS**. I was sending plain **HTTP** to an encrypted port.

**Fix:** used `https://192.168.56.102` (drop the `:443`, browsers use it automatically for https). Got the self-signed-cert warning → Advanced → Proceed → logged in as `admin`. Dashboard loaded.

**Lesson:** `:443 = https://`, `:80 = http://`. Wazuh's dashboard is HTTPS-only.

---

## Entry 10 — Phase 3 start: onboarding the Windows endpoint

**Did:**
- Used the dashboard's **Deploy new agent** wizard, set the server address to the manager's host-only IP `192.168.56.102`, and installed the Wazuh agent on my Windows host from an **admin PowerShell**.
- Took a snapshot named **"wazuh-working"** — full stack running + agent added — as a safe restore point.

**Next steps:**
- Confirm the agent shows **Active** in the dashboard.
- Install **Sysmon** (SwiftOnSecurity config) for richer Windows telemetry, and configure Wazuh to ingest the Sysmon channel.
- Then Phase 4: generate contained attacks, watch the alerts fire, write a custom rule, tune a false positive, and map everything to MITRE ATT&CK.

---

## Entry 11 — Dashboard API auth-token error (post-restart)

**Problem:** After restarting only the manager, the dashboard showed `[API connection] No API available to connect`, then after a retry, `AxiosError: Error getting the authorization token (3000 ... run as)`.

**Diagnosis:** The dashboard authenticates to the manager API (port 55000) for a token; the `run_as` capability check failed because the manager's API auth context and the indexer's security were out of sync after restarting only one service.

**Fix:** Restarted the whole stack **in order** — indexer → manager → dashboard (waiting between each) — then reloaded. Token connected fine.

**Lesson:** After an API/auth-token error, restart the stack in order (indexer first, dashboard last), not just one service.

---

## Entry 12 — File Integrity Monitoring (FIM) test

**Goal:** Prove Wazuh detects and reports any change in a watched folder.

**Did:**
- Added a real-time watch to the Windows agent config (`C:\Program Files (x86)\ossec-agent\ossec.conf`) inside the `<syscheck>` block:
  ```xml
  <directories check_all="yes" realtime="yes" report_changes="yes">C:\test</directories>
  ```
- Restarted the agent and created / modified / deleted a test file (`C:\test\secret.txt`).

**Problem hit:** `Restart-Service WazuhSvc` failed with "Cannot open WazuhSvc service" — an **access-denied** error because the PowerShell window wasn't elevated. **Fix:** ran PowerShell **as Administrator**, then it restarted fine.

**Result:** All three events fired in the dashboard's File Integrity Monitoring view — file **added** (rule 554), **modified** (rule 550, with content diff), **deleted** (rule 553) — with paths, hashes, and timestamps. Wazuh's default Windows monitoring also flagged registry changes (rule 751).

**Follow-up confusion:** the events seemed to "disappear" — turned out to be the dashboard's **time-range filter**, not data loss. Widening the top-right time picker brought them back.

**Lessons:**
- Windows service control (`Restart-Service WazuhSvc`) needs an **Administrator** PowerShell.
- In Wazuh/OpenSearch dashboards, "logs vanished" almost always means the **time range or a filter**, not deleted data — check the top-right time picker first.

---

## Entry 13 — Sysmon integration (Phase 3 complete)

**Goal:** Add deep endpoint telemetry (process creation with command lines + hashes) beyond default Windows logs, so Phase 4 detections have real substance.

**Did:**
- Downloaded Sysmon (Sysinternals) + the SwiftOnSecurity config (`sysmonconfig-export.xml`).
- Installed with the config (admin PowerShell): `.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml`.
- Added a standalone `<localfile>` block to the agent's `ossec.conf` to collect the channel:
  ```xml
  <localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
  ```
- Saved (admin Notepad), `Restart-Service WazuhSvc`.

**Result:** Events flowing under `rule.groups: sysmon` — Process Create (Event ID 1) with rich fields: `image`, `commandLine`, `parentImage`, `hashes`, `integrityLevel`.

**Lessons:** one `<localfile>` block = one log source (order doesn't matter). Sysmon's `commandLine` field is what turns "a process ran" into "*this exact command* ran" — the difference between noise and a real detection.

---

## Entry 14 — Phase 4 / Detection 1: New local user creation (T1136)

**Simulation (admin PowerShell, contained, on my own host):**
```
net user hacker Password123 /add
net localgroup administrators hacker /add
net user hacker /delete            # cleanup
```

**Result:** One action lit up a whole cluster of correlated alerts across two data sources:
- **"Administrators Group Changed"** — `rule.id 60154`, **level 12** (critical band), Windows Event **4732** (member added). This is the headline / escalate-worthy alert. `firedtimes: 2` — it fired for both the add (4732) and the cleanup removal (4733), so I captured the full lifecycle.
- Supporting: "User account enabled or created" (4722), "User account changed" (4738), "Users Group Changed", plus Sysmon Event ID 1 rows showing the literal command line `net localgroup administrators hacker /add`.

**Reading the level-12 alert (triage practice):** who = `subjectUserName: LENOVO`; what = member added/removed; where = group `Administrators`, `targetSid S-1-5-32-544` (the universal built-in local Administrators SID); the member = SID …-1003 (the `hacker` account).

**Detection-engineering observation:** Wazuh tagged this rule **T1484 (Domain Policy Modification)** — imprecise for a *local* admin-group change; **T1098 (Account Manipulation)** fits better. Flagged as a candidate for the custom rule. Also noticed the rule auto-maps to compliance controls (PCI-DSS 10.2.5, NIST 800-53, HIPAA, GDPR).

**Lesson:** the Windows Security event says *something changed*; the Sysmon event says *what command changed it*. Together = a defensible finding.

---

## Entry 15 — Custom detection rule: encoded PowerShell, parent-agnostic

**Approach:** Read the stock rule that caught my Detection 2 (rule **92057**, `/var/ossec/ruleset/rules/0800-sysmon_id_1.xml`) and found **two blind spots**:
1. It requires `parentImage` = `powershell.exe` → misses encoded PowerShell launched from `cmd.exe`, a Word macro, `wscript`, etc.
2. Its regex alternation lists `en|enco|encode|encodedcommand` but **skips `enc` and `encoded`** — the most common abbreviations.

**Custom rule (in `/var/ossec/etc/rules/local_rules.xml`, id 100100, level 12):** dropped the `parentImage` condition and added `enc|encod|encoded` to the pattern.

**Test:** ran `powershell.exe -nop -w hidden -enc <base64>` from **cmd.exe** → rule **100100 fired at level 12** where the stock 92057 stayed silent (wrong parent + missing abbreviation). Custom rule catches obfuscation the default rule misses.

**Interview line:** "I analyzed Wazuh's built-in encoded-PowerShell rule, found it only fires when PowerShell is the parent and missed the `-enc`/`-encoded` abbreviations, and wrote a custom rule that detects encoded PowerShell regardless of parent process."

---

## Entry 16 — Tuning a false positive: a critical-severity FP

**Found it by ranking alerts:** ~83% of alerts were low-level (3–5) noise, but **level 15 (critical) was 4.6% (~25 alerts/24h)** — critical alerts should be rare, so that was the red flag.

**Investigation:** the level-15 alerts were rule **92213** "Executable file dropped in folder commonly used by malware" (T1105), firing on files like `C:\Users\...\AppData\Local\Temp\__PSScriptPolicyTest_<random>.<random>.ps1`. The random-looking filenames *almost* looked like DNS-exfil encoding — but verified it was benign: **PowerShell's own execution-policy self-test** (it writes a throwaway `.ps1` to Temp, runs it to check the policy, deletes it). Local file-create by the signed system `powershell.exe`, no network, random temp naming — not exfil. Rule 92213's broad "any script in Temp" heuristic just can't tell it apart from a real payload.

**Surgical tune (id 100200, level 0):**
```xml
<rule id="100200" level="0">
  <if_sid>92213</if_sid>
  <field name="win.eventdata.targetFilename" type="pcre2">(?i)__PSScriptPolicyTest_</field>
  <description>Benign PowerShell execution-policy self-test in Temp - tuned FP [custom]</description>
</rule>
```

**Verified both directions:** after restart, PowerShell self-tests no longer alert — but dropping a real `ero-virus.bat` into Temp **still fired 92213 at level 15**. Silenced the noise, kept the signal.

**Lesson:** a false positive at level 15 is worse than a thousand at level 3 — it erodes trust in the critical queue. And tune *surgically* (`if_sid` + a `field` condition + `level 0`), never blanket-mute a rule that also catches real threats.

---

## Running list of lessons (quick reference)

1. Right-size the architecture to the hardware (single-VM design for 8 GB).
2. Match the OS version to what the tool officially supports (Ubuntu 24.04 for Wazuh 4.14).
3. VirtualBox 7.x hides multi-NIC config behind **Expert** mode.
4. Take a snapshot right after a clean OS install.
5. Swap is a cheap safety net on low-RAM boxes.
6. **Don't chase the assumed cause — read the actual log.** (RAM vs the real disk-space issue.)
7. Ubuntu's guided LVM install often only uses half the disk — `lvextend` + `resize2fs` to reclaim it.
8. Fix a broken dpkg package by blanking its `prerm`/`postrm` scripts, then force-purge.
9. `:443 = https`, not http.
10. After an API auth-token error, restart the stack **in order** (indexer → manager → dashboard).
11. Windows service commands (`Restart-Service WazuhSvc`) require an **Administrator** PowerShell.
12. "Logs disappeared" in the dashboard = **time range / filter**, not data loss. Check the time picker first.
13. One `<localfile>` block per source; Sysmon's `commandLine` is the field that makes detections real.
14. `S-1-5-32-544` = the built-in local **Administrators** group SID on every Windows box — a fast tell in alerts.
15. Wazuh's stock rule→ATT&CK mapping isn't always precise (local admin change tagged T1484 vs better T1098) — a reason to write custom rules.
16. Read the built-in rule before writing or tuning — its logic shows you the exact gap to fill.
17. `__PSScriptPolicyTest_*.ps1` in Temp = benign PowerShell execution-policy self-test — a very common false positive, not malware/exfil.
18. Tune surgically: `<if_sid>` + a `<field>` condition + `level="0"` silences the benign case while keeping the rule live for real threats.
