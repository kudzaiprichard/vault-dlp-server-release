# Vault admin guide

How to run data loss prevention day to day from the dashboard.

## Roles

| Role         | Can                                                                                      |
| ------------ | ---------------------------------------------------------------------------------------- |
| **Admin**    | Everything: policies, data types, devices, users, registration keys, settings, audit log |
| **Analyst**  | View everything on the dashboard; triage incidents (status, notes, assignment)           |
| **Viewer**   | Read-only: overview, incidents, devices                                                  |
| **Employee** | No dashboard access. Exists so policies can target people and incidents can name them    |

Five wrong passwords lock an account for 15 minutes. An admin can unlock it under _Users_. Every
sign-in and change is in the _Audit log_.

## How detection is organised

- A **data type** is a kind of sensitive data, e.g. _Payment card number_ (`card_number`). Its
  **weight** feeds the risk score: a request's risk is the sum of weight × confidence over its
  findings.
- A data type is found by one or more **rules**:
  - **Regex rules:** a pattern, with an optional validator (card/Luhn/IBAN/IP), context words and
    allowlists. Patterns use RE2 syntax, as on the agent; the editor refuses lookarounds and
    backreferences.
  - **Keyword rules:** words or phrases, whole-word and case options.
  - **ML labels:** plain words such as _medical condition_. The GLiNER2 model on each PC finds them
    with no retraining.
- Built-in rules (from the agent's rule packs and the Gitleaks set) can be switched off or have
  their confidence tuned. Custom rules can be fully edited.
- The **test box** (_Data types → Test rules_, or on a data type's page) shows which regex and
  keyword rules match sample text. ML labels can only be tested on an agent.

## Policies

A policy says: when a request contains **at least N findings** of **these data types** (above a
**minimum confidence**, optionally above a **risk score**, optionally only to **these
destinations**), for **these people or devices**, do **this action**.

- Actions, mildest to strictest: **allow, log, alert, warn, block**. When several policies match,
  the strictest wins.
- **Monitor mode** only logs. Every policy you create starts there, as do the fuzzier defaults
  (ZW IDs, bank details, PII to webmail, confidential markings). The three precise defaults
  (private keys, card numbers, secrets) enforce from the start. Watch a monitored policy's
  incidents for a week, mark
  false positives, tune the thresholds, then switch it to **enforce**.
- **Act on rule hits immediately** (hard block) skips waiting for the ML model, for things like
  private keys and valid card numbers.
- **Scope:** leave users, departments and devices empty to apply to everyone. Departments are
  expanded to their members' Windows usernames.
- **Where it applies:** any destination, chosen AI services ([below](#ai-services)), chosen
  destination hosts, or AI services plus hosts.
- **Leaving through:** web, email clients, clipboard, USB storage and printing (all by default).
  Destinations and AI services only narrow web and email. Older agents only check the web.
  [Endpoint controls](#endpoint-controls) explains what the agent can do on each channel.
- **Allowed destinations** (company systems) are never warned about or blocked; matches there
  are only logged.

Every change publishes a new policy version, and agents pick it up on their next poll. A change
that would produce a bundle the agent can't load is refused and nothing is saved.

## AI services

_Detection → AI services_ lists the AI tools agents recognise: about 45 chat assistants (ChatGPT,
Gemini, Claude, Copilot, DeepSeek, Grok, Perplexity, Mistral, Meta AI, Qwen, Kimi and more),
coding tools (GitHub Copilot, Cursor, Windsurf, Codex, Claude Code and more) and model APIs. Each
one has:

- **Hosts:** the sites its web app, desktop app and command-line tools send to.
- **Apps:** its Windows executables, e.g. `claude.exe` or `cursor.exe`. Agents recognise these
  even when they call a host the list doesn't have yet.
- **Hosts and paths to skip:** assets and telemetry that aren't AI use.

Use them in policies:

- In the policy editor, under **Where it applies**, turn on **Any AI service**, or choose
  services. The policy then covers those tools in Chrome, Edge and their desktop and command-line
  apps. Policies that apply to _any destination_ already cover AI tools too.
- The default **Alert on sensitive data sent to AI services** policy (monitor mode) alerts on
  personal, financial, identity and secret data sent to any AI tool. The block policies for card
  numbers, secrets and private keys apply to AI tools like everywhere else.

See what the agents can actually read. Each service shows its **coverage** over the last 7 days:

| Coverage             | Meaning                                                                            |
| -------------------- | ---------------------------------------------------------------------------------- |
| _Visible_            | Everything sent was read and scanned                                               |
| _Partly visible_     | Some requests couldn't be read: open the service to see which device, app and host |
| _Blind spot_         | Requests were seen but none could be read, so data sent there isn't checked        |
| _Not seen_           | Not used in the period                                                             |
| _No coverage report_ | Incidents arrived, but the agents are too old to report coverage                   |

- **Unknown hosts the agents couldn't read** lists other sites where agents saw traffic they
  couldn't read, or apps that refused the agent's certificate. If one is an AI tool, click **Add
  as AI service**.
- **When content can't be read** decides what agents do when an AI tool sends data in a format
  they can't read yet: allow, log (the default), warn or block. Start with _Log_ and watch the
  coverage table before blocking.
- The incident list has an **AI service** filter, and each incident shows the AI service and app.
- Built-in services can be edited or disabled but not deleted. Upgrades add new built-in services
  and keep your changes.

**Command-line tools** such as Claude Code and Codex only send through the agent when the agent
sets their proxy and certificate variables (`HTTPS_PROXY`, `NODE_EXTRA_CA_CERTS`, ...). That's an
agent setting, off by default. Until it's on, those tools show as _Not seen_.

## Exact data

_Detection → Exact data_ protects your **real identifiers**: customer account numbers, staff
numbers, national IDs of your clients, email addresses. A pattern rule catches anything with the
right shape; an exact data list catches only the values on your list, so there are far fewer
false positives.

1. **New list:** name it and say what the values look like (numbers, codes such as
   `CL-2024-00871`, Zimbabwe ID numbers, email addresses, or your own pattern). A data type named
   after the list is created for you.
2. **Upload** a CSV (choose the column) or a text file with one value per line, up to 200 MB.
   Upload again to replace the list.
3. **Test** sample text to see which listed values it contains.
4. Add the list's data type to a policy, e.g. _Block customer account numbers_.

How the values are protected: the server turns each value into a one-way hash with a secret salt
while uploading, and keeps only the hashes; the values themselves are never stored or logged.
Agents get the salt and hashes and compare hashes of what they see. Hashes can't be turned back
into the list, but short numbers could be guessed by trying every possibility, so number lists
need at least 8 digits. All enabled lists together can hold 10 million values.

## Endpoint controls

_Detection → Endpoint controls_ covers what happens on the PC itself, whatever the content. The
agent runs without administrator rights or drivers, so some controls act **after** the event.
Each control shows how many PCs can apply it.

| Control               | Choices                              | What the agent really does                                                                                                                                  |
| --------------------- | ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| USB storage           | Allow, Monitor, Remove after copying | Scans files just after they're written to a USB drive; reports them, or deletes files that break a policy. It can't stop the copy itself                    |
| Printing              | Allow, Monitor, Block                | Sees the job (document name, pages, printer), not the printed content; Block cancels the job                                                                |
| Copying into AI tools | Inspect, Block                       | Checks copied text while an AI tool is in front; Block clears the clipboard (best effort). Pasting into AI websites is also caught when the message is sent |
| Email clients         | Policies with _Email clients_ ticked | Webmail is fully covered. Outlook is partly readable; mail apps that don't use the system proxy (e.g. Thunderbird) aren't covered                           |

**Blocked programs:** the agent refuses every connection from a program on this list. Use it for
desktop apps that refuse the agent's certificate (the _AI services_ blind spots offer a **Block**
button for them), so people use the inspected web version instead. It only works for programs
that use the system proxy.

**Encrypted files:** the agent recognises password-protected archives and documents (zip, 7z,
rar, PDF, Office) without trying passwords. They're reported as _Encrypted or password-protected
file_; the default _Warn on encrypted files_ policy starts in monitor mode.

## Incidents

- Agents report incidents with values already **masked** on the PC (`4532-****-****-0366`). The
  server rejects anything that looks unmasked, and never shows or stores full values.
- **Filter** by status, action, data type, device, user, dates or risk, or search destinations,
  users, devices, policies and snippets. Sort by time, risk, destination or status. **Export CSV**
  downloads exactly what the filters show.
- The list updates **live** as incidents arrive (the _Live_ marker shows the connection).
- **Triage:** _Start investigating → Resolve_, or _Mark false positive_. Assign the incident to an
  analyst and add notes; notes must not contain sensitive values either. Each false positive
  counts against its data types, which shows on the data type page as a tuning signal.
- Incidents older than the retention period (Settings, default 180 days) are deleted
  automatically.

## Devices

- **Agents protect their PC from the moment they're installed**, with built-in rules and the AI
  services list, even before they've reached the server. Once a PC has a key they work fully
  offline too: they keep the last policies and queue everything they do (incidents, activity,
  performance) until they can upload it. Anything you do to a PC that's offline (disable, block,
  revoke its key, send a command, change its key's settings) waits on the server and takes effect
  the next time it connects.
- **Download agent** (button on _Devices_): give users a ready-to-run agent.
  - Upload the agent's setup `.exe` once, from its release page. Paste the release's SHA-256 too,
    and the upload is refused if the file doesn't match.
  - Then download a package: a zip with the installer, `agent-settings.toml` (this server's
    address) and the server's certificate, with a short README. Users extract it and
    double-click the setup; the installer reads the settings from its folder.
  - **No key** (recommended): the zip holds no secret. PCs appear as waiting and you assign a key,
    or a rule does.
  - **Include a new key**: creates a key with auto-approve on that only exists in that zip (the
    server keeps just its hash). PCs are active at once. Share it carefully and disable the key
    when the rollout is done.
  - Uploads and downloads are admin-only and audited, never with the key itself.
- **Joining:** install the agent with a **registration key** in its settings, or with none.
  - _With a key:_ keys with auto-approve make the PC active immediately. Others wait under
    **Pending approval**, where you **approve** or **reject** it (rejecting blocks the PC).
  - _Without a key:_ the PC appears under **Pending approval** with **No key**. Click **Assign
    key** and either pick an existing key (its settings are shown beside it) or **Create a new
    key** with its settings in the same dialog. Assigning a key makes the PC active. You get a
    _new PC waiting_ notification if a channel subscribes to it.
  - _Automatically:_ **assignment rules** on the _Registration keys_ page give a key to PCs that
    match a hostname pattern (`FIN-*`), an IP range (`10.20.0.0/16`) or a Windows user or domain
    (`CORP\*`). The first matching rule, in order, wins.
- **Keys** carry settings for the PCs that use them: auto-approve, an expiry date, a maximum
  number of uses, **device groups**, and offline settings (heartbeat and upload intervals, after
  how many days offline the user is told to reconnect, how long and how much the agent may
  queue). Changes reach the PCs at their next heartbeat. **Revoke key** on a device page sends
  that PC back to waiting (it keeps protecting with its last policies); **Revoke from its
  devices** on a key does it for every PC using the key. **Rotate** makes a new key with the same
  settings and moves its rules over.
- **Groups** come from the PC's key, and you can edit them on the device page. Policies can
  target groups (for example _Finance_) as well as departments and individual PCs.
- The **Status** column says whether a PC is actually protected:
  - _Protected_
  - _Degraded:_ the ML model is missing (rules still run), policies are out of date, or the proxy
    has a problem
  - _Offline:_ no heartbeat, so the agent may have been stopped or uninstalled
  - _At risk:_ the agent reports that its protection is being bypassed or tampered with: the
    system proxy was turned off or overridden, its certificate was removed, a browser isn't
    inspected (e.g. Firefox it couldn't configure) or a browser extension (such as a VPN
    extension) routes traffic around it, someone stopped the agent, or the user has
    administrator rights. A detected VPN only shows as _Degraded_, because detection is
    unreliable
  - _Disabled_, _Pending_, _Blocked_
  - A **Stale** tag marks PCs that haven't been seen for longer than the _stale after_ setting:
    probably retired or reinstalled. Remove them if so.
- **Controls** (each audited), on the device page or in bulk from the list:
  - **Disable:** the agent stops inspecting and shows "Disabled by administrator". **Enable**
    restores it.
  - **Block:** the PC gets one final "blocked" answer, then is cut off. It keeps enforcing its
    last policies. **Unblock** lets it register again.
  - **Send command:** _sync policies now_, _upload queued data now_, _request diagnostics_
    (masked log lines appear on the device page) or _restart agent_. The status goes pending → delivered → acknowledged as the
    agent picks it up.
- The device page shows OS, network and hardware details, health, performance charts (added
  latency, CPU, memory), history of IP, OS and agent changes, and the PC's incidents. Its
  **Agent activity** card shows what the agent reported after the fact: starts and stops
  (including crashes), policies applied, users who chose to proceed after a warning, and
  **offline periods** with how much it uploaded when it came back.

## Overview

Answers two questions at a glance:

- **Which devices are unprotected right now?** The _Unprotected right now_ card lists offline,
  disabled, at-risk and proxy-down PCs, oldest heartbeat first.
- **What leaked most this week?** Top data types, users and destinations, each linking to the
  matching incidents.

Plus today's counts, a 14-day incident chart, device health and fleet-wide added latency.

## Notifications

_Organisation → Notifications_ sends alerts so nobody has to watch the dashboard. Add a
**channel**:

- **Email:** a list of addresses, sent through your company mail relay. The relay is set in the
  server's `.env` (`SMTP_HOST`, `SMTP_FROM`, ...; see the install guide).
- **Webhook:** Microsoft Teams (a Workflows "post to a channel when a webhook request is received"
  URL), Slack (incoming webhook) or any system that accepts JSON. The URL is stored encrypted and
  never shown again.

Each channel chooses what it gets: **incidents** (blocks only, or warnings and blocks, ...),
**unprotected devices** and their recovery, **devices at risk**, **new devices waiting for
approval**, **new blind spots**, and **backup problems**. Bursts of incidents arrive as one
digest, and a channel gets at most 30 separate incident messages an hour. **Send test** checks a
channel straight away; failed sends are retried for about three hours and shown with the error.
Messages contain masked values only.

## Settings

- **Incident retention** (days), with a _Delete expired incidents now_ button.
- **Heartbeat interval:** sent to agents.
- **Offline after:** how long without a heartbeat before a PC counts as offline. Use at least
  twice the heartbeat interval.
- **Device registration:**
  - _Let agents register without a key_ (on by default). Turn it off to require a key in every
    agent's settings.
  - _At most waiting at once_ (500): more keyless PCs are refused until you deal with them.
  - _Remove unassigned after_ (30 days): PCs that never got a key and stopped connecting are
    deleted (audited).
  - _Stale after_ (30 days without contact): when to tag a PC as stale.
- **Backups:** when the last nightly backup ran, its size, whether it was copied off this disk,
  and any problem.
