# Connecting the desktop agents

The desktop agent is installed on every PC you want to protect. It inspects what's sent to AI
services and other sites, blocks or records sensitive data according to the server's policies,
and reports incidents (masked) to the server. Agent releases and their own documentation are in
[vault-dlp-agent-release](https://github.com/kudzaiprichard/vault-dlp-agent-release).

Each agent needs two things from the server: its **address** and its **root certificate**
(`<install folder>\agent\root.crt`). The dashboard can package both with the installer.

## The easy way: an agent package from the dashboard

1. **Upload the installer once.** Download the agent's setup `.exe` and its `.sha256` from the
   [agent releases](https://github.com/kudzaiprichard/vault-dlp-agent-release/releases/latest). In
   the dashboard, open _Devices > Download agent_, choose the `.exe`, paste the SHA-256 (or the
   `.sha256` file's content), and click **Upload installer**. The server checks the file is a
   Windows program and matches the hash.
2. **Download a package.** Choose the address agents should use, and whether to include a key:
   - **Without a key** (safe to share widely): new PCs appear under _Devices_ as _waiting for a
     key_ until you assign one, or an assignment rule does it for you.
   - **With a new key:** PCs are activated straight away. The key is created with auto-approve on
     and exists only in that zip; the server keeps just its hash. Treat that zip like a password.
3. **Give users the zip.** It contains the installer, `agent-settings.toml` (the server address,
   and the key if you chose one), `root.crt` and a README. Users extract it and run the setup;
   there's nothing to type. Setup asks Windows to trust the agent's own certificate: answer _Yes_.

Every upload and download is recorded in the audit log (never with the key).

## Keys, approval and assignment rules

- **Registration keys** (_Registration keys_) decide whether a new PC is accepted. A key can
  auto-approve, or leave PCs _pending_ until an admin approves them under _Devices_.
- **Assignment rules** give waiting PCs a key automatically, for example every PC whose name
  starts with `DESKTOP-`.
- Revoking a key stops new registrations with it. PCs that already registered keep working until
  you disable or block them under _Devices_.

## Setting up an agent by hand

Point the agent at the server in its settings:

```toml
[server]
url = "https://vault-server"          # one of the server's addresses
registration_key = "vrk_..."          # optional: without it the PC waits for a key
ca_cert = "C:\\ProgramData\\...\\root.crt"
```

Copy `root.crt` from the server's install folder to the PC. Use `ca_cert`, not certificate
pinning: the server's site certificate rotates about every 12 hours, while the root lasts 10
years. See the agent's documentation for where its settings file lives.

## Checking a PC

Open it under _Devices_. The page shows its state (active, waiting, pending, disabled, blocked),
health (healthy, degraded, offline), the agent version, the last heartbeat, which AI services it
can inspect, and recent activity. _Degraded_ lists the reasons, including uploads the server
refused.

Commands on the device page ask the agent to sync policies now, upload queued data, send
diagnostics or restart. They're delivered with its next heartbeat.
