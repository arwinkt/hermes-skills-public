---
name: kanban-orchestrator
description: "Decompose multi-agent work into Kanban tasks — route, don't execute. Anti-temptation rules and fan-out playbook."
version: 3.0.0
license: MIT
platforms: [linux, macos, windows]
tags: [kanban, multi-agent, orchestration, routing]
related_skills: [kanban-worker]
---

# Kanban Orchestrator — Decomposition Playbook

## First-Run Setup

No credentials required. Requires Hermes Agent with Kanban enabled.

**Verify Kanban is available:**
```bash
hermes kanban list   # should return an empty list or existing tasks
```

If Kanban is not enabled, see the Hermes docs for enabling the Kanban toolset.

---

## What This Skill Is For

You're an orchestrator. Your job is to:
1. **Discover** what profiles (workers) exist on this machine
2. **Decompose** the user's goal into concrete cards
3. **Route** each card to the right profile
4. **Report back** to the user with what you created

**You do NOT execute the work yourself.**

---

## Step 0: Discover Available Profiles (Before Planning)

The dispatcher silently fails for unknown assignee names — a card assigned to a
profile that doesn't exist just sits in `ready` forever.

**Always run one of these before creating any cards:**

```bash
hermes profile list   # table of all profiles on this machine
```

Or ask the user: "What profiles do you have set up?"

Cache the result. Re-asking every turn wastes tool calls.

---

## When to Use the Board (vs. Just Doing the Work)

Create Kanban tasks when any of these are true:

1. **Multiple specialists needed.** Research + analysis + writing is three profiles.
2. **Work should survive a crash or restart.** Long-running or important tasks.
3. **User might want to interject.** Human-in-the-loop at any step.
4. **Multiple subtasks can run in parallel.** Fan-out for speed.
5. **Review / iteration is expected.** A reviewer profile loops on drafter output.
6. **Audit trail matters.** Board rows persist in SQLite forever.

If none of those apply — small one-shot reasoning task — use `delegate_task` instead.

---

## The Anti-Temptation Rules

- **Do not execute the work yourself.** Create a task for the right specialist.
- **For any concrete task, create a Kanban card and assign it.** Every time.
- **Split multi-lane requests before creating cards.** "Also" and "and" often mean two separate cards.
- **Run independent lanes in parallel.** Don't link tasks that don't need each other.
- **Never create dependent work as independent ready cards.** Use `parents=[...]` on create.
- **If no specialist fits available profiles, ask.** Don't invent profile names.

---

## Decomposition Playbook

### Step 1 — Understand the Goal

Ask clarifying questions if the goal is ambiguous.
Cheap to ask; expensive to spawn the wrong fleet.

### Step 2 — Sketch the Task Graph

Before creating anything, draft the graph out loud. For each lane:

1. Extract workstreams from the request
2. Map each lane to one of the profiles you discovered in Step 0
3. Decide: is each lane independent or gated by another?
4. Create independent lanes as parallel cards with no parent links
5. Create synthesis/review cards with parent links to the lanes they depend on

**Examples (use your actual profile names — these are placeholders):**

- "Build an app" → `<design-profile>` for product/UI + `<engineer-profile>` for implementation + `<reviewer-profile>` after both
- "Fix blockers and check model variants" → `<engineer>` for fixes + `<researcher>` for variant check, separately
- "Research docs and implement" → research + codebase-discovery can run in parallel; implementation waits only if it truly needs those findings

Words like "also", "finally", "and" do NOT automatically imply a dependency.

**Show the graph to the user before creating cards.** Let them correct it.

### Step 3 — Create Tasks and Link

```python
t1 = kanban_create(
    title="research: Postgres cost vs current",
    assignee="<profile-A>",   # replace with actual profile name
    body="Compare estimated infrastructure costs over a 3-year window.",
)["task_id"]

t2 = kanban_create(
    title="research: Postgres performance vs current",
    assignee="<profile-A>",   # same profile, run in parallel
    body="Compare query latency and throughput at expected data volume.",
)["task_id"]

t3 = kanban_create(
    title="synthesise migration recommendation",
    assignee="<profile-B>",
    body="Read T1 and T2 findings. Produce a 1-page recommendation with trade-offs.",
    parents=[t1, t2],         # stays in todo until both parents are done
)["task_id"]
```

`parents=[...]` gates promotion — children auto-promote to `ready` when all parents reach `done`.

### Step 4 — Mark Your Own Task Done (If Applicable)

If you were spawned as a task yourself (e.g., a planner card):

```python
kanban_complete(
    summary="decomposed into T1-T4: 2 research lanes in parallel, 1 synthesis, 1 prose draft",
    metadata={
        "task_graph": {
            "T1": {"assignee": "<profile-A>", "parents": []},
            "T2": {"assignee": "<profile-A>", "parents": []},
            "T3": {"assignee": "<profile-B>", "parents": ["T1", "T2"]},
        },
    },
)
```

### Step 5 — Report Back

Tell the user what you created in plain prose, using the actual profile names.

---

## Common Patterns

**Fan-out + fan-in (research → synthesise):** N research cards with no parents, one synthesis card with all of them as parents.

**Parallel implementation + validation:** implementer card + verifier card in parallel; reviewer card depends on both.

**Pipeline with gates:** `planner → implementer → reviewer`. Each stage gated on the previous.

**Human-in-the-loop:** Any task can `kanban_block()` to wait for input. The comment thread carries context.

---

## Pitfalls

**Inventing profile names.** The dispatcher silently drops unknown assignees. Always assign to a profile from Step 0 discovery.

**Bundling independent lanes into one card.** "Fix blockers and check model variants" is two cards.

**Over-linking because of wording.** "Finally check X" may still be parallel if X is static config or docs.

**Forgetting dependency links.** If the graph says `research → implement → review`, use parent links.

**Don't pre-create the whole graph if shape depends on intermediate findings.** Let synthesis tasks decide their own sub-graph after reading parent outputs.

---

## Recovering Stuck Workers

When a worker keeps crashing or getting blocked:

1. **Reclaim:** `hermes kanban reclaim <task_id>` — abort the running worker, reset to `ready`
2. **Reassign:** `hermes kanban reassign <task_id> <new-profile> --reclaim` — switch to a different profile
3. **Change model:** `hermes -p <profile> model` — edit profile config, then Reclaim to retry
