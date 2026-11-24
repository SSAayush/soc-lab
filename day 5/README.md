# Day 5: Setting Up and Debugging the Brute-Force Alert

Part of my [Home SOC Lab](../README.md) project. On Day 4 I wrote a brute-force detection and saved it as an alert. On Day 5 I found out it wasn't actually firing, traced the problem through four separate causes, fixed each one, and got the alert to trigger on a live simulated attack.

**Result:** a scheduled Splunk alert that fires when one account has 5+ failed logons within 10 minutes.
**MITRE ATT&CK:** [T1110: Brute Force](https://attack.mitre.org/techniques/T1110/)

**Full detection loop:**

```
Wrong passwords on Windows → Event ID 4625 → Universal Forwarder → Splunk
→ scheduled search every 5 min → Triggered Alert
```

---

## Final Alert Configuration

| Setting | Value | Reason |
|---|---|---|
| Title | Brute Force - 5+ Failed Logons in 5 Min | Describes what fired |
| Search | see below | Counts failed logons per account in the window |
| Alert type | Scheduled | Runs automatically |
| Schedule | Cron: `*/5 * * * *` | Every 5 minutes |
| Time range | Relative: 10 minutes ago → now | Overlaps runs so slow-arriving logs aren't missed |
| Trigger condition | Number of results > 0 | Any row means an account crossed the threshold |
| Trigger | Once | One alert per run |
| Trigger action | Add to Triggered Alerts | Records it under Activity → Triggered Alerts |
| Expires | 7 days | Keeps triggered alerts visible long enough to review |

**Alert search:**

```spl
index=endpoint EventCode=4625
| stats count by user
| where count >= 5
```

This differs from the Day 4 hunting search, which used `bin _time span=5m`. See troubleshooting issue #4 for why.

---

## Testing

1. Locked the Windows VM and entered the wrong password 7 times in quick succession, then logged in normally
2. Waited for the next scheduled runs
3. Confirmed the alert appeared in **Activity → Triggered Alerts**, with a result row for the targeted account
4. Confirmed the **Alerts** count on the Searches, Reports, and Alerts page increased

---

## Troubleshooting Log

The alert was saved and showed as **Enabled**, but **Triggered Alerts** stayed empty. I worked through the problem one layer at a time.

| # | Symptom | How I found it | Root cause | Fix |
|---|---|---|---|---|
| 1 | Cron set to `*/6 * * * *` | Reviewed the schedule against the time range | Running every 6 minutes while looking back 5 minutes leaves a 1-minute blind spot between runs | Changed to `*/5 * * * *` so the schedule matches the window |
| 2 | Alert enabled but never ran | **Next Scheduled Time** on the Searches, Reports, and Alerts page showed a date a week away | The schedule had been saved as a weekly run, not the intended cron | Used **Edit → Advanced edit** to check and set `cron_schedule` to `*/5 * * * *` directly. Next Scheduled Time then showed the next 5-minute mark |
| 3 | Alert ran but always returned 0 results | The `search` field in Advanced edit ended at `\| bin span=5m _time` | The saved search had been cut off, leaving out `stats` and `where` | Replaced it with the complete search |
| 4 | Design risk: attacks could be split | Reviewed how `bin` groups events | `bin` cuts time at fixed clock boundaries (:40, :45, :50...). Attempts from 5:43–5:46 would be counted as two partial buckets, so neither might reach 5: a **false negative** | Removed `bin` from the alert and counted all failures in the 10-minute window instead |
| 5 | Still no triggered alert after a new test | Scheduler logs showed every run succeeded with `result_count=0`; the newest event in `index=endpoint` was hours old; the timeline showed one bar | The **SplunkForwarder** service on the Windows VM wasn't running, so no new logs were arriving | Started the service, confirmed new events, and made it more resilient (see below) |
| 6 | Event times didn't match my clock | Splunk showed 21:15 for an attempt made at 3:15 PM | Splunk was displaying times in **UTC** (6 hours ahead of Central) | Noted the offset; the user time zone can be changed under **Administrator → Preferences** |

**Checking whether the alert actually ran:**

```spl
index=_internal sourcetype=scheduler savedsearch_name="Brute Force*"
| table _time status result_count alert_actions
```

This showed the alert was healthy (running every 5 minutes, `status=success`) and that the problem was upstream: it simply had no matching events to find.

**Confirming the pipeline end to end:**

| Layer | Check | Result |
|---|---|---|
| Splunk server listening | `sudo ss -tlnp \| grep 9997` | Listening ✅ |
| Forwarder service | Services → SplunkForwarder | **Stopped** ❌ (the break) |
| Data arriving | `index=endpoint` over the last 15 minutes | Events resumed after starting the service ✅ |

**Hardening the forwarder service** (Services → SplunkForwarder → Properties):

- **Startup type:** Automatic (Delayed Start), so it starts after the network is fully up
- **Recovery:** Restart the Service on first, second, and subsequent failures

---

## Reading Cron Expressions

Cron has five fields: *minute, hour, day of month, month, day of week*.

| Expression | Meaning |
|---|---|
| `*/5 * * * *` | Every 5 minutes |
| `*/6 * * * *` | Every 6 minutes |
| `0 5 * * 1` | 05:00 every Monday |
| `0 * * * *` | Every hour, on the hour |

---

## Key Takeaways

- **"Enabled" doesn't mean "working."** Check Next Scheduled Time and the scheduler logs in `index=_internal` to confirm an alert really runs.
- **Read the stored config, not just the form.** Advanced edit showed the real `cron_schedule` and `search` values, which revealed two of the problems.
- **The schedule, time range, and detection window should line up.** A mismatch leaves gaps where activity is never checked.
- **Fixed time buckets can split an attack.** That's fine for hunting, but for alerting, a count over the whole window avoids false negatives.
- **A detection is only as good as its data pipeline.** The alert logic was correct; it failed because the forwarder had stopped. A useful next detection would alert when a host stops sending logs.
- **Know your time zone.** Correlating events is much easier when you know whether timestamps are in UTC or local time.
