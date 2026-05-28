---
name: paperclip-setup
description: "Set up and configure Paperclip AI with the Hermes adapter on Windows — local npm install, PostgreSQL trust auth, agent config via psycopg2."
version: 1.0.0
license: MIT
platforms: [windows]
tags: [paperclip, hermes, devops, windows, agents, setup]
related_skills: [paperclip-debugging, paperclip-restart]
---

# Paperclip AI — Windows Setup with Hermes Adapter

## First-Run Setup

### What You Need

| Requirement | Where to Get |
|---|---|
| Node.js v20+ | https://nodejs.org |
| Python 3.10+ | https://python.org |
| Git for Windows (Git Bash) | https://git-scm.com |
| Hermes Agent | https://hermes-agent.nousresearch.com |
| OpenRouter API key | https://openrouter.ai |

**Hermes Agent must be installed and accessible from your PATH:**
```bash
hermes --version   # confirm it works
```

### Find Your Hermes Path

```bash
which hermes       # e.g. /c/Users/<username>/.hermes-personal/hermes.sh
# OR
where hermes       # Windows: e.g. C:\Users\<username>\.hermes-personal\hermes.cmd
```

Save this path — you'll use it as `hermesCommand` in agent configs.

---

## Paperclip Deployment Model

Paperclip runs as a **local npm project**, not via `npx paperclipai@latest` (which
pulls fresh packages on every run and wipes any patches you've applied).

**Why local npm:**
- Deterministic installs — no version drift
- `patch-package` patches survive `npm install`
- Full control over startup environment

---

## Step 1: Install Paperclip as a Local npm Project

Create a dedicated install directory:

```bash
mkdir -p ~/.paperclip-stack/install
cd ~/.paperclip-stack/install
```

Create `package.json`:

```json
{
  "name": "paperclip-stack",
  "version": "1.0.0",
  "dependencies": {
    "paperclipai": "2026.517.0",
    "hermes-paperclip-adapter": "0.3.0",
    "@paperclipai/adapter-utils": "2026.517.0",
    "patch-package": "^8.0.0"
  },
  "scripts": {
    "postinstall": "patch-package"
  }
}
```

Install:
```bash
npm install --legacy-peer-deps
```

---

## Step 2: Create the Startup Files

### `server.js` (bypasses the CLI wizard — required for background/non-TTY runs)

```js
// ~/.paperclip-stack/install/server.js
process.env.PAPERCLIP_INSTANCE_DIR = process.env.PAPERCLIP_INSTANCE_DIR
    || "C:\\Users\\<YOUR_USERNAME>\\.paperclip\\instances\\default";

const { startServer } = require("./node_modules/@paperclipai/server/dist/index.js");
console.log("Starting Paperclip server...");
startServer().catch(err => {
    console.error("SERVER FATAL:", err.message);
    process.exit(1);
});
```

### `start-paperclip.bat` (sets required env vars — run this, not Git Bash)

```batch
@echo off
SET SYSTEMROOT=C:\WINDOWS
SET PAPERCLIP_INSTANCE_DIR=C:\Users\<YOUR_USERNAME>\.paperclip\instances\default
SET PAPERCLIP_MIGRATION_AUTO_APPLY=true
cd /d %~dp0install
node server.js
```

**Critical:** Always start via this `.bat` file or PowerShell. Never from Git Bash —
`SystemRoot` is unset in Git Bash which causes `spawn EFTYPE` errors.

---

## Step 3: Onboard Paperclip (First Time Only)

```bash
# From PowerShell or CMD (not Git Bash):
cd C:\Users\<YOUR_USERNAME>\.paperclip-stack\install
npx paperclipai onboard --yes
```

This creates the instance data at `~/.paperclip/instances/default/`.

---

## Step 4: Apply Windows Adapter Patches

Two critical patches for Windows compatibility:

### Patch 1: Newline sanitisation in `quoteForCmd()`

Without this, agent prompts containing newlines cause `cmd.exe` to fail with exit code 2.

### Patch 2: `.sh` script dispatch

Windows can't execute `.sh` files directly. This patch dispatches `.sh` commands via `bash.exe`.

Create the patches directory and add your patch files:

```bash
mkdir -p ~/.paperclip-stack/install/patches
```

**patches/@paperclipai+adapter-utils+2026.517.0.patch:**

```diff
--- a/node_modules/@paperclipai/adapter-utils/dist/index.js
+++ b/node_modules/@paperclipai/adapter-utils/dist/index.js
@@ quoteForCmd function @@
-  return `"${str}"`;
+  return `"${str.replace(/\n/g, ' ').replace(/\r/g, '')}"`;
```

(See the Paperclip adapter source for exact line numbers — they vary by version.)

Apply patches:
```bash
cd ~/.paperclip-stack/install
npx patch-package
```

---

## Step 5: Enable PostgreSQL Trust Auth (Skip Password Hunting)

Paperclip uses embedded PostgreSQL with a dynamically generated password that's hard
to retrieve from logs. Bypass it entirely with trust auth:

**Find your pg_hba.conf:**
```
~/.paperclip/instances/default/db/pg_hba.conf
```

**Change the auth method to `trust`:**
```
# Before:
host    all    all    127.0.0.1/32    password
# After:
host    all    all    127.0.0.1/32    trust
```

This is safe — embedded Postgres binds only to `127.0.0.1` with no external network access.

**Restart Paperclip** for the change to take effect.

---

## Step 6: Start Paperclip and Get Your Company ID

```bash
# Start (from PowerShell or double-click the .bat):
start-paperclip.bat

# Health check:
curl -s http://127.0.0.1:3200/api/health
```

**Find your Company ID:**

```bash
curl -s http://127.0.0.1:3200/api/companies | python3 -c "
import json, sys
for c in json.load(sys.stdin):
    print(c['id'], c['name'])
"
```

Save your Company ID — you'll use it in every subsequent API call.

---

## Step 7: Configure Agents via PostgreSQL

Paperclip v2 does NOT support updating `adapterConfig` via the REST API.
Use Python + psycopg2 directly:

```python
import psycopg2, json

# Find your PG port from postmaster.pid (line 4):
# ~/.paperclip/instances/default/db/postmaster.pid

conn = psycopg2.connect(
    host='127.0.0.1',
    port=54330,          # <-- check postmaster.pid for your actual port
    user='paperclip',
    dbname='paperclip'
    # no password needed — trust auth
)
cur = conn.cursor()

CO = '<YOUR_COMPANY_ID>'    # from Step 6

updated_config = json.dumps({
    "hermesCommand": r"C:\Users\<YOUR_USERNAME>\.hermes-personal\hermes.cmd",
    "maxTurns": 5,
    "persistSession": False,
    "timeoutSec": 600,
    "quiet": False,
    "verbose": True,
    "model": "anthropic/claude-sonnet-4-6",   # or your preferred model
    "provider": "openrouter"
})

cur.execute(
    "UPDATE agents SET adapter_config = adapter_config || %s::jsonb WHERE company_id = %s",
    (updated_config, CO)
)
conn.commit()
print(f"Updated {cur.rowcount} agents")
conn.close()
```

**Install psycopg2 if needed:**
```bash
pip install psycopg2-binary
```

---

## Step 8: Verify Agent Config

```python
import psycopg2, json

conn = psycopg2.connect(host='127.0.0.1', port=54330, user='paperclip', dbname='paperclip')
cur = conn.cursor()

cur.execute("""
    SELECT name,
           adapter_config->>'hermesCommand' as cmd,
           adapter_config->>'maxTurns' as turns,
           adapter_config->>'model' as model
    FROM agents
    WHERE company_id = '<YOUR_COMPANY_ID>'
""")

for row in cur.fetchall():
    print(f"{row[0]:<30} cmd={row[1]} turns={row[2]} model={row[3]}")
```

---

## Step 9: Wake an Agent

```bash
# Via API
curl -s -X POST "http://127.0.0.1:3200/api/agents/<AGENT_ID>/wakeup" \
  -H "Content-Type: application/json" \
  -d '{"reason": "test"}'

# Via CLI (if pc is installed)
pc agent wake <agent-name> --reason test
```

---

## Clean Config Checklist

After setup, verify every agent has:

- [ ] `hermesCommand` — full path to your Hermes binary (`.cmd` or `.sh`)
- [ ] `maxTurns: 5` — prevents runaway spending
- [ ] `promptTemplate: ""` — MUST be empty (custom templates cause prompt injection)
- [ ] `persistSession: false` — prevents duplicate context across heartbeats
- [ ] `timeoutSec: 600` — Hermes cold boot takes 25–35s; 60s default is too short
- [ ] `quiet: false` + `verbose: true` — lets you see what agents actually do

---

## Known Issues (Windows)

| Symptom | Cause | Fix |
|---|---|---|
| `spawn EFTYPE` on all agents | `SystemRoot` unset in Git Bash | Start via `.bat` or PowerShell |
| Exit code 2 on all agents | Newlines in prompts break `cmd.exe` | Apply quoteForCmd patch |
| `401 Missing Authentication header` | API keys in `adapterConfig.env` get wrapped | Put key in `~/.hermes/config.yaml` instead |
| Agents stuck in `running`, never execute | Budget overspend | PATCH `budgetMonthlyCents` higher + reset to `idle` |
| Port 3200 occupied, Paperclip falls back to 3201 | Stale node.exe from previous run | `taskkill /F /IM node.exe`, restart clean |

---

## Architecture Notes

**Correct separation of concerns:**
- **Paperclip** = task tracker + orchestrator (manages issues, routines, scheduling)
- **Hermes** = agent brain (does the actual reasoning and execution)

**Do NOT:**
- Run `npx paperclipai@latest run` — wipes patches every time
- Start from Git Bash — `SystemRoot` is unset
- Put API keys in `adapterConfig.env` — Paperclip wraps them and breaks auth
- Use `promptTemplate` for instructions — put everything in `AGENTS.md`

**DO:**
- Use the `.bat` launcher or PowerShell
- Put API keys in `~/.hermes/config.yaml` under `providers.openrouter.api_key`
- Set instructions in `~/.paperclip/instances/default/companies/<CO_ID>/agents/<AGENT_ID>/instructions/AGENTS.md`
