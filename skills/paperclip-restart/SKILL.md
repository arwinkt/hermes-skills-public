---
name: paperclip-restart
description: "Safe stop and restart procedure for Paperclip AI on Windows — handle stale processes, PostgreSQL recovery, and the SystemRoot requirement."
version: 1.0.0
license: MIT
platforms: [windows]
tags: [paperclip, hermes, devops, windows, restart, recovery]
related_skills: [paperclip-setup, paperclip-debugging]
---

# Paperclip AI — Windows Restart Procedure

## First-Run Setup

No additional credentials needed. You need:
- Paperclip installed as a local npm project (see `paperclip-setup` skill)
- The `start-paperclip.bat` launcher file
- Git Bash or PowerShell

**Verify your install path:**
```bash
ls ~/.paperclip-stack/install/     # should have node_modules/, server.js, etc.
ls ~/.paperclip/instances/default/ # should have config.json, db/, logs/
```

---

## Quick Restart (Normal Case)

**Option A: Double-click the .bat file from Windows Explorer**

1. Navigate to your install directory in File Explorer
2. Double-click `start-paperclip.bat`
3. A CMD window opens — Paperclip starts
4. Health check: `curl -s http://127.0.0.1:3200/api/health`

**Option B: PowerShell**
```powershell
Start-Process -FilePath "C:\path\to\start-paperclip.bat"
```

**Never start from Git Bash.** `SystemRoot` is unset in Git Bash, causing `spawn EFTYPE`
errors on every agent subprocess.

---

## Kill + Restart (Server Stuck or Crashed)

```powershell
# From PowerShell:
taskkill /F /IM node.exe

# Wait 3 seconds for process to fully exit
Start-Sleep -Seconds 3

# Restart
Start-Process -FilePath "C:\path\to\start-paperclip.bat"
```

**Check for multiple zombie instances first:**
```powershell
netstat -ano | findstr "LISTENING" | findstr ":3200 \|:3201 \|:3202 "
```

If you see multiple ports listening with node.exe — you have zombies. Kill all:
```powershell
taskkill /F /IM node.exe
```
Then restart once cleanly.

---

## Verify Startup

```bash
# From Git Bash after restart:
sleep 5  # give Paperclip ~25s to fully boot

# Health check
curl -s http://127.0.0.1:3200/api/health | python3 -c "
import json, sys
d = json.load(sys.stdin)
print(d.get('status'), d.get('version'))
"
# Expected: ok  2026.517.0
```

---

## Does Paperclip Survive Terminal Closure?

**No.** Paperclip runs as a foreground process in the CMD window. If that window is
closed, Paperclip exits.

**Recovery options:**

1. **Manual restart** — double-click the .bat file
2. **Watchdog** — a Python script that polls the health endpoint every 5 minutes and
   relaunches the .bat file if Paperclip is down:

```python
# ~/.hermes/scripts/paperclip-watchdog.py
import time, subprocess, urllib.request

HEALTH_URL = "http://127.0.0.1:3200/api/health"
BAT_PATH = r"C:\path\to\start-paperclip.bat"
POLL_INTERVAL = 300  # 5 minutes

while True:
    try:
        urllib.request.urlopen(HEALTH_URL, timeout=5)
    except Exception:
        print("Paperclip down — relaunching...")
        subprocess.Popen(
            [BAT_PATH],
            creationflags=0x00000010,  # CREATE_NEW_CONSOLE
            close_fds=True
        )
    time.sleep(POLL_INTERVAL)
```

Register as a Hermes cron job (no-agent mode, runs the script directly):
```
hermes cron create --name "Paperclip Watchdog" --schedule "*/5 * * * *" \
  --script paperclip-watchdog.py --no-agent
```

---

## Silent Port Fallback (3200 → 3201)

Paperclip silently falls back to port 3201 if 3200 is occupied — even when `config.json`
explicitly sets port 3200. The startup log says:

```
Server 3201 (requested 3200)
```

**Detection:**
```bash
netstat -ano 2>/dev/null | grep -E ':(3200|3201|3202) '
```

**Fix:** Kill the process occupying 3200, then restart:
```powershell
# Find what's on port 3200
netstat -ano | findstr ":3200 "
# Kill it (replace 1234 with actual PID):
taskkill /F /PID 1234
# Restart Paperclip
```

---

## PostgreSQL Recovery (Shared Memory / Interrupted Recovery)

If Paperclip fails with `pre-existing shared memory block is still in use`
or `database system was interrupted while in recovery`:

### Step 1: Find the embedded Postgres binary

```bash
find ~/.paperclip-stack/install/node_modules -name "pg_ctl.exe" 2>/dev/null
```

Typical path: `~/.paperclip-stack/install/node_modules/@embedded-postgres/windows-x64/native/bin/pg_ctl.exe`

### Step 2: Stop the stale PG instance

```python
import subprocess

pg_ctl = r"C:\path\to\pg_ctl.exe"
db_path = r"C:\Users\<YOUR_USERNAME>\.paperclip\instances\default\db"

p = subprocess.Popen(
    [pg_ctl, 'stop', '-D', db_path, '-m', 'immediate'],
    stdout=subprocess.PIPE, stderr=subprocess.PIPE,
    creationflags=0x08000000  # CREATE_NO_WINDOW
)
out, err = p.communicate(timeout=15)
print(out.decode(), err.decode())
```

### Step 3: Start PG manually as a detached process

```python
import subprocess

postgres = r"C:\path\to\postgres.exe"
db_path = r"C:\Users\<YOUR_USERNAME>\.paperclip\instances\default\db"

p = subprocess.Popen(
    [postgres, '-D', db_path, '-p', '54330'],
    stdout=open('pg_manual.log', 'a'),
    stderr=subprocess.STDOUT,
    creationflags=0x00000008 | 0x00000200,  # DETACHED_PROCESS | CREATE_NEW_PROCESS_GROUP
    close_fds=True
)
print(f"PG started with PID {p.pid}")
```

Wait for port 54330 to open (1–5s for a clean DB, up to 120s if recovery needed):
```python
import time, socket

def wait_for_pg(port=54330, timeout=120):
    for i in range(timeout):
        try:
            s = socket.create_connection(('127.0.0.1', port), timeout=1)
            s.close()
            print(f"PG ready after {i}s")
            return True
        except OSError:
            time.sleep(1)
    return False

wait_for_pg()
```

### Step 4: Start Paperclip

Only after port 54330 is confirmed open, start Paperclip via the `.bat` file.

---

## What NOT To Do

| Don't | Why |
|---|---|
| Start from Git Bash | `SystemRoot` unset → `spawn EFTYPE` on all agents |
| Use `npx paperclipai run` | Requires TTY; crashes with "stdin is not a tty" in background |
| Use `npx paperclipai@latest run` | Pulls fresh packages, wipes patch-package patches |
| Kill node.exe from Git Bash while planning to restart there | Can't cleanly restart from Git Bash |
| Trust the Web UI for agent status | UI may poll a different Company ID than your actual agents |

---

## `start-paperclip.bat` Template

```batch
@echo off
REM Critical: SystemRoot must be set for Paperclip to spawn agent subprocesses
SET SYSTEMROOT=C:\WINDOWS

REM Your Paperclip instance directory
SET PAPERCLIP_INSTANCE_DIR=C:\Users\<YOUR_USERNAME>\.paperclip\instances\default

REM Auto-apply pending DB migrations on startup
SET PAPERCLIP_MIGRATION_AUTO_APPLY=true

REM Move to the install directory and start the server
cd /d C:\Users\<YOUR_USERNAME>\.paperclip-stack\install
node server.js
```
