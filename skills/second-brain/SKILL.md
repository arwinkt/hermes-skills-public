---
name: second-brain
description: "Obsidian-connected slash commands for daily planning, session capture, reflections, and knowledge management."
version: 1.0.0
license: MIT
platforms: [linux, macos, windows]
tags: [obsidian, second-brain, pkm, daily-notes, planning, capture]
---

# Second Brain Slash Commands

## First-Run Setup

### What You Need

1. **Obsidian** — or any markdown vault folder on your filesystem
2. **Hermes Agent** with file access to your vault

### Configuration

Set your vault root. Replace `<YOUR_VAULT_PATH>` throughout this skill with
the absolute path to your vault folder.

**Examples:**
- Windows: `C:\Users\<username>\second_brain`
- macOS/Linux: `/home/<username>/second_brain` or `~/second_brain`

### Vault Structure (Required)

Create these folders inside your vault root (or update the paths in each command below
to match your existing structure):

```
<YOUR_VAULT_PATH>/
  daily/       — daily logs (YYYY-MM-DD.md)
  now/         — active context (pending_actions.md, active projects)
  core/        — identity, goals, ideal day
  business/    — active ventures and projects
  thinking/    — reflections, processing
  knowledge/   — compiled knowledge articles
  clippings/   — saved web content
```

You can rename any folder — just update the paths in the command definitions below.

---

## How to Invoke

Type a slash command in your Hermes chat. Hermes detects the `/commandname` pattern
and executes the corresponding workflow.

These are **skill-level triggers**, not registered Hermes CLI slash commands.
They work when typed — but won't appear in the `/` autocomplete dropdown.

---

## The Commands

### /today

**What it reads:** today's daily note + last 3 daily notes + `<VAULT>/now/pending_actions.md` + scan `<VAULT>/now/`

**Output:**
1. The 3 things that actually matter today
2. Anything urgent or overdue
3. One thing being avoided that should be faced
4. Energy read: deep focus / admin / recovery

---

### /capture

**What it does:** Review the conversation so far. Extract decisions made, things learned,
ideas worth keeping, unfinished threads. Append to today's daily note.

**Format appended:**
```
### Session (HH:MM) — [brief title]
**Context:** what we were working on
**Key decisions:**
-
**What I learned:**
-
**Ideas that came up:**
-
**Unfinished / carry forward:**
-
```

---

### /closeday

**What it reads:** today's daily note + recent conversation.

**Output:** progress across active projects, new ideas, carry-forwards, anything to update
in project notes. Appends end-of-day summary to daily note.

---

### /superpowers [task]

**Planning mode — use before any build.**

**Output:**
1. What needs to happen, broken into concrete steps
2. Order and reasoning
3. Likely failure points and unknowns
4. Prerequisites (files, keys, dependencies, decisions)
5. Cleanest approach given current stack
6. What NOT to do

Asks for confirmation before proceeding. Never starts executing until confirmed.

---

### /gsd [task/plan]

**Execution mode.**

Read the plan (from `/superpowers` or user description). Rules: stay on steps,
note tangents without chasing them, flag real blockers vs rabbit holes. Start with step 1, go.

For context reload mid-session: provide task, completed steps, current step, open blockers.

---

### /ultrareview

**Deep debrief (use ~3×/month).**

**Output:**
1. What was actually completed (enough detail to cold-start)
2. Every decision and its reasoning
3. What is fragile, untested, or held by assumptions
4. What future-me needs to know to continue safely
5. What vault notes need creating or updating
6. Patterns connecting to bigger things
7. What next session starts with

Then: write detailed daily log entry, list every vault note needing updates.

---

### /reflect

**What it reads:** recent daily notes + `<VAULT>/core/`

**Mode:** asks one question at a time. Starts with: "What's taking up the most mental space
right now that you haven't fully acknowledged?"

For deep reflection: address the gap between stated identity and actual behaviour,
what's being avoided, what you'd tell yourself in 12 months.

---

### /brainstorm

**What it reads:** entire vault — `business/`, `core/`, `thinking/`, recent clippings.

**Generates ideas across:** active project next moves, connections between unrelated things,
things you haven't considered.

Grounds everything in actual vault content. Pushes into territory not yet visited.

---

### /review [weekly/monthly/quarterly]

**Weekly:** read last 7 days of daily notes + `now/pending_actions`.

**Output:** what moved, what didn't (honest), carry-overs, top 3 priorities next week,
thing you keep avoiding.

**Monthly/quarterly:** expand scope to 30 days / full vault.

---

### /challenge [topic]

**What it reads:** vault notes on the topic.

**Output:** contradictions, wrong assumptions, pushback using your own notes.
Hurts sometimes — that's the point.

---

### /schedule

**What it reads:** current projects, goals, priorities in vault.

**Output:** suggested weekly time blocks, conflicts between stated priorities and
actual time use. Best run weekly (e.g., Sunday night).

---

### /clip [url or content]

**Saves to:** `<VAULT>/clippings/YYYY-MM-DD-title.md`

**Format:** frontmatter (title, url, date, tag, status: unread), personal note, key points extracted.

**Tags to use:** based on your active projects and interests.

---

### /compile

**Trigger:** runs your clippings or session log compilation scripts.

Adapt to your vault's compile scripts if you have them. Otherwise, use `/capture` instead.

---

### /graduate

**What it reads:** daily notes from last 14 days.

**Finds:** recurring or underdeveloped ideas.

**Creates:** standalone files in the vault with core claim, context, and links to related notes.

---

### /drift

**What it reads:** entire vault.

**Finds:**
- Topics recurring across unrelated notes (subconscious circling)
- Stated wants not being acted on
- What you're moving toward without committing
- Beliefs showing up in how you write

Specific — uses actual quotes as evidence.

---

### /emerge

**What it reads:** `knowledge/`, `core/`, last 30 days of daily notes.

**Finds:**
- Idea clusters at critical mass
- Next moves the vault is pointing to
- Research sitting unacted on

For each finding: what is it, what evidence supports it, exact first step.

---

### /ideas

**What it reads:** whole vault.

**Generates:** tools to build, people to reach out to, topics to investigate,
things to create or write, project moves. Grounded in actual vault patterns.

---

### /trace [topic]

**What it reads:** entire vault, filtered by topic.

**Output:** first appearance, evolution over time, current connections. Specific notes and dates.

---

### /connect [topic A] [topic B]

**What it reads:** vault filtered for both topics.

**Output:** notes that link them, patterns between them.

---

### /ghost [question]

**What it reads:** identity, beliefs, how-you-think notes.

**Output:** answers the question in your voice, referencing specific notes.

Use for: drafts, comms, content that should sound like you.

---

### /context

**What it reads:** last 7 days of daily notes + project files.

**Output:** active projects, recent reflections, priorities from last 7 days.

---

## Pitfalls

### Reliable File Appending

When appending content to daily notes, `write_file` can be inconsistent.
**Preferred approach:** use the terminal tool with `printf` + shell redirection:

```bash
# Git Bash / macOS / Linux
printf "\n---\nNew content to append" >> "<VAULT>/daily/2026-05-28.md"
```

### First Confirm the File Exists

```python
# 1. Check if today's note exists
read_file("<VAULT>/daily/YYYY-MM-DD.md")

# 2. If it doesn't, create it with a header first
write_file("<VAULT>/daily/YYYY-MM-DD.md", "# YYYY-MM-DD\n\n")

# 3. Then append via terminal
```

---

## Customisation

You can add, remove, or rename any command. The commands are defined in this skill file —
edit them to match your vault structure and workflow.

Common customisations:
- Change folder names to match your existing vault
- Add project-specific commands (e.g., `/startup`, `/endofweek`)
- Adjust output formats to match your note style
