# Running the Vault server

## Vault Control

**Vault Control** is in the install folder (`C:\Vault\Vault Control.cmd` by default), with a
desktop shortcut if you chose one. Double-click it for a menu, or run it with a command:

| Command                                            | What it does                                                                                                               |
| -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `status`                                           | What's running, the dashboard address, the version and the last backup                                                     |
| `start`                                            | Start the database, the server and HTTPS                                                                                   |
| `stop`                                             | Stop until the next start. A service install still starts with the PC                                                      |
| `restart`                                          | Stop, then start                                                                                                           |
| `pause`                                            | Stop, and don't start with the PC until `resume`                                                                           |
| `resume`                                           | Undo `pause` and start                                                                                                     |
| `open`                                             | Open the dashboard                                                                                                         |
| `logs [server\|https\|database\|backup] [-Follow]` | The latest log lines; `-Follow` keeps watching                                                                             |
| `backup`                                           | Back up the database now                                                                                                   |
| `restore -File <backup.dump>`                      | Restore a backup. **Replaces all current data**                                                                            |
| `config`                                           | Change the addresses, ports, run mode, backups or email (re-runs setup's questions)                                        |
| `uninstall [-RemoveData]`                          | Remove services, tasks, firewall rule and certificate. `-RemoveData` also deletes the install folder, database and backups |

Add `-Yes` to skip a confirmation. For a service install, commands that change something ask for
administrator rights.

**When the server is stopped or paused**, the desktop agents keep protecting their PCs with their
last policies and queue their reports. They send them when the server is back.

`stop` (and `restart`, `pause`, upgrades) first asks the server to shut down by itself, so it
withdraws its network announcement at once; whatever doesn't stop within 20 seconds is
force-stopped. If the server is killed some other way, Windows drops the announcement within
about 30 seconds.

## Where things are

In the install folder:

| Path                | What it is                                                   |
| ------------------- | ------------------------------------------------------------ |
| `vault.config.json` | The settings chosen in setup (no passwords)                  |
| `server\.env`       | Secrets and server settings. Administrators only             |
| `logs\`             | `server.log`, `caddy.log`, `postgres\`, `backup-status.json` |
| `backups\`          | Database backups                                             |
| `pgdata\`           | The built-in database                                        |
| `caddy\data\`       | HTTPS certificate authority                                  |
| `agent\root.crt`    | The root certificate the agents trust                        |

## Backups

With backups on, a backup runs every night at the time chosen in setup and backups older than
the retention period are deleted. `Vault Control.cmd backup` makes one at any time.

- Each backup is checked by reading it back, and the result is shown under _Settings > Backups_
  in the dashboard. Admins subscribed to _Backup problems_ are notified if a backup or its copy
  fails, or none finishes for 36 hours.
- **Keep a copy off this disk.** Set _Also copy backups to_ (`backups.copyTo`) to another disk or
  a share such as `\\fileserver\vault-backups`. Each copy is checked against the original's
  SHA-256. A service install copies as the computer account (`DOMAIN\PCNAME$`), which needs write
  access to the share.
- **Also keep a copy of** `server\.env` (the secrets) and `caddy\data` (the certificate
  authority). Without the certificate authority, every agent would need a new `root.crt`.

## Restore

```powershell
& "C:\Vault\Vault Control.cmd" restore -File C:\Vault\backups\vault-20261106-020000.dump
```

This **replaces all data** with the backup's: anything recorded after the backup was made is lost,
so restore the newest backup you can. Without `-File`, Vault Control lists the latest backups to
choose from. Vault is stopped during the restore and started again afterwards, even if the restore
fails.

## Upgrading

1. Back up: `Vault Control.cmd backup`.
2. Download and check the new release, and extract it.
3. Run its `Setup Vault.cmd`. It finds the existing install and keeps the database, keys,
   settings and secrets, then updates the program files and applies any database changes.
4. Check `Vault Control.cmd status` shows the new version.

The dashboard is down for a minute or two; agents queue their reports meanwhile. Agents keep
trusting the server, since its certificate authority doesn't change.

Setup stops the old version cleanly before replacing it: the server is asked to shut down, so it
withdraws its network announcement and finishes what it's doing. Anything still running from the
install folder after that, including a backup in progress or a stuck helper, is force-stopped, and
locked files are retried, so the old version is always fully replaced. Versions before 0.4.3 had
no clean stop and are force-stopped.

To upgrade without questions: `& ".\Setup Vault.cmd" -Quiet`. It uses the saved settings.

A **major** version may need the agents to be upgraded too. Its release notes say so.

## Changing settings

`Vault Control.cmd config` asks setup's questions again, starting from the current settings. Use
it to add an address (a new server name or IP for agents), change a port, switch between service
and manual, or change backups and email. New addresses get certificates from the same certificate
authority, so agents don't need a new `root.crt`.

## Email and webhooks

Email alerts use the mail server set in setup (or `config`). Webhooks for Teams, Slack or other
tools are set up in the dashboard under _Notifications_ and need no server settings, but the
server must be able to reach the webhook's address. _Notifications > Recent notifications_ shows
each delivery and its error, and **Send test** checks a channel.

## Uninstalling

```powershell
& "C:\Vault\Vault Control.cmd" uninstall              # keeps the data
& "C:\Vault\Vault Control.cmd" uninstall -RemoveData  # deletes the install folder, database and backups
```

Uninstalling removes the services, scheduled tasks, firewall rule and the trusted certificate.
`-RemoveData` asks you to type `DELETE` to confirm. Copy anything you want to keep out of
`backups\` first.

## Security notes

- Every dashboard page and action checks the user's role. Every agent request checks the device's
  token and state. Sign-in is throttled per address and per account.
- The server only stores masked values. Agent uploads that look unmasked are refused and nothing
  from them is stored.
- Responses use a strict Content Security Policy, `X-Frame-Options: DENY`, `nosniff`,
  `Referrer-Policy: no-referrer` and HSTS.
- The server only listens on this PC's HTTPS port; the app and database ports stay local.
