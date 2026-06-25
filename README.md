# Detection Engineering: Zeek + Elastic Security on a 3-VM Range

![Project Status](https://img.shields.io/badge/Status-Foundation%20Complete-success)
![Zeek](https://img.shields.io/badge/NSM-Zeek-00ADD8)
![Elastic](https://img.shields.io/badge/SIEM-Elastic%20Security-005571)
![Sysmon](https://img.shields.io/badge/Telemetry-Sysmon-0078D6)

A detection home lab built to validate end-to-end visibility from network traffic
to endpoint process execution — network sensor, SIEM, and endpoint telemetry stood
up from scratch and proven to talk to each other before anything gets attacked.

---

## Project Overview

### Objective

Build a detection engineering range — network visibility, SIEM, endpoint telemetry —
from a blank set of VMs, and validate that each layer correctly generates and ships
data to the SIEM. This is the foundation phase. Part 2 (attack simulation + custom
detection rules) builds directly on this range.

### Key Components

- **Zeek** - Network traffic analysis and protocol-level logging
- **Elastic Security (Hosted)** - SIEM, log aggregation, and endpoint detection
- **Elastic Defend** - Endpoint protection and alerting
- **Sysmon** - Granular Windows process and registry telemetry

### Tech Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| Network IDS | Zeek | 8.0 |
| SIEM | Elastic Security (Hosted) | 9.x |
| Endpoint Agent | Elastic Agent | 9.4.2 |
| Endpoint Telemetry | Sysmon (Sysinternals) | Latest |
| Network Sensor OS | Ubuntu | 24.04 |
| Attack Simulation OS | Parrot OS | Latest |
| Target OS | Windows | 11 |
| Hypervisor | VMware Workstation Pro | Latest |

---

## Architecture

![Lab Setup Architecture](screenshots/architecture.png)

**Data Flow:**
1. Parrot OS generates network activity (Nmap scan) against the Windows target
2. Zeek on Ubuntu observes traffic on the segment, logs connection metadata in JSON
3. Elastic Agent + Sysmon on Windows capture process execution and auth events
4. Both sources ship to Elastic Security (Hosted), correlated in Discover and Alerts

---

## Implementation

### Phase 1 — Build the Range

Updated and upgraded both Linux distros before installing any tooling.

#### Problem: Neither Linux box could ping the Windows VM

Windows Defender Firewall blocks inbound ICMP echo requests by default. Before
assuming a network or adapter issue, tested Linux-to-Linux connectivity first to
isolate the variable — if Parrot could reach Ubuntu, the problem was specific to
the Windows host, not the segment.

![Ubuntu Ping Parrot](screenshots/ubuntu_ping_parrot.png)
*Linux-to-Linux pings confirmed the segment was healthy.*

![Windows Ping Fail](screenshots/ubuntu_problem_unable%20to%20ping%20windows.png)
*Ubuntu unable to reach Windows — confirmed Defender was the cause.*

**Fix:** Added an explicit inbound allow rule for ICMPv4 via PowerShell:

```powershell
New-NetFirewallRule -DisplayName "Allow ICMPv4-In" -Protocol ICMPv4 -IcmpType 8 -Action Allow
```

Bidirectional ping confirmed across all three hosts after the rule was applied.

#### Disabling Windows Defender (intentionally, and for a reason)

Later phases use tooling that Defender flags as malicious by design. Rather than
have Defender intercept activity before it reaches the SIEM, it was disabled via
Group Policy and the change verified through Windows Security:

`gpedit.msc` → Administrative Templates → Windows Components → Microsoft Defender
Antivirus → **Turn off Microsoft Defender Antivirus** → Enabled

Under **Real-time Protection**: disabled real-time protection, enabled behavior
monitoring.

![Defender Disabled Confirmed](screenshots/windows_disabled%20defender%20after%20applying%20gpedit.png)
*Windows Security confirms real-time protection off, managed by administrator —
proof the GPO took effect rather than a manual toggle Defender would silently revert.*

> **Why this matters operationally:** the detection surface under test here is
> Elastic Defend and Sysmon, not Defender. Leaving Defender active would mean
> activity gets caught upstream before it ever generates a log worth analyzing.

---

### Phase 2 — Zeek: Network Visibility

Installed via the official Zeek repo on Ubuntu:

```bash
su root
echo 'deb https://download.opensuse.org/repositories/security:/zeek/xUbuntu_24.04/ /' | sudo tee /etc/apt/sources.list.d/security:zeek.list
curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_24.04/Release.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null
sudo apt update
sudo apt install zeek-8.0
```

#### Problem: `curl` not found

`curl` is not installed by default on a minimal Ubuntu build, which blocks the
GPG key fetch mid-install.

![Curl Missing](screenshots/ubuntu_problem_curl%20is%20not%20installed.png)

```bash
apt install curl
```

#### Configuration

**`networks.cfg`** — declared `192.168.125.0/8` as local so Zeek classifies lab
traffic correctly instead of treating it as external/unknown:

![Networks Config](screenshots/ubuntu_zeek%20configuration_networks%20cfg.png)

**`node.cfg`** — corrected the listening interface from the template default (`eth0`)
to the actual adapter (`ens33`), confirmed via `ip addr`:

![Zeek Configuration](screenshots/ubuntu_zeek%20configuration.png)

#### Problem: `zeekctl: command not found`

![Zeek Not Found](screenshots/ubuntu_problem_zeek%20not%20found.png)

Zeek's binary exists at `/opt/zeek/bin` but isn't symlinked to PATH by default.
Ran it directly from the install location:

```bash
cd /opt/zeek/bin
./zeekctl
deploy
```

![Zeek Deployed](screenshots/ubuntu_solution_zeek%20not%20found.png)

Zeek deployed and running. Snapshotted all three VMs here as a clean baseline
before introducing Elastic.

---

### Phase 3 — Elastic Security: The SIEM Layer

#### Problem: Serverless deployment was a dead end

Signed up for the 14-day Elastic trial and defaulted into a **serverless**
deployment. Spent real time hunting for an agent installer that serverless mode
doesn't expose the same way a hosted deployment does — the agent management
workflow simply isn't there.

Stepped back, reconsidered the deployment model rather than the tooling, and
switched to **hosted deployment** with full SIEM capabilities — Elastic Defend
included.

![Serverless Deployment](screenshots/elastic_create%20serverless.png)

> The fix wasn't a command — it was recognizing the wrong deployment architecture
> had been chosen for the goal, not a missing flag or broken install.

![Elastic Welcome](screenshots/elastic_welcome.png)

#### Installing Elastic Agent on the Windows endpoint

```powershell
$ProgressPreference = 'SilentlyContinue'
Invoke-WebRequest -Uri https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-9.4.2-windows-x86_64.zip -OutFile elastic-agent-9.4.2-windows-x86_64.zip
Expand-Archive .\elastic-agent-9.4.2-windows-x86_64.zip -DestinationPath .
cd elastic-agent-9.4.2-windows-x86_64
.\elastic-agent.exe install --url=https://<fleet-url>:443 --enrollment-token=<token>
```

![Agent Confirmed](screenshots/elastic_agent%20confirmed.png)
*Elastic confirms enrollment from the Fleet side.*

#### Problem: Agent enrolled, but no incoming data for 5+ minutes

The Fleet UI sat on *"Listening for incoming data from enrolled agents…"* with
nothing arriving. Worked through the standard checklist — checked agent status
directly on the host, generated fresh Windows event activity, restarted the
agent service to force a new handshake. All came back clean, ruling out the
agent itself.

![No Incoming Data](screenshots/elastic_problem_no%20incoming%20data.png)

The actual cause: **outbound firewall restrictions** blocking ports 443 (HTTPS)
and 8220 (Fleet Server). The agent could complete enrollment but couldn't sustain
the data channel. Opening 443/8220 outbound resolved it immediately.

#### Tuning Elastic Defend: Prevent → Detect

Switched Malware, Ransomware, Memory Threat, and Malicious Behavior protections
from **Prevent** to **Detect** — consistent with the earlier Defender decision.
Automatic blocking removes activity before it can be observed and logged.

![Prevent to Detect](screenshots/elastic_endpoint%20setting_from%20prevent%20to%20detect.png)

---

### Phase 4 — Zeek → Elastic Integration

Zeek logs need to ship in JSON for Elastic to parse them cleanly. The documented
approach (`@load policy/tuning/json-logs.zeek`) references a `defaults` policy
folder that no longer exists in Zeek 8.0 — it was folded into Zeek's own defaults
upstream. Used the underlying config variable directly instead, appended to
`local.zeek`: redef LogAscii::use_json = T;
Verified by checking `conn.log` under `/opt/zeek/logs/current` for valid JSON
structure before wiring anything to Elastic.

![Zeek JSON Verified](screenshots/ubuntu_verify%20that%20zeek%20json%20is%20working.png)

Created a dedicated Zeek integration policy in Elastic (`DEB_x86_64` build,
matching the Ubuntu sensor's architecture) and confirmed enrollment.

![Zeek Agent Enrolled](screenshots/ubuntu_successfuly%20installed%20elastic%20agent.png)

---

### Phase 5 — Proving the Pipeline: Nmap Validation

From Parrot OS, ran a service-detection scan against the Windows target to
generate network traffic Zeek would have to observe and log:

```bash
sudo nmap -sV 192.168.125.136
```

![Nmap Scan](screenshots/parrot_nmap%20done.png)

Confirmed in Elastic's built-in Zeek Log Overview dashboard — visible traffic
spikes correlating to the scan window — and independently in Discover using
`event.dataset:zeek.connection`.

![Nmap Logged Dashboard](screenshots/elastic_nmap%20trial%20logged.png)

---

### Phase 6 — Endpoint Detection: EICAR Test

Downloaded Palo Alto's `wildfire-test-pe-file.exe` — a benign file built to
trigger AV/EDR detection — to validate Elastic Defend's detection path on the
Windows endpoint.

![Wildfire Test File](screenshots/windows_wildfire%20test%20pe%20file.png)
![EICAR Detected](screenshots/elastic_EICAR%20File%20detected.png)

Triggered a **Critical** severity alert, risk score 99, with a full interactive
process-tree visualization showing the execution chain:
`userinit.exe → explorer.exe → wildfire-test-pe-file.exe → conhost.exe`

![Process Tree Analyzer](screenshots/elastic_EICAR%20File%20detected_amazing%20analyzer%20%20graph.png)

#### PowerShell Activity Logging

Ran a sequence of recon and simulated exfil commands to test command-level
logging:

```powershell
whoami
hostname
Get-ComputerInfo
ipconfig
Get-ComputerInfo > systeminformation.txt
```

Queried individual commands via `process.name: "whoami.exe"` then pivoted to
`process.parent.name: "powershell.exe"` to surface the full parent-child chain
across all five commands in one view.

![PowerShell Chain](screenshots/elastic_powershell%20powershell.png)

---

### Phase 7 — Closing the Visibility Gap: Sysmon

Elastic Defend provided alert-level detail but limited granularity at the
process and registry level — enough to know something happened, not enough to
fully reconstruct how. Installed **Sysmon (Sysinternals)** with a community
SwiftOnSecurity-style config, added the Windows integration in Elastic
(`event.dataset: windows.sysmon_operational`), and re-ran the exact same
EICAR + PowerShell sequence for a direct before/after comparison.

![Sysmon Event Data](screenshots/elastic_event%20data%20sysmon.png)

553 records returned, 100% attributed to `windows.sysmon_operational` — the
same activity that produced a handful of Elastic Defend alerts produced
significantly deeper telemetry once Sysmon was in the pipeline.

---

## Key Takeaways

- **Deployment architecture decisions compound.** The serverless-vs-hosted
  choice early on nearly derailed the entire lab — not because the tooling was
  broken, but because the wrong deployment model was matched to the goal.
- **"No data" is rarely the integration's fault.** Both major blockers here
  (Windows ping, Elastic Agent data flow) traced back to firewall rules, not
  misconfigured software. Network path first, application config second.
- **Prevent mode and Detect mode answer different questions.** Prevent tells
  you "it's blocked." Detect tells you "here's what it actually does." For
  detection engineering, you need the second one.
- **One data source is never enough.** Zeek gave the network's story. Elastic
  Defend gave the endpoint's alert story. Sysmon gave the endpoint's *process*
  story. Each closed a gap the others couldn't see.

---

## What's Next

This range is the foundation for Phase 2:

- [ ] Run adversary simulation chains (Atomic Red Team) against the range
- [ ] Write and tune custom KQL detection rules against Zeek + Sysmon data
- [ ] Map validated techniques to MITRE ATT&CK (T1046, T1059, and beyond)
- [ ] Add Suricata alongside Zeek for signature-based network detection
- [ ] Build a unified Kibana dashboard combining Zeek, Sysmon, and Defend
- [ ] Document full attack → detection → response chains

---

## License

MIT License - See [LICENSE](/License) file for details

---

## Contact

**LinkedIn:** https://www.linkedin.com/in/johnwilliamestacio/

Questions? Open an issue or reach out directly.

---

## Acknowledgments

**Tech Stack Credits:**
- [Zeek](https://zeek.org/)
- [Elastic Security](https://www.elastic.co/security)
- [Sysinternals (Sysmon)](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [VMware](https://www.vmware.com/)
