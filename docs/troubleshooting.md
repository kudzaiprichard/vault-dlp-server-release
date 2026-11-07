# Troubleshooting

Start with `Vault Control.cmd status`: it shows what's running, the dashboard address, the
version and the last backup. `Vault Control.cmd logs` shows the latest server log lines
(`logs https`, `logs database` and `logs backup` for the other parts).

## Setup

| Problem                                  | Fix                                                                                                                                                                  |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| "A port is in use"                       | Another program uses it (often 443). Accept the suggested port, or stop that program. The dashboard address then includes the port, e.g. `https://vault-server:8443` |
| "Windows protected your PC"              | The files came from the internet. _More info_ > _Run anyway_, or unblock the zip (_Properties_ > _Unblock_) and extract it again                                     |
| Setup is stuck at the administrator step | Approve the Windows prompt. Setup continues in a new window                                                                                                          |
| The admin password is refused            | At least 12 characters, three of: lower case, upper case, digits, symbols, and not containing the email name                                                         |

## Dashboard

| Problem                                       | Fix                                                                                                                                                                                            |
| --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| The browser says the connection isn't private | This PC doesn't trust Vault's certificate. Run `Vault Control.cmd config` and answer _Yes_ to trusting it. On other PCs, import `agent\root.crt` into _Trusted Root Certification Authorities_ |
| The dashboard doesn't open after a restart    | A manual install doesn't start with Windows: `Vault Control.cmd start`, or switch to a service with `config`. If it's paused, `resume`                                                         |
| "502" or the page doesn't load                | The server isn't running. `Vault Control.cmd logs`, then `restart`                                                                                                                             |
| "Invalid server configuration" in the log     | A secret in `server\.env` is missing or too short. Re-run setup to repair it                                                                                                                   |
| "Applying database migrations… failed"        | The database isn't running or can't be reached. `logs database`; for your own PostgreSQL, check its URL and password                                                                           |
| "Too many failed sign-in attempts"            | Wait 15 minutes, or another admin unlocks the account under _Users_                                                                                                                            |
| Forgot the admin password                     | Another admin resets it under _Users_. Create a second admin early                                                                                                                             |

## Agents

| Problem                                | Fix                                                                                                                                                                     |
| -------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Agents on other PCs can't connect      | The address they use must be in setup's address list; the firewall question must be answered _Yes_; test from the other PC: `Test-NetConnection vault-server -Port 443` |
| Certificate errors on the agent        | The agent must trust `agent\root.crt` from this server. Agent packages from _Devices > Download agent_ include it                                                       |
| A PC is _waiting for a key_            | Assign a key on its device page, or add an assignment rule                                                                                                              |
| A PC is _pending_                      | Approve it under _Devices_, or use a key with auto-approve                                                                                                              |
| A PC is _degraded_                     | Open it: the reasons say what's wrong, such as uploads the server refused or a stale policy                                                                             |
| PCs show _offline_                     | No heartbeat for longer than _Settings > Offline after_: the agent is stopped, uninstalled, or can't reach the server                                                   |
| "Download agent package" is greyed out | Upload the agent installer first, on the same page                                                                                                                      |

## Backups and alerts

| Problem                              | Fix                                                                                                                                              |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| _Settings > Backups_ shows a failure | `Vault Control.cmd logs backup`. For a network copy, the computer account needs write access to the share                                        |
| Alerts don't arrive                  | _Notifications > Recent notifications_ shows each send and its error. Use **Send test**. For email, check the mail server settings with `config` |

## Still stuck

Collect `Vault Control.cmd status` and the last lines of `logs\server.log`, and
[open an issue](https://github.com/kudzaiprichard/vault-dlp-server-release/issues). Remove any
addresses, names or keys you don't want to share first.
