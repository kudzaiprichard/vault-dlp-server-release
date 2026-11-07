# Setting up Vault in a new organisation

This guide takes you from nothing to every employee PC protected. The order matters:

1. [Pick the server machine](#1-pick-the-server-machine)
2. [Install Vault Server on it](#2-install-vault-server)
3. [Sign in and change the default admin password](#3-sign-in-and-secure-the-admin-account)
4. [Upload the agent installer once](#4-upload-the-agent-installer-once)
5. [Download the agent package once](#5-download-the-agent-package)
6. [Install the agent on each PC](#6-install-the-agent-on-each-pc)
7. [Approve the PCs and check they're protected](#7-approve-the-pcs-and-check-theyre-protected)

Then [keep it running](#8-keep-it-running): backups, upgrades and adding addresses later.

**What you need:** one Windows PC or server for Vault Server, administrator rights on it, and a
way to get a zip file to each employee PC (a file share, email, a USB stick, or Intune/Group
Policy). The server works without internet access.

---

## 1. Pick the server machine

Vault Server runs on one Windows machine. Every employee PC sends its reports there, so choose
one that:

| Need                                                                                            | Why                                                                                                                |
| ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| 64-bit Windows 10 or 11, or Windows Server 2019 or later; 4 GB RAM, 5 GB free disk              | The minimum. Give it more disk for many PCs and long backup retention                                              |
| **Stays on** during working hours (ideally always)                                              | Agents keep protecting while the server is off, but reports wait until it's back                                   |
| **A fixed address**: a static IP (or DHCP reservation) and/or a DNS name such as `vault-server` | Agents connect to this address, and the server's certificate is issued for it. If it changes, agents can't connect |
| **Reachable from every employee PC** on TCP port **443**                                        | The dashboard and the agents both use HTTPS on this port (another port if 443 is taken)                            |

Write down the address the employee PCs will use, for example `vault-server` or `192.168.1.10`.
Don't use `localhost` or `127.0.0.1`: those only mean "this PC".

On a domain network, ask whoever runs DNS to add a name for the server (an A record, e.g.
`vault-server.corp.example` → its IP). Names survive an IP change; IPs don't.

## 2. Install Vault Server

On the server machine:

1. **Download** `Vault-Server-v<version>-win-x64.zip` and its `.sha256` file from the
   [latest release](https://github.com/kudzaiprichard/vault-dlp-server-release/releases/latest).
2. **Check the download** in PowerShell, in the download folder. The two values must match:

   ```powershell
   (Get-FileHash .\Vault-Server-v0.4.6-win-x64.zip -Algorithm SHA256).Hash.ToLower()
   Get-Content .\Vault-Server-v0.4.6-win-x64.zip.sha256
   ```

3. **Extract** it (right-click > _Extract All_).
4. **Run `Setup Vault.cmd`.** It shows recommended settings for this PC. Check the **Dashboard**
   line's _also_ list: it must include the address you wrote down in step 1.
   - Setup adds this PC's name, `<name>.local` and its network IPs by itself. If that covers your
     address, press **Enter** to install.
   - If it doesn't (for example a DNS name like `vault-server.corp.example`), answer **No** and
     add the address to the _Addresses_ question, comma-separated. The certificate only covers
     the addresses listed, so a PC using any other name gets a certificate error.
   - Keep **Windows Firewall: Yes**. It opens the HTTPS port on private and domain networks so
     employee PCs can connect. If the network is marked _Public_, change it to _Private_ in
     Windows settings, or open the port yourself.
5. **Approve the Windows prompt** for administrator rights. Setup finishes in a minute or two
   and opens the dashboard.

Setup runs Vault as a Windows service, so it starts with the PC even when nobody is signed in.
The [installation guide](installation.md) explains every question and the unattended install.

**Check from an employee PC** that it can reach the server (PowerShell):

```powershell
Test-NetConnection vault-server -Port 443     # TcpTestSucceeded : True
```

## 3. Sign in and secure the admin account

1. Open `https://<server address>` (or the _Vault Dashboard_ shortcut on the server).
2. Sign in with the **default administrator**:
   - email `admin@vault.local`
   - password `@Vault12`
3. **Change that password straight away**: account menu (bottom left) > _Change password_. Every
   Vault install starts with the same default, so until you change it anyone who can reach the
   dashboard can sign in. A red banner warns every admin while it's still in use.
4. **Add your own admin accounts** under _Users > Add user_ (role _Admin_). Each person gets a
   one-time password and chooses their own at first sign-in. Have at least two admins, so one can
   always reset the other's password.
5. Optionally **deactivate the default account** once your own admins work: _Users_ > its menu >
   _Edit_ > turn off _Active_. You can't deactivate the last active admin.

Opening the dashboard from another PC shows a certificate warning until that PC trusts Vault's
certificate. To remove it, import `<install folder>\agent\root.crt` on that PC into _Trusted Root
Certification Authorities_. Agents trust it automatically (step 6).

**Before rolling out**, look at _Policies_: card numbers, private keys and secrets are blocked
from the start; everything else starts in _monitor_ mode, which only records. Adjust them now or
after watching a few days of incidents. The [admin guide](admin-guide.md) covers every page.

## 4. Upload the agent installer once

The server doesn't include the desktop agent; you add it once, and every package uses it.

1. From the
   [latest agent release](https://github.com/kudzaiprichard/vault-dlp-agent-release/releases/latest),
   download `DLP-Agent_<version>_x64-setup.exe` and `DLP-Agent_<version>_x64-setup.exe.sha256`.
2. In the dashboard: _Devices_ > **Download agent**.
3. Under _Agent installer (.exe)_, choose the `.exe`.
4. Open the `.sha256` file in Notepad and paste its content into _SHA-256 from the release page_.
5. Click **Upload installer**. The server checks the file is a Windows program and matches the
   SHA-256, then shows its version and hash. If the hash doesn't match, download the installer
   again.

When a new agent version comes out, upload it the same way; new packages use it from then on.

## 5. Download the agent package

Still on _Devices > Download agent_:

1. **Server address the agents use:** choose the address the employee PCs can reach (the name
   or IP from step 1). **Never `localhost` or `127.0.0.1`**: on an employee PC that would point
   at the PC itself.
2. **Key:**
   - **No key (recommended):** safe to share widely. New PCs show up under _Devices_ as _waiting
     for a key_, and you assign one (step 7).
   - **Include a new key:** PCs are activated as soon as they're installed. The key is in the zip
     and nowhere else, so treat the zip like a password.
3. Click **Download agent package**. You get `DLP-Agent-setup-<address>.zip` with the installer,
   the server's address and certificate (`agent-settings.toml`, `root.crt`) and a short README for
   users.

One package serves every PC. Make a new one only if the address, the key choice or the agent
version changes.

## 6. Install the agent on each PC

Give each employee PC the package. The
[agent's setup guide](https://github.com/kudzaiprichard/vault-dlp-agent-release/blob/main/docs/new-environment.md)
covers this part in detail, including silent and managed (Intune, Group Policy) installs. In
short, on each PC:

1. **Extract** the zip (right-click > _Extract All_). Running the setup from inside the zip leaves
   the server settings behind.
2. Double-click **`DLP-Agent_<version>_x64-setup.exe`** in the extracted folder.
3. Windows asks to install a certificate called **"DLP Agent Local Root CA"**: click **Yes**. It
   lets the agent check what is sent to AI services, and never leaves the PC. If you don't see the
   prompt, look for it on the taskbar.

The agent protects the PC from that moment, even before it reaches the server.

## 7. Approve the PCs and check they're protected

1. Open _Devices_. New PCs appear within a minute of installing.
   - Installed from a package **without a key**: the PC is _waiting for a key_. Open it and
     **Assign key**, or add an assignment rule once under _Registration keys_ > _New rule_
     (e.g. every PC named `DESKTOP-*`) so new PCs get a key automatically.
   - With a key that doesn't auto-approve, the PC is _pending_: **Approve** it.
2. Each active PC shows _healthy_, its agent version and its last heartbeat. _Degraded_ or
   _offline_ lists the reason.
3. **Test one PC:** in Chrome or Edge, type a made-up card number such as
   `4111 1111 1111 1111` into ChatGPT and send it. The agent blocks it, and the incident appears
   under _Incidents_ within a minute, masked as `4111-****-****-1111`.

## 8. Keep it running

- **Backups:** nightly by default. Set a second copy on another disk or a share (_Vault
  Control.cmd config_ > backups), and keep a copy of `server\.env` and `caddy\data`. See
  [operations](operations.md#backups).
- **Alerts:** add email or a Teams/Slack webhook under _Notifications_, so someone hears about
  blocks, PCs that stop reporting and failed backups.
- **Adding an address later** (a new DNS name, a changed IP): run _Vault Control.cmd config_ on the
  server and add it to the addresses. Certificates for it come from the same authority, so
  agents don't need anything new. Make a new package (step 5) for PCs installed afterwards.
- **Upgrading the server:** download the new release, extract it and run its `Setup Vault.cmd`,
  then press Enter at _Update and keep these settings?_. Data, keys and settings are kept, and
  agents keep protecting during the minute it takes.
- **Upgrading agents:** upload the new installer (step 4), download a new package (step 5) and
  install it over the old agent on each PC. PCs keep their identity.
- **Removing Vault:** _Vault Control.cmd uninstall_ on the server (`-RemoveData` also deletes the
  database and backups); uninstall the agent from _Apps_ on each PC.

If something doesn't work, see [troubleshooting](troubleshooting.md).
