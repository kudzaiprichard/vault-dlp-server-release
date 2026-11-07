# Installing the Vault server

## Requirements

| Need     | Notes                                                                                                                       |
| -------- | --------------------------------------------------------------------------------------------------------------------------- |
| Windows  | 64-bit Windows 10 or 11, or Windows Server 2019 or later                                                                    |
| Hardware | 4 GB RAM and 5 GB free disk space at least. More disk for many incidents and long backup retention                          |
| Rights   | Administrator, for a Windows service, the firewall rule and certificate trust. A manual install for testing can run without |
| Network  | One HTTPS port that the agents' PCs can reach (443 by default)                                                              |

Nothing else is needed: Node.js, PostgreSQL and Caddy (HTTPS) are inside the zip, and the server
works without internet access.

## 1. Download and check

From the [latest release](https://github.com/kudzaiprichard/vault-dlp-server-release/releases/latest),
download `Vault-Server-v<version>-win-x64.zip` and `Vault-Server-v<version>-win-x64.zip.sha256`.
In PowerShell, in the download folder:

```powershell
(Get-FileHash .\Vault-Server-v0.4.1-win-x64.zip -Algorithm SHA256).Hash.ToLower()
Get-Content .\Vault-Server-v0.4.1-win-x64.zip.sha256
```

Both lines must show the same value. If they don't, download again.

## 2. Extract

Right-click the zip > _Extract All_. Don't run setup from inside the zip.

If Windows shows "Windows protected your PC" when you start setup, the files are marked as coming
from the internet: click _More info_ > _Run anyway_, or right-click the zip > _Properties_ > tick
_Unblock_ > _OK_ and extract it again.

## 3. Run setup

Double-click **`Setup Vault.cmd`**. Press Enter to keep the suggestion shown in `[brackets]`.

| Question                               | Default                                        | Notes                                                                                                                                    |
| -------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Install folder                         | `C:\Vault`                                     | Where Vault runs from. The extracted folder isn't needed afterwards                                                                      |
| How should Vault run?                  | Windows service                                | **Service:** starts with the PC, even when no one is signed in. **Manual:** you start and stop it with Vault Control, under your account |
| Addresses                              | `localhost, 127.0.0.1`, this PC's name and IPs | Names and IPs that people and agents use. The **first** one is the dashboard link. Include the name or IP agents on other PCs will use   |
| HTTPS port                             | 443 (or 8443, 9443 if taken)                   | The dashboard and agent port                                                                                                             |
| Internal app port                      | 3000                                           | Used only on this PC                                                                                                                     |
| Database                               | Built-in PostgreSQL                            | Or an existing PostgreSQL server: give its connection URL                                                                                |
| First administrator                    | `admin@vault.local`                            | New installs only. Leave the password empty to generate one                                                                              |
| Nightly backups                        | On, 02:00, kept 14 days                        | Optionally copied to another disk or a network share                                                                                     |
| Email alerts (SMTP)                    | Off                                            | Your mail server, port, user and sender                                                                                                  |
| Windows Firewall                       | Yes                                            | Needed for agents on other PCs                                                                                                           |
| Trust the HTTPS certificate on this PC | Yes                                            | No browser warnings on this PC                                                                                                           |
| Desktop shortcut                       | Yes                                            | A _Vault Control_ shortcut                                                                                                               |

Setup shows a summary and asks to go ahead. It then asks for administrator rights (approve the
Windows prompt) and continues in a new window. It:

1. Copies Vault to the install folder.
2. Creates the database (built-in PostgreSQL) and writes the secrets to `server\.env`.
3. Sets up HTTPS with Vault's own certificate authority, and copies its root certificate to
   `agent\root.crt` for the agents.
4. Registers the services and scheduled tasks, the firewall rule and the nightly backup.
5. Starts Vault, applies the database migrations and, on an empty database, creates the first
   administrator and the default data types and policies.
6. Opens the dashboard.

**Administrator password.** At least 12 characters, with three of: lower case, upper case,
digits, symbols. It can't contain the email name. If setup generated one, it's shown **once**:
write it down. You'll choose a new password at first sign-in.

## 4. First sign-in

Open the dashboard address (`Vault Control.cmd open`) and sign in. Then:

1. **Create a second administrator** under _Users_, so a forgotten password can always be reset.
2. **Review the policies.** Blocking of private keys, payment card numbers and secrets is on from
   the start; the other policies start in _monitor_ mode, which only records.
3. **Connect the agents:** [Connecting agents](connecting-agents.md).

## Unattended setup

For scripted or repeated installs, put the answers in a settings file. Copy
`vault.config.example.json` from the zip, edit it, and run (as administrator for a service
install):

```powershell
& ".\Setup Vault.cmd" -Config my-vault.json -Quiet -AdminPassword "Choose-A-Strong-Pass1"
```

| Option                      | Meaning                                                                                 |
| --------------------------- | --------------------------------------------------------------------------------------- |
| `-Config <file>`            | The settings file (below)                                                               |
| `-Quiet`                    | Ask nothing; use the file and the defaults                                              |
| `-InstallDir <folder>`      | Install folder (overrides the file)                                                     |
| `-AdminPassword <password>` | The first administrator's password (new installs). Generated and shown once if left out |
| `-DatabaseUrl <url>`        | Use your own PostgreSQL: `postgresql://user:password@host:5432/vault?schema=public`     |
| `-SmtpPassword <password>`  | The mail server password                                                                |

Passwords never go in the settings file: pass them as options.

### Settings file

Leave out anything you don't want to change; missing values use the defaults.

```json
{
  "name": "Vault",
  "installDir": "C:\\Vault",
  "runMode": "service",
  "hosts": ["localhost", "127.0.0.1", "vault-server", "192.168.1.10"],
  "httpsPort": 443,
  "appPort": 3000,
  "database": { "mode": "bundled", "port": 5432, "name": "vault", "url": "" },
  "admin": { "email": "admin@example.com", "name": "Administrator" },
  "backups": { "enabled": true, "time": "02:00", "keepDays": 14, "copyTo": "" },
  "smtp": {
    "host": "",
    "port": 587,
    "secure": false,
    "user": "",
    "from": "Vault DLP <vault@example.com>",
    "allowSelfSigned": false
  },
  "firewall": true,
  "trustCertificate": true,
  "desktopShortcut": true,
  "openBrowser": true
}
```

| Setting                               | Meaning                                                                                        |
| ------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `name`                                | Name used for the services and tasks (`<name> Server`, `<name> HTTPS`, `<name> Backup`)        |
| `installDir`                          | Install folder                                                                                 |
| `runMode`                             | `service` (starts with the PC, needs administrator) or `manual`                                |
| `hosts`                               | Names and IPs for the dashboard and agents. The first is the dashboard link                    |
| `httpsPort`                           | Dashboard and agent port                                                                       |
| `appPort`                             | Internal port, only on this PC                                                                 |
| `database.mode`                       | `bundled` (built-in PostgreSQL on this PC) or `external` (your server, URL via `-DatabaseUrl`) |
| `database.port`, `database.name`      | Built-in database port and name                                                                |
| `admin.email`, `admin.name`           | First administrator (new, empty database only)                                                 |
| `backups.enabled`, `time`, `keepDays` | Nightly backup, its time (24-hour) and how long backups are kept                               |
| `backups.copyTo`                      | Also copy each backup to this folder or share, e.g. `\\fileserver\vault-backups`               |
| `smtp.*`                              | Mail server for alerts. Empty `host` turns email off. `secure` is for port 465                 |
| `firewall`                            | Allow other PCs through Windows Firewall to `httpsPort`                                        |
| `trustCertificate`                    | Trust Vault's certificate on this PC                                                           |
| `desktopShortcut`                     | Put a _Vault Control_ shortcut on the desktop                                                  |
| `openBrowser`                         | Open the dashboard when setup finishes                                                         |

Setup saves the settings used in `<install folder>\vault.config.json` (no passwords), so an
upgrade or `Vault Control.cmd config` starts from them.

## Ports and network

| Port            | Who uses it                 | Open in the firewall?        |
| --------------- | --------------------------- | ---------------------------- |
| HTTPS (443)     | Browsers and agents         | Yes, for agents on other PCs |
| App (3000)      | Only Caddy, on this PC      | No                           |
| Database (5432) | Only the server, on this PC | No                           |

The server uses no other ports and makes no outbound connections, except to your mail server and
webhooks if you configure them.

## Certificates

Vault runs its own certificate authority (Caddy's `tls internal`). Its root certificate is valid
for 10 years and is copied to `<install folder>\agent\root.crt`. Agents trust the server through
that file; the dashboard's [agent packages](connecting-agents.md) include it automatically.

To open the dashboard from another PC without a browser warning, import `root.crt` on that PC
under _Trusted Root Certification Authorities_.

Upgrades and address changes keep the same certificate authority, so agents keep trusting the
server.
