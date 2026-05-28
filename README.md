# hermes-skills-public

A collection of battle-tested skills for [Hermes Agent](https://hermes-agent.nousresearch.com) — built in production, open-sourced for the community.

---

## Why I Made This

I'm a 22-year-old startup founder building products and working on [Paperclip AI](https://github.com/paperclipai/paperclip) — an open-source AI agent orchestration platform with 30k+ GitHub stars.

Over the past few months I've been building out a personal agentic OS on top of Hermes Agent:
multi-agent Paperclip stacks running 12 autonomous agents, an Obsidian second brain with slash
commands, subagent-driven software development workflows, and a bunch of debugging methodology
that saved me hours of thrashing.

These skills are what I actually use every day. They're not toy examples — they're the
exact workflows that power how I build, debug, and think.

I'm open-sourcing them because the Hermes community has been excellent and I think
these patterns are useful beyond just my setup.

---

## What's in Here

### Development Workflow Skills

| Skill | What It Does | Who It's For |
|---|---|---|
| [`systematic-debugging`](skills/systematic-debugging/) | 4-phase root cause methodology — understand bugs before fixing | Any developer |
| [`writing-plans`](skills/writing-plans/) | Write implementation plans with bite-sized tasks, exact paths, complete code | Anyone building features |
| [`subagent-driven-development`](skills/subagent-driven-development/) | Execute plans via delegate_task subagents with 2-stage review (spec + quality) | Hermes power users |
| [`test-driven-development`](skills/test-driven-development/) | Strict RED-GREEN-REFACTOR cycle — no production code without a failing test first | Any developer |
| [`requesting-code-review`](skills/requesting-code-review/) | Pre-commit verification pipeline: static security scan, baseline tests, independent reviewer subagent, auto-fix loop | Any developer |
| [`spike`](skills/spike/) | Throwaway experiments to validate an idea before committing to a real build | Anyone evaluating approaches |

### Multi-Agent / Kanban Skills

| Skill | What It Does | Who It's For |
|---|---|---|
| [`kanban-orchestrator`](skills/kanban-orchestrator/) | Decompose multi-agent work into Kanban tasks — route, don't execute | Orchestrator profiles |
| [`kanban-worker`](skills/kanban-worker/) | Pitfalls, handoff shapes, retry diagnostics for Kanban workers | Worker profiles |

### Personal Knowledge Management

| Skill | What It Does | Who It's For |
|---|---|---|
| [`second-brain`](skills/second-brain/) | 22 Obsidian slash commands — daily planning, capture, reflection, brainstorm | Obsidian + Hermes users |

### Paperclip AI on Windows

| Skill | What It Does | Who It's For |
|---|---|---|
| [`paperclip-setup`](skills/paperclip-setup/) | Full Windows setup guide — local npm install, PostgreSQL trust auth, agent config | Paperclip self-hosters |
| [`paperclip-debugging`](skills/paperclip-debugging/) | Systematic debugging — auth errors, model mismatches, log locks, budget halts | Paperclip users |
| [`paperclip-restart`](skills/paperclip-restart/) | Safe restart procedure — zombie instances, PG recovery, SystemRoot requirement | Paperclip on Windows |

---

## How to Use These Skills

### Option 1: Install Individually

Copy any `SKILL.md` into your Hermes skills directory:

```bash
# Example — install systematic-debugging
mkdir -p ~/.hermes-personal/skills/software-development/systematic-debugging/
cp skills/systematic-debugging/SKILL.md ~/.hermes-personal/skills/software-development/systematic-debugging/
```

Then load it in a Hermes session:
```
skill_view(name="systematic-debugging")
```

### Option 2: Clone the Whole Repo

```bash
git clone https://github.com/arwinkt/hermes-skills-public
cd hermes-skills-public

# Copy all skills to your Hermes skills directory
cp -r skills/* ~/.hermes-personal/skills/
```

### Option 3: Use as a Reference

Even if you don't use Hermes Agent, the methodology skills
(`systematic-debugging`, `writing-plans`) work as standalone process documents
you can follow manually or adapt to any AI assistant.

---

## Quick Start by Use Case

**"I want to debug bugs more systematically"**
→ Install `systematic-debugging`. Read Phase 1 carefully. The Iron Law is real.

**"I want Hermes to implement features for me with quality guarantees"**
→ Install `writing-plans` + `subagent-driven-development`. Use them in sequence.

**"I want to enforce TDD discipline on every feature"**
→ Install `test-driven-development`. The Red Flags section is worth reading before you start.

**"I want automated code review before every commit"**
→ Install `requesting-code-review`. It runs a static scan + independent reviewer subagent + auto-fix loop.

**"I want to validate an idea before spending days building it"**
→ Install `spike`. Build a throwaway experiment first, get a VALIDATED/INVALIDATED verdict, then commit to the real build.

**"I want an Obsidian-connected second brain with slash commands"**
→ Install `second-brain`. Update `<YOUR_VAULT_PATH>` to your vault. Type `/today`.

**"I'm running Paperclip AI on Windows and things keep breaking"**
→ Install `paperclip-setup` first, then `paperclip-debugging`. The `SystemRoot` and
`adapterConfig.env` wrapping issues in the debugging skill will save you hours.

**"I want to run multi-agent workflows in Hermes"**
→ Install `kanban-orchestrator` + `kanban-worker`. Run `hermes kanban list` to verify
Kanban is enabled.

---

## Also Published: QUT Student Toolkit

If you're a QUT student, I also maintain a separate repo for automating Canvas and
Outlook: [arwinkt/qut-student-toolkit](https://github.com/arwinkt/qut-student-toolkit)

---

## Notes

- All skills include a **First-Run Setup** section that tells you exactly what credentials
  or config you need before the skill will work correctly.
- The Paperclip skills are Windows-specific — they document the actual pain points of
  running Paperclip + Hermes on Windows with Git Bash and Node.js v24.
- These skills evolve as I use them. Check back for updates.

---

## Contributing

Found a bug in a skill, or have a pitfall to add? Open an issue or PR.
The skills are markdown — easy to contribute to.

---

## License

MIT — use freely, attribution appreciated but not required.
