---
name: paperclip-debugging
description: "Systematic debugging for Paperclip AI + Hermes adapter on Windows — auth errors, model mismatches, log locks, budget halts, exit code 2."
version: 1.0.0
license: MIT
platforms: [windows]
tags: [paperclip, hermes, debugging, devops, windows, agents]
related_skills: [paperclip-setup, paperclip-restart]
---

# Paperclip + Hermes — Debugging Guide

## First-Run Setup

Before debugging, confirm your environment:

```bash
# Paperclip API responding?
curl -s http://127.0.0.1:3200/api/health

# Your Company ID (needed for most API calls)
curl -s http://127.0.0.1:3200/api/companies | python3 -c "
import json, sys
for c in json.load(sys.stdin):
    print(c['id'], c['name'])
"
```

Set a shell variable for convenience:
```bash
CO="<YOUR_COMPANY_ID>"
```

---

## 4-Line Bleed-Status Check (Run First on Every Return)

Before deep debugging, always verify current state:

```bash
echo "Paperclip API: $(netstat -ano 2>/dev/null | grep -E ':3200 ' | grep LISTEN | head -1 || echo 'DEAD')"
echo "Gateway:       $(netstat -ano 2>/dev/null | grep -E ':8642 ' | grep LISTEN | head -1 || echo 'DEAD')"
LATEST=$(ls -t ~/.paperclip/instances/default/logs/*.log 2>/dev/null | head -1)
echo "Latest log:    $(tail -1 "$LATEST" 2>/dev/null | head -c 200)"
```

If Paperclip API is DEAD, no LLM calls can happen. You have time to debug safely.

---

## Systematic Diagnosis Steps

### Step 1: List Agent Status

```bash
curl -s "http://127.0.0.1:3200/api/companies/${CO}/agents" | python3 -c "
import json, sys
agents = json.load(sys.stdin)
for a in agents:
    ac = a.get('adapterConfig', {})
    model = ac.get('model', '?')
    spent = a.get('spentMonthlyCents', 0) / 100
    budget = a.get('budgetMonthlyCents', 0) / 100
    print(f'{a[\"name\"]:<30} {a[\"status\"]:<10} model={model} spent=\${spent:.0f}/\${budget:.0f}')
"
```

Look for:
- `status: error` — agent failed
- `status: running` with no recent progress — possible budget halt
- Wrong model names

### Step 2: Check Heartbeat-Run Errors

```bash
curl -s "http://127.0.0.1:3200/api/companies/${CO}/heartbeat-runs?limit=10" | python3 -c "
import json, sys
for r in json.load(sys.stdin):
    err = r.get('error', '')
    if err:
        agent = r.get('agentName', '?')
        print(f'{agent}: {err[:200]}')
"
```

Common errors and what they mean — see the table below.

### Step 3: Check API Key Availability

```bash
curl -s "http://127.0.0.1:3200/api/companies/${CO}/agents" | python3 -c "
import json, sys
for a in json.load(sys.stdin):
    ac = a.get('adapterConfig', {})
    has_key = ac.get('apiKey') or 'OPENROUTER_API_KEY' in str(ac.get('env', {}))
    print(f'{a[\"name\"]:<30} apiKey={\"SET\" if has_key else \"NOT SET\"}')
"
```

If all show `NOT SET`, the adapter relies on `~/.hermes/config.yaml`.
Verify: `grep OPENROUTER_API_KEY ~/.hermes/config.yaml`

### Step 4: Check Server Log for System-Wide Errors

```bash
grep -E "error|fail|WARN" ~/.paperclip/instances/default/logs/server.log | tail -20
```

---

## Error Reference Table

| Error | Root Cause | Fix |
|---|---|---|
| `401 Missing Authentication header` | API key not reaching adapter | Put key in `~/.hermes/config.yaml` under `providers.openrouter.api_key`, NOT in `adapterConfig.env` |
| `spawn EFTYPE` / exit code 2 | `SystemRoot` unset (Git Bash) | Restart Paperclip via `.bat` or PowerShell |
| `quoteForCmd` multiline prompt / exit code 2 | Newlines in prompts break `cmd.exe` | Apply quoteForCmd newline sanitisation patch |
| `PermissionError: [WinError 32] agent.log` | Hermes gateway holds log file open, rotation fails | `hermes config set logging.max_size_mb 50 && hermes gateway restart` |
| `model not found` | Model ID stale or invalid | Verify against `curl https://openrouter.ai/api/v1/models` |
| Agent stuck in `running`, issue stays `todo` | `spentMonthlyCents > budgetMonthlyCents` | PATCH `budgetMonthlyCents` higher + reset agent to `idle` |
| Agent succeeds with zero output | `quiet: true` in adapterConfig | Set `quiet: false, verbose: true` via psycopg2 update |
| Issue stays `in_progress` after agent run | Disposition not completed | Check agent's AGENTS.md includes explicit `PATCH /api/issues/{ID}` with `status: done` |
| All agents fail simultaneously | Log lock (WinError 32) | `hermes gateway restart` |

---

## Critical: API Key in `adapterConfig.env` is BROKEN

Paperclip wraps any string value in `adapterConfig.env` into
`{type: "plain", value: "..."}`. The Hermes adapter passes this JSON object literally
as the env var value, breaking all auth.

**Symptom:** `401 Missing Authentication header` on every agent despite the key appearing set.

**Fix:** Put API keys in `~/.hermes/config.yaml`:
```yaml
providers:
  openrouter:
    api_key: sk-or-v1-...
```

Or use `adapterConfig.apiKey` field directly (not `adapterConfig.env`).

**Detection:** If agent env values look like `{type: plain, value: sk-...}` instead of
bare strings — they've been wrapped and broken.

---

## Critical: Budget Overspend Silent-Halt

An agent can be silently halted when `spentMonthlyCents > budgetMonthlyCents`. Signs:
- `status: "running"` (not error)
- `lastHeartbeatAt` is recent
- The assigned issue stays in `todo`

**Fix:**
```python
import psycopg2

conn = psycopg2.connect(host='127.0.0.1', port=54330, user='paperclip', dbname='paperclip')
cur = conn.cursor()

CO = '<YOUR_COMPANY_ID>'

# Raise budget ceiling
cur.execute("""
    UPDATE agents
    SET budget_monthly_cents = GREATEST(100000, spent_monthly_cents * 3),
        status = 'idle'
    WHERE company_id = %s
    AND spent_monthly_cents >= budget_monthly_cents
""", (CO,))
conn.commit()
print(f"Fixed {cur.rowcount} agents")
conn.close()
```

---

## Critical: `quiet: true` Suppresses All Agent Output

Agents appear to run and exit cleanly but no tool calls, no output visible.

**Fix:**
```python
import psycopg2, json

conn = psycopg2.connect(host='127.0.0.1', port=54330, user='paperclip', dbname='paperclip')
cur = conn.cursor()

CO = '<YOUR_COMPANY_ID>'

# Always GET full adapterConfig first, merge, then update
cur.execute("SELECT id, adapter_config FROM agents WHERE company_id = %s", (CO,))
for agent_id, ac in cur.fetchall():
    ac = ac or {}
    ac['quiet'] = False
    ac['verbose'] = True
    cur.execute("UPDATE agents SET adapter_config = %s WHERE id = %s",
                (json.dumps(ac), agent_id))
conn.commit()
conn.close()
```

---

## Critical: `promptTemplate` Pollution

**Symptom:** Agent heartbeat "succeeds" but the issue description contains raw system
prompt text, escape codes, or mission boilerplate instead of actual work output.

**Cause:** `adapterConfig.promptTemplate` was used to inject instructions.
The adapter prepends this verbatim to every task, and models echo it back.

**Fix:** Set `promptTemplate` to `""` for all agents. Instructions belong ONLY in
the agent's `AGENTS.md` file:
```
~/.paperclip/instances/default/companies/<CO_ID>/agents/<AGENT_ID>/instructions/AGENTS.md
```

**Detection:**
```bash
curl -s http://127.0.0.1:3200/api/companies/${CO}/agents | python3 -c "
import json, sys
for a in json.load(sys.stdin):
    pt = a.get('adapterConfig', {}).get('promptTemplate', '')
    if len(pt) > 10:
        print(f'{a[\"name\"]}: {len(pt)} chars in promptTemplate — should be 0')
"
```

---

## Agent Unstick Procedure

After fixing the underlying issue (auth, model, budget, log lock):

```bash
# Via REST API — reset to idle
curl -s -X PATCH "http://127.0.0.1:3200/api/agents/<AGENT_ID>" \
  -H "Content-Type: application/json" \
  -d '{"status": "idle"}'

# Via psycopg2 — batch reset all errored agents
# cur.execute("UPDATE agents SET status = 'idle' WHERE status = 'error' AND company_id = %s", (CO,))
```

---

## Discovering Valid API Routes

The Paperclip REST API has no route listing endpoint. When you get 404s:

1. **Check server logs for real traffic:**
   ```bash
   grep -E "GET|POST|PATCH" ~/.paperclip/instances/default/logs/server.log | tail -30
   ```

2. **Known working routes (v2026.517.0):**

| Purpose | Route |
|---|---|
| Health | `GET /api/health` |
| Agent list | `GET /api/companies/{CO_ID}/agents` |
| Agent status update | `PATCH /api/agents/{AGENT_ID}` |
| Agent wakeup | `POST /api/agents/{AGENT_ID}/wakeup` (NOT `/wake`) |
| Issue list | `GET /api/companies/{CO_ID}/issues` |
| Heartbeat runs | `GET /api/companies/{CO_ID}/heartbeat-runs` |

3. **`/api/agents` (no company ID) returns 404** — always scope to `/api/companies/{CO_ID}/agents`

---

## Isolation Test Pattern

Before assuming the problem is auth or model — prove it:

```bash
# Test Hermes CLI directly (as the adapter would call it)
export OPENROUTER_API_KEY="$(grep -A1 'openrouter' ~/.hermes/config.yaml | grep api_key | awk '{print $2}')"
echo "Say hello in 3 words." | hermes chat --provider openrouter --model <your-model> --yolo 2>&1
```

- If this works → adapter is the problem (check hermesCommand, args construction)
- If this fails → auth or model is the problem

**This saves 20+ minutes of log chasing.**

---

## PostgreSQL Direct Access

```python
import psycopg2

# Port is in ~/.paperclip/instances/default/db/postmaster.pid (line 4)
conn = psycopg2.connect(
    host='127.0.0.1',
    port=54330,        # check postmaster.pid for your actual port
    user='paperclip',
    dbname='paperclip'
    # no password — trust auth (see paperclip-setup skill)
)
```

**Only use direct Postgres for:** adapterConfig changes, budget resets, status bulk-updates.
**Don't use for:** things the REST API can do (`PATCH /api/agents/{id}` for status is fine).

---

## `hermes.cmd` vs `hermes.sh` as hermesCommand

| | `.cmd` wrapper | `.sh` wrapper |
|---|---|---|
| Works without patches | Yes | Requires `.sh` dispatch patch |
| Handles pipes/redirects | Cmd.exe quoting quirks | Bash handles cleanly |
| Current recommendation | Either, depends on your setup | If you have the `.sh` patch applied |

Use whichever your Hermes installation provides. Check `which hermes` / `where hermes`.
