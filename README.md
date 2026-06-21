# Detection Engineering Home Lab
### Building visibility from scratch: Zeek + Elastic Security on a 3-VM range

---

## Why I built this

Triaging alerts all night teaches you how to read a SIEM. It doesn't teach you how
one gets built. I wanted to know what's actually happening underneath the dashboards
I stare at every shift — so I built the pipeline myself: network sensor, endpoint
agent, and the SIEM stitching them together, from a blank VM to a working detection
stack.

This isn't a guide that pretends everything worked the first time. It didn't. The
value here is in what broke and what I had to figure out to fix it.

---

## Lab Topology

| Host | Role | IP |
|---|---|---|
| Ubuntu 24.04 | Network Sensor (Zeek) | `192.168.125.134` |
| Parrot OS | Attacker / Adversary Simulation | `192.168.125.135` |
| Windows 11 | Target / Endpoint | `192.168.125.136` |

![ParrotOS Specs](screenshots/ParrotOS%20Specs.png)
![Windows 11 Specs](screenshots/Windows%2011%20specs.png)
![Ubuntu Specs](screenshots/Ubuntu%20specs.png)

All three VMs isolated on a NAT segment, 8GB RAM / multi-core each. Specs above are
intentionally over-provisioned for a lab this size — better to never bottleneck while
you're mid-investigation than to find out your VM chokes during a test.

---

## Phase 1 — Build the Range

Updated and upgraded both Linux distros first.

![Parrot Update](screenshots/parrot_update_upgrade.png)

### Problem: Neither Linux box could ping the Windows VM

Classic first-day-of-lab wall. Windows Defender Firewall blocks inbound ICMP echo
requests by default — this isn't a bug, it's Windows being Windows.

![Ubuntu Ping Parrot](screenshots/ubuntu_ping_parrot.png)
*Linux-to-Linux pings worked fine — confirms the network segment itself was healthy
before troubleshooting Windows.*

![Windows Ping Fail](screenshots/ubuntu_problem_unable%20to%20ping%20windows.png)

**Fix:** Added an inbound firewall rule for ICMPv4 via PowerShell:

```powershell
New-NetFirewallRule -DisplayName "Allow ICMPv4-In" -Protocol ICMPv4 -IcmpType 8 -Action Allow
```

![Both Pinging](screenshots/windows_can%20ping%20both%20OS.png)

### Disabling Windows Defender (intentionally, and for a reason)

I'd be running tools like Meterpreter later that Defender will — correctly — flag as
malicious. Rather than fight the AV mid-test, I disabled it deliberately through Group
Policy:

`gpedit.msc` → Administrative Templates → Windows Components → Microsoft Defender
Antivirus → **Turn off Microsoft Defender Antivirus** → Enabled, then under
**Real-time Protection** → disabled real-time protection and enabled behavior
monitoring.

![Defender Disabled Confirmed](screenshots/windows_disabled%20defender%20after%20applying%20g%5Bpo%5D.png)

Confirmed via Windows Security, which now shows real-time protection off with the
note *"This setting is managed by your administrator"* — proof the GPO took effect
rather than a manual toggle that Defender would silently revert.

> **Why this matters operationally:** disabling AV in a lab isn't about disabling
> security — it's about controlling *which* layer catches the activity. I wanted
> Elastic Defend and Sysmon to be my detection surface for this exercise, not
> Defender doing the catching invisibly before the data ever reached my SIEM.

---

## Phase 2 — Zeek: Network Visibility

Installed via the official Zeek repo on Ubuntu.

```bash
su root
echo 'deb https://download.opensuse.org/repositories/security:/zeek/xUbuntu_24.04/ /' | sudo tee /etc/apt/sources.list.d/security:zeek.list
curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_24.04/Release.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null
sudo apt update
sudo apt install zeek-8.0
```

### Problem: `curl` not found

Minimal Ubuntu installs don't ship curl by default. Easy fix, but the kind of thing
that stops a guided install cold if you've never hit it before.

![Curl Missing](screenshots/ubuntu_problem_curl%20is%20not%20installed.png)

```bash
apt install curl
```

### Configuration

**`networks.cfg`** — declared my lab's network as local so Zeek doesn't treat my own
traffic as external/unknown: `192.168.125.0/8`

![Networks Config](screenshots/ubuntu_zeek%20configuration_networks%20cfg.png)

**`node.cfg`** — pointed Zeek at the correct interface. Used `ip addr` to confirm mine
was `ens33`, not the default `eth0` in the template.

![Zeek Configuration](screenshots/ubuntu_zeek%20configuration.png)

### Problem: `zeekctl: command not found`

![Zeek Not Found](screenshots/ubuntu_problem_zeek%20not%20found.png)

Zeek wasn't symlinked to PATH. Ran it directly from its install location instead:

```bash
cd /opt/zeek/bin
./zeekctl
deploy
```

![Zeek Deployed](screenshots/ubuntu_solution_zeek%20not%20found.png)

Zeek deployed and started successfully. Snapshotted all three VMs at this point as a
clean baseline before touching Elastic.

---

## Phase 3 — Elastic Security: The SIEM Layer

### Problem: Serverless deployment was a dead end

Signed up for the 14-day Elastic trial and defaulted into a **serverless**
deployment. Spent real time hunting for an agent installer that, in serverless mode,
doesn't expose itself the way you'd expect from a traditional self-managed stack.
Got frustrated enough to consider scrapping the whole approach.

Stepped back, reconsidered the deployment model rather than the tooling, and switched
to **hosted deployment** with full SIEM capabilities — Elastic Defend included.

![Serverless Deployment](screenshots/elastic_create%20serverless.png)

> This was the most valuable failure in the whole lab. The fix wasn't a command —
> it was recognizing I'd picked the wrong deployment architecture for what I was
> trying to do, not a missing flag or broken install.

![Elastic Welcome](screenshots/elastic_welcome.png)

### Installing Elastic Agent on the Windows endpoint

```powershell
$ProgressPreference = 'SilentlyContinue'
Invoke-WebRequest -Uri https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-9.4.2-windows-x86_64.zip -OutFile elastic-agent-9.4.2-windows-x86_64.zip
Expand-Archive .\elastic-agent-9.4.2-windows-x86_64.zip -DestinationPath .
cd elastic-agent-9.4.2-windows-x86_64
.\elastic-agent.exe install --url=https://<fleet-url>:443 --enrollment-token=<token>
```

![Agent Installed Windows](screenshots/windows_installed%20elastic%20agent.png)
![Agent Confirmed](screenshots/elastic_agent%20confirmed.png)

### Problem: Agent enrolled, but no incoming data for 5+ minutes

The Fleet UI sat on *"Listening for incoming data from enrolled agents…"* with
nothing coming through. Worked through the standard troubleshooting checklist —
checked agent status, generated fresh Windows event activity, restarted the agent
service to force a new handshake.

![No Incoming Data](screenshots/elastic_problem_no%20incoming%20data.png)

None of it fixed it. The actual cause: **outbound firewall restrictions** blocking
ports 443 (HTTPS) and 8220 (Fleet Server) — the agent could enroll but couldn't
sustain the data channel.

Once outbound 443/8220 were confirmed open, logs started flooding in.

### Tuning Elastic Defend: Prevent → Detect

Switched Malware, Ransomware, Memory Threat, and Malicious Behavior protections from
**Prevent** to **Detect**. For a detection-engineering exercise, an AV that silently
blocks the thing you're trying to observe defeats the purpose — same logic as the
earlier Defender decision. I want visibility first, enforcement decisions second.

![Prevent to Detect](screenshots/elastic_endpoint%20setting_from%20prevent%20to%20detect.png)

---

## Phase 4 — Zeek → Elastic Integration

Zeek logs needed to ship in JSON for Elastic to parse them cleanly. The documented
approach (`@load policy/tuning/json-logs.zeek`) referenced a `defaults` policy folder
that no longer exists in newer Zeek 8.0 — it's been folded into Zeek's defaults
upstream. Used the direct redef instead, appended to `local.zeek`: redef LogAscii::use_json = T;

Verified by checking `conn.log` under `/opt/zeek/logs/current` for valid JSON
structure.

![Zeek JSON Verified](screenshots/ubuntu_verify%20that%20zeek%20json%20is%20working.png)

Created a dedicated Zeek integration policy in Elastic (`DEB_x86_64` build, matching
the Ubuntu sensor's architecture) and confirmed enrollment.

![Zeek Agent Enrolled](screenshots/ubuntu_successfuly%20installed%20elastic%20agent.png)

---

## Phase 5 — Proving the Pipeline: NMAP Validation

From Parrot OS, ran a service-detection scan against the Windows target to generate
network traffic Zeek would have to observe and categorize:

```bash
sudo nmap -sV 192.168.125.136
```

![Nmap Scan](screenshots/parrot_nmap%20done.png)

Confirmed in Elastic's built-in Zeek Log Overview dashboard — visible traffic spikes
correlating to the scan window, and individually verified via Discover using
`event.dataset:zeek.connection`.

![Nmap Logged Dashboard](screenshots/elastic_nmap%20trial%20logged.png)

First time seeing my own generated traffic land cleanly in a SIEM I built from the
ground up — genuinely felt like watching my first forwarder light up in an old
Splunk home lab, except this time I'd built every layer underneath it myself.

---

## Phase 6 — Endpoint Detection: EICAR Test

Downloaded Palo Alto's `wildfire-test-pe-file.exe` — a benign file designed to be
flagged as malicious by AV/EDR — to validate Elastic Defend's detection path on the
Windows endpoint.

![Wildfire Test File](screenshots/windows_wildfire%20test%20pe%20file.png)
![EICAR Detected](screenshots/elastic_EICAR%20File%20detected.png)

In Elastic's Alerts view, the file triggered a **Critical** severity alert with a
risk score of 99, full rule reasoning, and an interactive process-tree visualization
(the "Analyze Event" graph) showing the complete execution chain:
`userinit.exe → explorer.exe → wildfire-test-pe-file.exe → conhost.exe`

![Process Tree Analyzer](screenshots/elastic_EICAR%20File%20detected_amazing%20analyzer%20%20graph.png)

Pivoted into Discover and queried `process.name: "wildfire-test-pe-file.exe"`
directly — same investigative muscle memory as working a BOTS dataset in Splunk,
just on infrastructure I'd stood up myself.

### PowerShell activity logging

Ran a sequence of standard recon/exfil-adjacent commands to see how they'd surface
in the dataset:

```powershell
whoami
hostname
Get-ComputerInfo
ipconfig
Get-ComputerInfo > systeminformation.txt
```

Queried Discover for individual commands (`whoami.exe`) and then pivoted broader with
`process.parent.name: "powershell.exe"` to see the full parent-child chain across all
five commands in one view.

![PowerShell Chain](screenshots/elastic_powershell%20powershell.png)

---

## Phase 7 — Closing the Visibility Gap: Sysmon

Elastic Defend alone gave good alert-level detail, but I wanted deeper process-level
telemetry — registry, parent-child relationships, image loads at a finer grain than
the endpoint agent alone surfaces.

Installed **Sysmon (Sysinternals)** with a community SwiftOnSecurity-style config via
GitHub, added the Windows integration in Elastic (`event.dataset:
windows.sysmon_operational`), and re-ran the exact same EICAR + PowerShell sequence
to do a direct before/after comparison.

![Sysmon Event Data](screenshots/elastic_event%20data%20sysmon.png)

553 records returned, 100% attributed to `windows.sysmon_operational` — the same
activity that produced a handful of Elastic Defend alerts produced an order of
magnitude more granular telemetry once Sysmon was in the pipeline.

---

## Key Takeaways

- **Deployment architecture decisions compound.** The serverless-vs-hosted choice
  early on nearly derailed the entire lab — not because the tooling was broken, but
  because I'd matched the wrong deployment model to the goal.
- **"No data" is rarely the integration's fault.** Both major blockers I hit
  (Windows ping, Elastic Agent data flow) traced back to firewall rules, not
  misconfigured software. Network path first, application config second.
- **Prevent mode and Detect mode answer different questions.** Prevent tells you
  "it's blocked." Detect tells you "here's what it actually does." For learning
  detection engineering, you need the second one.
- **One data source is never enough.** Zeek gave me the network's story. Elastic
  Defend gave me the endpoint's alert story. Sysmon gave me the endpoint's *process*
  story. Each closed a gap the others couldn't see — which is the whole argument for
  defense in depth, just proven on my own infrastructure instead of read in a book.

---

## Stack

`Zeek 8.0` · `Elastic Security (Hosted)` · `Elastic Defend` · `Elastic Agent 9.4.2` ·
`Sysmon (Sysinternals)` · `Ubuntu 24.04` · `Parrot OS` · `Windows 11`

---

*Built and documented by John William Estacio ([@CleverSec](https://github.com/CleverSec))*
