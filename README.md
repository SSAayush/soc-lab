# Home SOC Lab: Splunk SIEM Detection Lab

A home Security Operations Center (SOC) lab built to practice log collection, threat detection, and alert triage using Splunk. The lab simulates a small corporate environment: a Windows 11 endpoint sends security logs to a Splunk server, where I write detections for common attacker techniques and map them to the MITRE ATT&CK framework.

**Author:** Aayush Bahadur Koirala
**Status:** 🚧 In progress

---

## Objectives

- Build an isolated virtual lab with a SIEM server and a Windows endpoint
- Collect Windows Event Logs and Sysmon telemetry in Splunk
- Simulate common attack activity in a safe, contained environment
- Write Splunk searches and alerts that detect that activity
- Map each detection to a MITRE ATT&CK technique
- Document the build, troubleshooting, and findings like a real SOC analyst

---

## Lab Architecture

```
            ┌──────────────────────────────────────────┐
            │  Host: Acer Nitro 5 (16 GB RAM)          │
            │  Hypervisor: Oracle VirtualBox 7.x       │
            │                                          │
            │   NAT Network: SOC-Lab (10.0.2.0/24)     │
            │                                          │
            │   ┌───────────────┐   logs   ┌────────┐  │
            │   │ Win11-Target  │ ───────► │ Splunk │  │
            │   │ (Endpoint)    │          │ Server │  │
            │   └───────────────┘          └────────┘  │
            └──────────────────────────────────────────┘
```

| VM | OS | RAM | vCPU | Disk | Role |
|---|---|---|---|---|---|
| Splunk-Server | Ubuntu Server 26.04 LTS | 4 GB | 2 | 50 GB | SIEM server (Splunk Enterprise) |
| Win11-Target | Windows 11 Enterprise Evaluation | 4 GB | 2 | 60 GB | Monitored endpoint |

**Network:** Both VMs are attached to a VirtualBox NAT Network named `SOC-Lab` (10.0.2.0/24, DHCP enabled). This keeps the lab isolated from my home network while still allowing internet access for updates.

---

## Tools & Technologies

- **Oracle VirtualBox:** virtualization
- **Ubuntu Server 26.04 LTS:** Splunk host
- **Windows 11 Enterprise Evaluation:** endpoint
- **Splunk Enterprise (Free):** SIEM *(planned)*
- **Sysmon:** detailed Windows endpoint telemetry *(planned)*
- **Splunk Universal Forwarder:** log shipping *(planned)*
- **MITRE ATT&CK:** detection mapping *(planned)*

---

## Progress

- [x] **Day 1:** Build the lab environment (network + both VMs)
- [ ] **Day 2:** Install Splunk and set a static IP on the server
- [ ] **Day 3:** Install Sysmon and the Universal Forwarder on Windows; confirm logs arrive in Splunk
- [ ] **Day 4:** Simulate attack activity and write detection searches
- [ ] **Day 5:** Turn searches into alerts and build a dashboard
- [ ] **Day 6:** Map detections to MITRE ATT&CK and finalize documentation

---

## Day 1: Building the Lab Environment

### What I did

1. **Created an isolated lab network.** Built a VirtualBox NAT Network called `SOC-Lab` (10.0.2.0/24) with DHCP enabled.
2. **Built the Splunk server VM.** Installed Ubuntu Server 26.04 LTS with OpenSSH, ran system updates, and attached it to `SOC-Lab`.
3. **Built the Windows endpoint VM.** Installed Windows 11 Enterprise Evaluation with a local account and privacy telemetry turned off (to reduce background noise in future logs). Attached it to `SOC-Lab`.
4. **Verified connectivity.** Confirmed with `ipconfig` / `ip a` that both VMs received addresses on 10.0.2.0/24, then successfully pinged the Splunk server from the Windows VM.

### Screenshots

**VirtualBox Manager with both VMs**
![VirtualBox Manager with both VMs](screenshots/day1/virtualbox-vms.png)

**SOC-Lab NAT Network settings**
![SOC-Lab NAT Network settings](screenshots/day1/soc-lab-network.png)

**Ubuntu server `ip a` output**
![Ubuntu server ip a output](screenshots/day1/ubuntu-ip.png)

**Windows `ipconfig` output**
![Windows ipconfig output](screenshots/day1/windows-ipconfig.png)

**Successful ping from Windows to Ubuntu**
![Successful ping from Windows to Ubuntu](screenshots/day1/ping-success.png)

### Troubleshooting Log

Real problems I ran into while building the lab, and how I solved them.

| # | Problem | Root Cause | Fix |
|---|---|---|---|
| 1 | Guest Additions installer returned `command not found` on Ubuntu Desktop | The Guest Additions ISO was inserted but not mounted, so the file path didn't exist | Mounted the virtual CD (`sudo mount /dev/cdrom /mnt/cdrom`) and ran the installer from the mount point |
| 2 | Ubuntu installed and booted without asking for any settings | VirtualBox's **Unattended Installation** was enabled by default | Rebuilt the server, then verified/added OpenSSH, updated packages, and checked disk allocation. Lesson: check **"Skip Unattended Installation"** for manual control |
| 3 | VM was not on the lab network after install | Network adapter was left on the default setting before first boot | Changed Adapter 1 to **NAT Network → SOC-Lab** and re-applied networking with `sudo netplan apply` |
| 4 | Windows 11 VM showed a black screen at boot and the installer never appeared | Display/firmware incompatibility during EFI boot on this host (graphics controller and/or virtual Secure Boot) | Set graphics controller to **VMSVGA**, disabled 3D acceleration, and disabled Secure Boot (EFI and TPM 2.0 kept enabled) |
| 5 | Black screen during Windows setup restarts | Display not redrawing between install stages | A VM reset (without pressing keys, so it booted from disk) resumed setup normally |
| 6 | `ping` returned "The syntax of the command is incorrect" | Typed the `< >` placeholder brackets, which Command Prompt treats as redirection operators | Re-ran the command with the plain IP address |

### Key Takeaways

- Always set a VM's network adapter **before** the first boot.
- Disable unattended installation when specific install options matter (SSH, disk layout).
- When a VM shows a black screen, check the status bar for disk/CPU activity to tell "not booting" apart from "not drawing."
- Isolate one variable at a time when troubleshooting (I changed three settings at once on issue #4, so I can't say for certain which one fixed it).
- Take VM snapshots at known-good checkpoints so you can roll back instead of rebuilding.

---

## Day 2: Splunk Installation

*Coming soon.*

---

## Repository Structure

```
soc-lab/
├── README.md
├── screenshots/
│   ├── day1/
│   ├── day2/
│   └── ...
└── detections/          # Splunk searches (SPL) for each detection
```

---

## Disclaimer

All attack simulations in this lab are performed only on virtual machines I own, inside an isolated network, for educational purposes.
