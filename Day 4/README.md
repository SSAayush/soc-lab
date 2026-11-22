# Day 4: Splunk Add-ons and First Detection (Brute Force)

Part of my [Home SOC Lab](../README.md) project. Day 4 makes the raw logs searchable by field, then puts them to work: I simulated a brute-force attack against the Windows endpoint, analyzed the resulting events, wrote a detection in SPL, and scheduled it as a Splunk alert.

**Detection built:** Brute-force login attempts
**MITRE ATT&CK:** [T1110: Brute Force](https://attack.mitre.org/techniques/T1110/)

---

## What I did

### 1. Installed Splunk add-ons for field extraction

Out of the box, Splunk treated Sysmon events as one large XML blob. Add-ons (Technology Add-ons, or TAs) teach Splunk how to parse a data source into named fields.

- **Splunk Add-on for Sysmon**
- **Splunk Add-on for Microsoft Windows**

Both were downloaded from Splunkbase as `.spl` packages and installed through **Apps → Manage Apps → Install app from file**, which uploads them to the Splunk server. After installation, fields like `EventCode`, `Image`, `ParentImage`, `CommandLine`, and `user` became searchable.

### 2. Found a specific process creation event

Narrowed thousands of events down to a single Notepad launch by layering filters:

```spl
index=endpoint EventCode=1 Image=*notepad.exe
```

| Part | Meaning |
|---|---|
| `index=endpoint` | Where the data is stored |
| `EventCode=1` | Sysmon process creation |
| `Image=*notepad.exe` | The program's path ends in `notepad.exe` (`*` is a wildcard) |

Then examined the event's `ParentImage` to see which process launched it.

### 3. Simulated a brute-force attack

On the Windows VM, locked the screen and entered the wrong password for my account several times in a row, then logged in normally.

### 4. Identified the right Event ID

Instead of scrolling through events one by one, filtered the Security log for failures and let Splunk summarize them:

```spl
index=endpoint source="WinEventLog:Security" Keywords="Audit Failure"
| stats count by EventCode
```

This surfaced **Event ID 4625: An account failed to log on.** (Its counterpart, 4624, is a successful logon.)

### 5. Analyzed a 4625 event

| Field | Value | Meaning |
|---|---|---|
| EventCode | 4625 | Failed logon |
| Account Name (failed) | `aayush` | The targeted account |
| Logon Type | 2 | Interactive (keyboard or lock screen) |
| Failure Reason | Unknown user name or bad password | General reason |
| Sub Status | `0xC000006A` | Precise reason: **account exists, wrong password** |
| Source Network Address | 127.0.0.1 | Attempt originated on the machine itself |

**Why the details matter:**

- **Sub Status** is more specific than the Failure Reason text. `0xC000006A` means the account exists but the password is wrong, while `0xC0000064` means the account doesn't exist. Many `0xC0000064` failures across different names suggests username spraying; many `0xC000006A` failures on one account suggests password guessing against a real user.
- **Logon Type** shows how the attempt was made: `2` interactive, `3` network, `10` Remote Desktop. In a real attack, type 3 or 10 from an external address is far more concerning than type 2 from `127.0.0.1`.
- The **Subject** account was the computer itself (`DESKTOP-2FCA0CN$`), which is expected: the logon process checks the password on the user's behalf.

### 6. Summarized failed logons by account

```spl
index=endpoint EventCode=4625
| stats count by user
```

### 7. Wrote the detection

One failed login is normal. Many failures for the same account in a short time is suspicious. The detection needs both a **threshold** and a **time window**:

> **5 or more failed logons for the same account within 5 minutes**

```spl
index=endpoint EventCode=4625
| bin _time span=5m
| stats count by _time, user
| where count >= 5
```

| Step | What it does |
|---|---|
| `index=endpoint EventCode=4625` | Collect failed logons |
| `bin _time span=5m` | Round each event's timestamp into a 5-minute bucket |
| `stats count by _time, user` | Count failures per bucket, per account |
| `where count >= 5` | Keep only buckets that crossed the threshold |

The threshold of 5 is in line with common account lockout policies.

### 8. Scheduled it as an alert

Saved the search as a Splunk alert so it runs automatically:

| Setting | Value | Reason |
|---|---|---|
| Title | Brute Force - 5+ Failed Logons in 5 Min | Describes exactly what fired |
| Schedule | Cron: `*/5 * * * *` | Runs every 5 minutes, matching the detection window |
| Time range | Last 5 minutes | Each run checks only the newest window |
| Trigger condition | Number of results > 0 | Any row means an account crossed the threshold |
| Trigger action | Add to Triggered Alerts | Shows up in Splunk's alert list for review |

**Why every 5 minutes instead of hourly:** an hourly schedule could leave an attacker up to 59 minutes of unnoticed guessing. Running more often detects faster, at the cost of more server load, which is a tradeoff real SOCs balance across many alerts.

**Cron syntax:** the five fields are *minute, hour, day of month, month, day of week*. `*/5` in the minute field means "every 5 minutes," and `*` means "every" for the rest.

---

## SPL Lessons Learned

Mistakes I made while learning SPL, and what each one taught me:

| Mistake | Lesson |
|---|---|
| `index=notepad` | `index` is *where* data is stored (`endpoint`). What I'm looking for goes in a **field** (`Image=*notepad.exe`) |
| Looked for an "Image" file to open | `Image` is a field name: Windows' term for the path of the program that ran |
| `index=enpoint` | A typo in the index name returns zero results with **no error**, so double-check spelling first when a search comes back empty |
| `EventCode= 4625` | No spaces around `=` in `field=value` |
| `stats count by 4625` | `by` takes a **field name** to group by. Values go **before** the pipe as filters |
| `stats count by user bin` | `bin` is its own command and needs its own pipe step, **before** `stats` |
| Expected `bin` to change the results | `bin` only rounds timestamps behind the scenes; its effect shows once `stats` groups by `_time` |

**Mental model:** pipes form an assembly line. Each command does one job and passes its output to the next:

```
[filter which events] | [transform] | [summarize] | [filter the summary]
```

---

## Key Takeaways

- **Add-ons turn raw logs into usable data.** Without field extraction, detections on fields like `ParentImage` or `user` aren't possible.
- **Let the tool summarize.** `stats` turns thousands of events into a short table, so nobody has to scroll event by event.
- **Look up Event IDs.** Knowing where to find them (and recognizing key ones like 1, 4624, and 4625) is daily SOC work.
- **Details separate attacks from noise.** Sub Status codes and Logon Types change how serious a failed logon is.
- **A detection is a threshold plus a time window,** and the alert schedule should match the window.

---

## Next Steps

- Confirm the alert fires by repeating the brute-force simulation and checking **Activity → Triggered Alerts**
- Simulate and detect more techniques: new local admin account creation, suspicious PowerShell, and registry Run key persistence

---

**Previous:** [Day 3: Sysmon, Universal Forwarder, and Log Ingestion](../day3/README.md)
