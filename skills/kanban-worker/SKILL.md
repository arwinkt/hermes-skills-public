---
name: kanban-worker
description: "Pitfalls, examples, and edge cases for Hermes Kanban workers — good handoff shapes, retry diagnostics, block patterns."
version: 2.0.0
license: MIT
platforms: [linux, macos, windows]
tags: [kanban, multi-agent, collaboration, workflow, pitfalls]
related_skills: [kanban-orchestrator]
---

# Kanban Worker — Pitfalls and Examples

## First-Run Setup

No credentials required. Requires Hermes Agent with Kanban enabled.

```bash
hermes kanban list   # verify Kanban is available
```

---

## Overview

You're a Kanban worker. The Hermes dispatcher spawned you with a task. Your lifecycle:

1. **Orient** — `kanban_show` to read the task; check for prior runs
2. **Work** — do the task; send heartbeats for long jobs
3. **Block or Complete** — `kanban_block()` if you need human input; `kanban_complete()` when done

This skill is the deeper reference for edge cases and patterns.

---

## Workspace Handling

Your workspace kind determines how to work inside `$HERMES_KANBAN_WORKSPACE`:

| Kind | What it is | How to work |
|---|---|---|
| `scratch` | Fresh tmp dir, yours alone | Read/write freely; GC'd on archive |
| `dir:<path>` | Shared persistent directory | Treat like long-lived state |
| `worktree` | Git worktree at resolved path | Commit work here |

---

## Good Summary + Metadata Shapes

The `kanban_complete(summary=..., metadata=...)` handoff is how downstream workers read what you did.

**Coding task:**
```python
kanban_complete(
    summary="shipped rate limiter — token bucket, keys on user_id with IP fallback, 14 tests pass",
    metadata={
        "changed_files": ["rate_limiter.py", "tests/test_rate_limiter.py"],
        "tests_run": 14,
        "tests_passed": 14,
        "decisions": ["user_id primary, IP fallback for unauthenticated requests"],
    },
)
```

**Coding task needing human review (use `block` not `complete`):**
```python
import json

kanban_comment(
    body="review-required handoff:\n" + json.dumps({
        "changed_files": ["rate_limiter.py", "tests/test_rate_limiter.py"],
        "tests_run": 14,
        "tests_passed": 14,
        "decisions": ["user_id primary, IP fallback for unauthenticated requests"],
    }, indent=2),
)
kanban_block(
    reason="review-required: rate limiter shipped, 14/14 tests pass — needs eyes on user_id/IP fallback choice before merging",
)
```

Use `kanban_complete` only when genuinely terminal (typo fix, docs change, research writeup).
For code changes, block with `review-required:` prefix so the dashboard surfaces the row.

**Research task:**
```python
kanban_complete(
    summary="3 libraries reviewed; vLLM wins on throughput, SGLang on latency",
    metadata={
        "sources_read": 12,
        "recommendation": "vLLM",
        "benchmarks": {"vllm": 1.0, "sglang": 0.87},
    },
)
```

**Review task:**
```python
kanban_complete(
    summary="reviewed PR #123; 2 blocking issues found",
    metadata={
        "pr_number": 123,
        "findings": [
            {"severity": "critical", "file": "api/search.py", "line": 42, "issue": "raw SQL concat"},
            {"severity": "high", "file": "api/settings.py", "issue": "missing CSRF middleware"},
        ],
        "approved": False,
    },
)
```

---

## Claiming Cards You Created

If your run produced new kanban tasks, pass the ids in `created_cards` on `kanban_complete`.
The kernel verifies each id exists and was created by your profile.

```python
# GOOD — capture return values, then claim them
c1 = kanban_create(title="remediate SQL injection", assignee="security-worker")
c2 = kanban_create(title="fix CSRF middleware", assignee="web-worker")

kanban_complete(
    summary="Review done; spawned remediations for both findings.",
    metadata={"pr_number": 123, "approved": False},
    created_cards=[c1["task_id"], c2["task_id"]],
)
```

```python
# BAD — claiming ids you don't have captured return values for
kanban_complete(
    summary="Created remediation cards t_a1b2c3d4, t_deadbeef",  # hallucinated
    created_cards=["t_a1b2c3d4", "t_deadbeef"],                   # gate rejects this
)
```

**Only list ids from a successful `kanban_create` return value.** Never invent ids.

---

## Block Reasons That Get Answered Fast

**Bad:** `"stuck"` — no context for the human.

**Good:** one sentence naming the specific decision you need. Leave context as a comment.

```python
kanban_comment(
    task_id=os.environ["HERMES_KANBAN_TASK"],
    body="Context: users are behind NATs with thousands of peers. Keying on IP alone causes false positives.",
)
kanban_block(reason="Rate limit key: IP (simple, NAT-unsafe) or user_id (requires auth)?")
```

The block message appears in the dashboard. The comment is the deeper context.

---

## Heartbeats Worth Sending

**Good:** `"epoch 12/50, loss 0.31"`, `"scanned 1.2M/2.4M rows"`, `"uploaded 47/120 videos"`

**Bad:** `"still working"`, empty notes, sub-second intervals.

Send every few minutes for long tasks; skip entirely for tasks under ~2 minutes.

---

## Retry Scenarios

If `kanban_show` returns `runs: [...]` with closed prior runs, you're a retry.
Read the prior runs' `outcome` / `summary` / `error`. Don't repeat that path.

| Prior outcome | What it means |
|---|---|
| `timed_out` | Hit `max_runtime_seconds`. Chunk the work or shorten it. |
| `crashed` | OOM or segfault. Reduce memory footprint. |
| `spawn_failed` | Profile config issue (missing credential, bad PATH). Block with explanation. |
| `blocked` | Previous attempt blocked; check thread for the unblock comment. |

---

## Do NOT

- Call `delegate_task` as a substitute for `kanban_create`. `delegate_task` is for short reasoning subtasks inside YOUR run; `kanban_create` is for cross-agent handoffs.
- Modify files outside `$HERMES_KANBAN_WORKSPACE` unless the task body says to.
- Create follow-up tasks assigned to yourself — assign to the right specialist.
- Complete a task you didn't actually finish. Block it instead.

---

## Pitfalls

**Task state can change between dispatch and your startup.** Always `kanban_show` first.
If it reports `blocked` or `archived`, stop — you shouldn't be running.

**Workspace may have stale artefacts.** Especially `dir:` and `worktree` workspaces can
have files from previous runs. Read the comment thread — it usually explains why you're
running again.

**Don't rely on the CLI when the kanban tools are available.** The `kanban_*` tools work
across all terminal backends. `hermes kanban <verb>` from a terminal tool may fail in
containerised backends.

---

## CLI Fallback (for scripting)

```bash
hermes kanban show <id> --json
hermes kanban complete <id> --summary "..." --metadata '{...}'
hermes kanban block <id> "reason"
hermes kanban create "title" --assignee <profile> [--parent <id>]
```

Use the tools from inside an agent; the CLI exists for humans at the terminal.
