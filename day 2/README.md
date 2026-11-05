# Day 2: Static IP, SSH Access, and Splunk Installation

Part of my [Home SOC Lab](../README.md) project. Day 2 turns the Ubuntu VM into a working Splunk server: a fixed IP address, remote SSH access from the host, and Splunk Enterprise installed and running as a service.

---

## What I did

### 1. Assigned a static IP to the Splunk server

Disabled cloud-init's network management and configured netplan so the server always uses `10.0.2.100`. A SIEM needs a fixed address because every forwarder is configured to send logs to it.

```yaml
# /etc/netplan/00-installer-configure.yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [10.0.2.100/24]
      routes:
        - to: default
          via: 10.0.2.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

```bash
echo "network: {config: disabled}" | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
sudo chmod 600 /etc/netplan/00-installer-configure.yaml
sudo netplan apply
```

### 2. Set up SSH access from the host

Installed and enabled OpenSSH, then added a VirtualBox NAT Network port forwarding rule so I could administer the server from my laptop's terminal with copy-paste, instead of typing into the VM console.

```bash
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
# from the host:
ssh -p 2222 vboxuser@127.0.0.1
```

On first connection I compared the ED25519 host key fingerprint with the one the server generated at first boot, confirming I was connecting to the right machine.

### 3. Installed Splunk Enterprise

Downloaded the `.deb` package with `wget`, installed it to `/opt/splunk`, and did the first start as the dedicated `splunk` user (not root) to create the admin account.

```bash
sudo dpkg -i splunk*.deb
sudo -u splunk /opt/splunk/bin/splunk start --accept-license
```

### 4. Configured Splunk to start on boot

Set Splunk up as a systemd-managed service running under the `splunk` user.

```bash
sudo -u splunk /opt/splunk/bin/splunk stop
sudo /opt/splunk/bin/splunk enable boot-start -user splunk -systemd-managed 1 --run-as-root
sudo systemctl daemon-reload
sudo systemctl start Splunkd
```

### 5. Accessed Splunk Web from the host

Added a second port forwarding rule and logged in at `http://127.0.0.1:8000`.

| Name | Protocol | Host IP | Host Port | Guest IP | Guest Port | Purpose |
|---|---|---|---|---|---|---|
| ssh | TCP | 127.0.0.1 | 2222 | 10.0.2.100 | 22 | SSH administration |
| splunk-web | TCP | 127.0.0.1 | 8000 | 10.0.2.100 | 8000 | Splunk Web |

Both rules bind only to `127.0.0.1`, so the services are reachable from my laptop but not exposed to the rest of my home network.

### 6. Prepared Splunk to receive logs

- Enabled a receiving port on **9997** (Settings → Forwarding and receiving) for the Universal Forwarder
- Created an **`endpoint`** index to hold the Windows endpoint data (Settings → Indexes)

---

## Screenshots

**Static IP applied (`10.0.2.100`, no longer "dynamic")**
![Static IP applied](screenshots/static-ip.png)

**Internet connectivity confirmed after the change**
![Ping after static IP](screenshots/ping-static.png)

**NAT Network port forwarding rules**
![Port forwarding rules](screenshots/port-forwarding.png)

**SSH login from the host**
![SSH login from host](screenshots/ssh-login.png)

**Splunk first start completed**
![Splunk first start](screenshots/splunk-first-start.png)

**Splunkd running as a systemd service**
![Splunkd service status](screenshots/splunkd-service.png)

**Splunk Web home page**
![Splunk Web home page](screenshots/splunk-home.png)

**Receiving enabled on port 9997**
![Receiving port 9997](screenshots/receiving-9997.png)

**`endpoint` index created**
![endpoint index](screenshots/index-endpoint.png)

---

## Troubleshooting Log

| # | Problem | Root Cause | Fix |
|---|---|---|---|
| 1 | Static IP didn't apply; `ip a` still showed a `dynamic` DHCP address | Three files in `/etc/netplan/` (the original DHCP config, a misspelled `.ymal` copy, and my edited file). Netplan merges every `.yaml` file and ignores other extensions, so my changes were overridden or ignored | Consolidated to a **single** `.yaml` file, then applied and rebooted |
| 2 | Static IP config still wasn't picked up | Typo in the config: `dhcps4` instead of `dhcp4` | Corrected the key name |
| 3 | Netplan warned that permissions were "too open" | Config file was readable by all users | `sudo chmod 600` on the netplan file |
| 4 | SSH failed with `kex_exchange_identification: read: Connection aborted` | The port forward reached the VM, but `openssh-server` was never installed (side effect of VirtualBox's unattended install) | Installed `openssh-server` and enabled it with `systemctl enable --now ssh` |
| 5 | `sudo splunk stop` didn't stop Splunk | Newer Splunk versions refuse to run commands as root without `--run-as-root` | Stopped Splunk as the service account: `sudo -u splunk /opt/splunk/bin/splunk stop` |
| 6 | `enable boot-start` failed: "splunk is currently running" | Knock-on effect of #5 | Ran it again after Splunk had actually stopped |
| 7 | `Unit Splunkd.service not found` | Boot-start had never succeeded, so no systemd unit existed | Fixed by #5 and #6; the unit was then created at `/etc/systemd/system/Splunkd.service` |
| 8 | Kernel `watchdog: BUG: soft lockup` messages on the server console | The host laptop was under heavy load, briefly starving the VM of CPU | Closed unnecessary apps on the host while running the lab |

---

## Key Takeaways

- A SIEM server needs a **static IP**, since forwarders are configured to send to a fixed address.
- `ls` the config directory before editing; stray or duplicate files are a common cause of "my change didn't work."
- Run services under a **dedicated, non-root service account**. Splunk now enforces this.
- Read error messages closely: "Connection aborted" (reached the VM, nothing listening) is a different problem from "Connection refused" or a timeout.
- Bind port forwards to `127.0.0.1` so lab services aren't exposed to the wider network.

---

**Next:** [Day 3: Sysmon and the Universal Forwarder](../day3/README.md) *(coming soon)*
