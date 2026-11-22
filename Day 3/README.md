# Day 3: Sysmon, Universal Forwarder, and Log Ingestion

Part of my [Home SOC Lab](../README.md) project. Day 3 connects the Windows endpoint to Splunk: Sysmon adds detailed endpoint telemetry, and the Splunk Universal Forwarder ships Windows Event Logs and Sysmon events to the `endpoint` index on the Splunk server.

**End result:**

```
Windows 11 (Sysmon + Event Logs) → Universal Forwarder → TCP 9997 → Splunk → index=endpoint
```

---

## What I did

### 1. Installed VirtualBox Guest Additions on Windows

Improved display performance and enabled full-screen and clipboard sharing. Located the Guest Additions CD drive from PowerShell and ran the installer from it.

```powershell
Get-Volume | Where-Object DriveType -eq 'CD-ROM'
cd D:
.\VBoxWindowsAdditions.exe
```

### 2. Installed Sysmon

[Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) (Microsoft Sysinternals) logs detailed endpoint activity that the default Windows logs miss: process creation with full command lines, network connections, registry changes, and more.

- Downloaded Sysmon and used **`Sysmon64.exe`** (64-bit x86; `Sysmon64a.exe` is for ARM, and `Sysmon.exe` is 32-bit)
- Used the community **SwiftOnSecurity sysmon-config** to reduce noise, and confirmed it was the raw `.xml` file rather than the saved GitHub web page
- Installed from an elevated PowerShell:

```powershell
.\Sysmon64.exe -accepteula -i .\sysmonconfig-export.xml
```

Output confirmed the config validated, the `SysmonDrv` kernel driver loaded, and the `Sysmon64` service started.

### 3. Verified Sysmon in Event Viewer

Opened **Applications and Services Logs → Microsoft → Windows → Sysmon → Operational**, launched Notepad, and found the matching **Event ID 1 (Process Creation)**, including the `Image`, `CommandLine`, `ParentImage`, `User`, and file `Hashes` fields.

### 4. Tested the network path before installing the forwarder

Confirmed the Windows VM could reach Splunk's receiving port:

```powershell
Test-NetConnection -ComputerName 10.0.2.100 -Port 9997
```

### 5. Installed the Splunk Universal Forwarder

- Installed the Windows 64-bit `.msi`, pointed at an on-premises Splunk Enterprise instance
- **Receiving indexer:** `10.0.2.100:9997`
- **Deployment server:** left blank (not needed for a single forwarder)
- **Windows Event Log checkboxes:** left unchecked, so I could configure inputs manually and send them to a dedicated index
- The service runs as **`NT SERVICE\SplunkForwarder`**, a low-privilege virtual service account (least privilege), instead of Local System

The installer generated `outputs.conf` (the "where"):

```ini
# C:\Program Files\SplunkUniversalForwarder\etc\system\local\outputs.conf
[tcpout]
defaultGroup = default-autolb-group

[tcpout:default-autolb-group]
server = 10.0.2.100:9997

[tcpout-server://10.0.2.100:9997]
```

### 6. Configured the inputs

Wrote `inputs.conf` by hand (the "what"), sending four logs to the `endpoint` index. Sysmon events are sent as XML (`renderXml = true`) to preserve full detail and to match what the Splunk Add-on for Sysmon expects.

```ini
# C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
[WinEventLog://Application]
index = endpoint
disabled = 0

[WinEventLog://Security]
index = endpoint
disabled = 0

[WinEventLog://System]
index = endpoint
disabled = 0

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = endpoint
disabled = 0
renderXml = true
```

Restarted the `SplunkForwarder` service so it loaded the new configuration.

### 7. Verified ingestion in Splunk

```spl
index=_internal | stats count by host
```
Confirms which machines are sending data. Forwarders send their own internal logs even before any inputs are configured, which makes this a quick connectivity check.

```spl
index=endpoint
```
Confirmed events from all four sources:

| Source | Contents |
|---|---|
| `WinEventLog:Application` | Application events |
| `WinEventLog:Security` | Logons, failed logons, account changes |
| `WinEventLog:System` | Services, drivers, shutdowns |
| `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational` | Sysmon telemetry |

---

## Troubleshooting Log

| # | Problem | Root Cause | Fix |
|---|---|---|---|
| 1 | `wmic` not recognized | `wmic` has been removed from recent Windows 11 builds | Used the PowerShell replacement `Get-Volume` |
| 2 | `Get-Volume` not recognized | Ran it in Command Prompt instead of PowerShell | Switched to PowerShell (prompt starts with `PS`) |
| 3 | `cd D` and `\VBoxWindowsAdditions.exe` failed | Drive letters need a colon (`D:`), and a path starting with `\` refers to the root of the *current* drive (`C:`) | Used `cd D:` and ran the installer from there |
| 4 | `sysmon64` not recognized | PowerShell doesn't run programs from the current folder without an explicit path (a security feature) | Ran it as `.\Sysmon64.exe` |
| 5 | `Test-NetConnection` returned `DestinationHostUnreachable` | The Splunk server VM was powered off | Started the VM |
| 6 | Ping succeeded but `TcpTestSucceeded : False` | Splunk wasn't running, and receiving on 9997 wasn't active | Started Splunk and enabled receiving on port 9997 |
| 7 | `http://127.0.0.1:8000` unreachable from the host | Same root cause as #6: Splunk was stopped. One cause, two symptoms | Started the `Splunkd` service |
| 8 | Edge in the VM showed a Google search instead of Splunk | Typed `http: //` with a space, so it was treated as search text | Removed the space |
| 9 | No forwarder data: `index=_internal \| stats count by host` showed only `Splunk-Server` | Port 9997 was no longer listening. Confirmed on the server with `sudo ss -tlnp \| grep 9997`, which returned nothing | Re-enabled receiving on 9997 in Splunk Web. The exact reason it stopped wasn't confirmed; the server had pending updates and a restart required, so a Splunk restart is the likely trigger |
| 10 | `SplunkForwarder` wouldn't stop (Error 1053) | The forwarder was trying to flush queued data to an unreachable indexer before shutting down | Fixed the receiving port, then force-stopped the `splunkd` process and restarted the service |
| 11 | `Test-NetConnection` failed from the host | Ran it on my laptop (`SourceAddress : 192.168.1.235`), which can't reach the isolated lab network directly | Ran it inside the Windows VM (`SourceAddress : 10.0.2.4`) |
| 12 | Application, Security, and System arrived, but **Sysmon didn't** | The low-privilege `NT SERVICE\SplunkForwarder` account couldn't read the Sysmon channel | Added the account to the built-in **Event Log Readers** group instead of granting admin rights, then restarted the forwarder |
| 13 | `Add-LocalGroupMember` failed with "Principal ... was not found" | Typo in the account name and a line break inside the quoted string | Typed `NT SERVICE\SplunkForwarder` correctly on one line |
| 14 | `Add-LocalGroupMember` then failed with "Access denied" | PowerShell wasn't running as Administrator | Reran it from an elevated PowerShell |

**Fix for #12:**

```powershell
Add-LocalGroupMember -Group "Event Log Readers" -Member "NT SERVICE\SplunkForwarder"
Restart-Service SplunkForwarder
```

---

## Key Takeaways

- **Troubleshoot one layer at a time.** `DestinationHostUnreachable` (host down) → ping OK but TCP fails (service or port down) → TCP OK. Each result narrowed the problem.
- **Read how an error *changes*.** "Not found" turning into "Access denied" meant the account name was finally correct and only permissions were left.
- **Prefer least privilege.** The Sysmon permission issue was fixed by granting read access to event logs, not by running the forwarder as Local System.
- **The forwarder's own logs are evidence.** `index=_internal` showed whether the forwarder was connecting at all, which separated network problems from input problems.
- **Check *where* you're running a command.** The `SourceAddress` field in `Test-NetConnection` reveals which machine the test came from.
- **Event ID 1 (process creation)** is one of the most valuable events in endpoint detection: `ParentImage` + `Image` + `CommandLine` show what ran, how, and what launched it.

---

## Commands Learned

| Command | Purpose |
|---|---|
| `Get-Volume` | List drives (modern replacement for `wmic`) |
| `.\program.exe` | Run a program from the current folder in PowerShell |
| `Sysmon64.exe -accepteula -i <config>` | Install Sysmon with a configuration file |
| `Test-NetConnection -ComputerName <ip> -Port <port>` | Test whether a TCP port is reachable |
| `sudo ss -tlnp` | List listening TCP ports and the programs that own them (Linux) |
| `Stop-Process -Name splunkd -Force` | Force-stop a hung process |
| `Restart-Service SplunkForwarder` | Restart a Windows service |
| `Add-LocalGroupMember` / `Get-LocalGroupMember` | Manage and verify local group membership |

---

**Previous:** [Day 2: Static IP, SSH Access, and Splunk Installation](../day2/README.md)
**Next:** [Day 4: Add-ons and First Detection](../day4/README.md) *(coming soon)*
