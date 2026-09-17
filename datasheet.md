**Firmware:** `zephyr_email` v1.0.0  **Platform:** Tibbo TPP2W(G2) in TPB2L(G2) enclosure

> ## ⚠️ Configuration changes require a device restart
>
> All configuration — SMTP settings, time zone, device name, alarm rules, retry
> policies, contacts, alert recipients and email templates — is **validated and
> loaded when the device starts**. Values edited in the web console are saved
> to flash immediately, but the running alarm engine keeps using the
> configuration it loaded at boot.
>
> **After any configuration change, restart the device.** Until it restarts,
> the change is stored but not applied. This is repeated in every relevant
> section below.

---

## 1. Overview

The Email Alarm Notifier monitors up to 16 GPIO inputs and sends email alerts
when an input goes HIGH or LOW, according to its configured trigger condition.
Each alarm is tracked as an *episode*: reminder emails follow a configurable backoff
schedule up to the policy's retry limit, and an optional recovery email is
sent when the input returns to normal.

Configuration and monitoring are done through a built-in web console. All
state (rules, episodes, event log) is kept in on-board flash and survives
power loss.

## 2. Hardware

| Item | Detail |
|---|---|
| Controller | Tibbo TPP2W(G2) Tibbo Project PCB |
| Enclosure | TPB2L(G2), DIN-rail mounting kit, vibration-proof kit |
| Power | Tibbit #09 power supply module (socket 11), APR-P0008 adapter |
| Inputs | 4 × Tibbit #00-1 direct I/O modules (sockets 1, 3, 5, 7), 4 lines each = **16 inputs**, all configured as inputs |
| Network | 10/100 Ethernet |
| Storage | 1 MB AT45DB081E dataflash, LittleFS file system |
| Status LEDs | On-board red/green LEDs used for the setup-error indication |

Input lines are numbered **Input 1 … Input 16**, in socket order:

| Socket | Inputs |
|---|---|
| 1 | 1 – 4 |
| 3 | 5 – 8 |
| 5 | 9 – 12 |
| 7 | 13 – 16 |

## 3. Network services

| Service | Setting |
|---|---|
| IP addressing | DHCP (default) or static IP / netmask / gateway |
| DNS | 8.8.8.8, 5 s timeout |
| Time sync | SNTP against 216.239.35.0 (time.google.com), applied with the configured time zone |
| Web console | HTTP, port 80, 3 concurrent connections |
| Outgoing email | SMTP client, TLS (implicit), port 465 by default, LOGIN authentication, 120 s timeout |

The device needs outbound access to the SMTP server port and to the DNS and
SNTP servers.

![Settings page, Ethernet tab](https://raw.githubusercontent.com/tibbotech/alarm_notifier_assets/refs/heads/master/assets/alarm_notifier_settings_ethernet.png)

## 4. Web console

Title: **Device Console**. Two accounts: `admin` (full access) and `user`
(regular access). Pages refresh every 2 s.

### 4.1 Settings

| Group | Setting | Default | Notes |
|---|---|---|---|
| General | Device Name | `Device_Name_1` | Used in email content |
| General | Device Time Zone | GMT+8 | Read-only in the console; set in the project |
| General | Episode ID, Alarm ID, Log Id | 0 | Internal counters, editable for maintenance only |
| SMTP | SMTP Server | — | Host name or IP, max 80 characters |
| SMTP | SMTP Port | 465 | 1 – 65535 |
| SMTP | SMTP From Address | — | Max 50 characters |
| SMTP | SMTP Username | — | Max 50 characters |
| SMTP | SMTP Password | — | Max 50 characters, masked |

![Settings page, SMTP tab](https://raw.githubusercontent.com/tibbotech/alarm_notifier_assets/refs/heads/master/assets/alarm_notifier_settings_smtp.png)

**Restart the device after changing any setting.**

### 4.2 Configuration pages

| Page | Purpose |
|---|---|
| Alarm Rules | One row per input line: enable monitoring and set the alert direction (§5.1) |
| Retry Policies | Reusable alert retry / backoff profiles (§5.2) |
| Contacts | Email recipients (§5.3) |
| Alert Recipients | Which contacts receive alerts for which input (§5.4) |
| Email Templates | Per-input subject and custom text (§5.5) |
| Status | Live view of every input: name, current state, alarm state, alarm start, last update, and a Configuration tab to display setup errors, if any |

![Status page, Inputs tab, all inputs normal](https://raw.githubusercontent.com/tibbotech/alarm_notifier_assets/refs/heads/master/assets/alarm_notifier_status_ok.png)

**Restart the device after editing any of these pages.**

## 5. Configuration model

### 5.1 Alarm rules (max 20)

The **Alarm Rules** page lists the 16 input lines. Each line has:

| Column | Default | Meaning |
|---|---|---|
| Monitoring | off | Turn monitoring of this line on or off. Lines that are off are ignored |
| Name | `GPIO_<n>` | Shown in emails and on the Status page, max 50 characters |
| Alert on High | off | On: an alarm starts when the input goes **high**. Off: an alarm starts when the input goes **low** |
| Debounce | 5 s | The alarm condition must hold for this many seconds (0 – 50) before an alarm starts |
| Clear Delay | 10 s | The input must be back to normal for this many seconds before the alarm clears |
| Retry Policy | — | Retry policy ID that governs reminder emails (§5.2). Must be set |
| Send Recovery Email | on | Send a `[CLEARED]` email when the alarm clears |

Changes are saved with the page's Save button. Only lines with Monitoring
turned on are evaluated.

![Alarm Rules page](https://raw.githubusercontent.com/tibbotech/alarm_notifier_assets/refs/heads/master/assets/alarm_notifier_alarm_rules.png)

### 5.2 Retry policies (max 20)

| Column | Range | Meaning |
|---|---|---|
| Profile ID | 1 – 50 | Referenced by alarm rules as the Retry Policy |
| Base Delay (s) | ≥ 0 | Delay before the first reminder |
| Backoff Multiplier | ≥ 1 | Each reminder waits `base × multiplier^n`, where n is the reminder number |
| Max Delay (s) | ≥ 0 | Cap on the computed delay |
| Max Retries | -1 or ≥ 0 | Number of reminders while the alarm stays active. -1 = unlimited |

Reminder delay:

```
delay(n) = min(max_delay, base_delay × multiplier ^ n)
```

![Retry Policies page](https://raw.githubusercontent.com/tibbotech/alarm_notifier_assets/refs/heads/master/assets/alarm_notifier_retry_policies.png)

### 5.3 Contacts (max 1000)

| Field | Meaning |
|---|---|
| Contact ID | 1 – 50, referenced by alert recipients |
| Email address | Max 100 characters |

![Contacts page](https://raw.githubusercontent.com/tibbotech/alarm_notifier_assets/refs/heads/master/assets/alarm_notifier_contacts.png)

### 5.4 Alert recipients (max 100)

Links an **Input ID** to a **Contact ID**. Every input with an alarm rule
needs at least one recipient.

![Alert Recipients page](https://raw.githubusercontent.com/tibbotech/alarm_notifier_assets/refs/heads/master/assets/alarm_notifier_alert_recipients.png)

### 5.5 Email templates (max 100)

| Field | Meaning |
|---|---|
| Input ID | Input the template applies to |
| Email subject | Max 50 characters. The event timestamp is appended automatically |
| Custom Text | Max 50 characters, inserted in the email body |

Every input with an alarm rule needs a template.

![Email Templates page](https://raw.githubusercontent.com/tibbotech/alarm_notifier_assets/refs/heads/master/assets/alarm_notifier_email_templates.png)

**Restart the device after any change in §5.** The setup check at boot
verifies that every rule has a matching retry policy, template, recipient
and contact, and refuses to run otherwise (§8).

## 6. Data storage

All tables live on the internal dataflash under `/lfs` and persist across
power cycles.

| Table | Max records | Type | Content |
|---|---|---|---|
| ALARM_RULES | 20 | Config | Alarm rules |
| RETRY_POLICIES | 20 | Config | Retry policies |
| CONTACTS | 1000 | Config | Contacts |
| ALERT_RECIPIENTS | 100 | Config | Input → contact links |
| EMAIL_TEMPLATES | 100 | Config | Subjects and custom text |
| INPUT_ALARM_STATES | 30 | Runtime (RAM) | Per-input live state: current/previous state, alarm active, debounce and clear counters, retry index, next-alert countdown. Rebuilt from the rules at boot |
| ALARM_EPISODES | 20 | Runtime | One row per alarm occurrence: start/clear time, alert and recovery email status (PENDING / DELIVERED / ABANDONED), send attempts, next attempt time |
| EVENT_LOG | 1000 | Log (ring) | Timestamped event history (§9) |
| LOG | 200 | Log (ring) | System log lines |

## 7. Alarm lifecycle

1. **Input change.** Every input is sampled continuously. A change is written
   to the input's state record and logged as `INPUT_CHANGED`.
2. **Debounce.** Once per second the device evaluates each rule. The trigger
   condition must hold for *Debounce* consecutive seconds.
3. **Alarm start.** The alarm becomes active, an `ALARM_STARTED` event is
   logged, an episode record is created and the **alert email** is queued.
4. **Alert email.** Sent to every recipient of the input. Subject:
   `<template subject> - <YYYY-MM-DD hh:mm:ss>`. Body: device name, custom
   text, previous and current state, time of occurrence.
5. **Reminders.** While the alarm stays active, a reminder is sent after
   each computed delay, with subject prefix `[REMINDER n of N]` and the
   elapsed duration in the body, until *Max Retries* is reached. An alert
   that exhausts its retries is marked ABANDONED.
6. **Clear.** When the input has been back in range for *Clear delay*
   seconds the alarm clears, the episode is closed with the clear time, an
   `ALARM_CLEARED` event is logged and, if *Send recovery email* is TRUE, a
   `[CLEARED]` email is sent with the occurrence time, clear time and total
   duration.

An active alarm on the Status page:

![Status page showing an input with an active alarm](https://raw.githubusercontent.com/tibbotech/alarm_notifier_assets/refs/heads/master/assets/alarm_notifier_status_error.png)

Example alert email:

![Alert email as received in a mail client](https://raw.githubusercontent.com/tibbotech/alarm_notifier_assets/refs/heads/master/assets/alarm_notifier_email.png)

Email delivery is only attempted while the network is up. Episodes with
pending emails are re-scanned every 10 s and after every reconnect, so a
mail interrupted by a network outage is sent once connectivity returns. Each
send is logged as `EMAIL_SENT` or `EMAIL_SEND_FAILED`.

## 8. Startup validation and setup-error indication

At every boot the device:

1. Records `SYSTEM_STARTED` in the event log and synchronises time.
2. Checks that SMTP server, from address, username and password are set.
3. Checks that at least one alarm rule exists.
4. For each rule, checks that its retry policy, email template, alert
   recipient and contact exist.
5. Builds the live input-state table from the rules.

If any check fails the device enters **setup error**:

- The status LEDs flash in a repeating pattern (both LEDs on, then off).
- The message is shown in the dashboard *Setup error* panel and logged as a
  `SYTEM_ERROR` event, e.g. `SMTP client not properly configured. Update and
  restart device.`, `No alarm rule configured in project.`, `Missing retry
  policy in setup.`, `Missing email template in setup.`, `Missing alert email
  recipient in setup.`, `Missing contact information in setup.`
- Alarm monitoring does not start.

![Status page, Configuration tab showing a setup error](https://raw.githubusercontent.com/tibbotech/alarm_notifier_assets/refs/heads/master/assets/alarm_notifier_setup_error.png)

**The setup-error state is cleared only by restarting a correctly configured
device.** Fix the configuration in the web console, then restart.

After a successful restart the Configuration tab reports no error:

![Status page, Configuration tab showing no error](https://raw.githubusercontent.com/tibbotech/alarm_notifier_assets/refs/heads/master/assets/alarm_notifier_setup_ok.png)

The same messages can appear during operation if a rule, policy, template,
recipient or contact that was present at boot is later deleted. Restore the
missing item and restart.

## 9. Event log

`EVENT_LOG` keeps the last 1000 events with timestamp, type, source, reference
and message:

| Type | Raised when |
|---|---|
| SYSTEM_STARTED | Device booted |
| SYTEM_ERROR | Setup validation failed (message included) |
| NETWORK_CONNECTED / NETWORK_DISCONNECTED | Ethernet link state changed |
| INPUT_CHANGED | Any of the 16 inputs changed state |
| ALARM_STARTED / ALARM_CLEARED | An alarm episode opened / closed |
| EMAIL_SENT / EMAIL_SEND_FAILED | Result of each SMTP delivery attempt |

![Log page, EVENT_LOG tab](https://raw.githubusercontent.com/tibbotech/alarm_notifier_assets/refs/heads/master/assets/alarm_notifier_logs.png)

## 10. Commissioning checklist

1. Mount the unit, wire the inputs and power it.
2. Connect Ethernet. Open `http://<device IP>/` and log in as `admin`.
3. Fill in the **SMTP** settings.
4. Create **Retry Policies**, **Contacts**, **Email Templates**, **Alert
   Recipients** and finally **Alarm Rules** for every input to monitor.
5. **Restart the device.**
6. Confirm the LEDs are not flashing the setup-error pattern and the
   Configuration tab in the Status page shows no setup error.
7. Trigger a test input and confirm the alert email arrives, then release it
   and confirm the recovery email.

Repeat step 5 after **every** later change to settings or configuration
tables. Changes are stored immediately but only take effect after a restart.
