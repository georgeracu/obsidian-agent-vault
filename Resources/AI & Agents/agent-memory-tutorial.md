---
title: How to Use the Agent Memory System
date: 2026-04-10
tags:
  - agent-memory
  - tutorial
  - ai-agents
aliases:
  - Agent Memory Tutorial
  - Memory Agent Guide
---

# How to Use the Agent Memory System

This tutorial walks through the agent memory protocol for anyone — human or AI — working in this vault. The full protocol reference lives in [[AGENTS]].

> [!info] What this system does
> Any AI agent (Claude, Gemini, Pi, etc.) that reads [[AGENTS]] at the start of a session can load prior context, write new memories, take a snapshot before stopping, and flag outdated entries for pruning — all using standard vault tooling.

---

## 1. Session Start

Before doing any work, an agent searches the memory store for relevant context.

**Search by topic using `qmd`:**

```bash
qmd search "topic or task description" -c <your-collection-name>
```

**Read a specific entry using `obsidian`:**

```bash
obsidian read file="2026-04-10-vault-structure"
```

**Check for a recent snapshot if resuming prior work:**

```bash
obsidian list path="agent-memory/snapshots"
```

> [!tip] What to look for
> Scan for entries of type `context` (active state), `decision` (choices to respect), and `mistake` (pitfalls to avoid). If there's a recent `snapshot`, read it first — it's the most complete picture of where things left off.

---

## 2. Writing Memory Entries

Write a new entry whenever you discover something worth preserving across sessions. Use `obsidian` to create the file — never raw file I/O.

### Entry types at a glance

| Type | When to write | Location |
|------|--------------|----------|
| `context` | Active project state, working assumptions | `agent-memory/context/` |
| `decision` | A choice made that future agents should respect | `agent-memory/decisions/` |
| `mistake` | A pitfall you hit — what failed and why | `agent-memory/mistakes/` |
| `pattern` | A reusable approach that worked well | `agent-memory/patterns/` |
| `snapshot` | End of session or before a context switch | `agent-memory/snapshots/` |

### Creating an entry

```bash
obsidian create \
  path="agent-memory/decisions/2026-04-10-obsidian-cli-only.md" \
  content="---\ntype: decision\ndate: 2026-04-10\ntags: [agent-memory, decision]\nstatus: active\nsummary: Always use obsidian cli for file I/O, never raw Bash\n---\n\n## Decision\n\nAll agents must use obsidian cli to create, read, and update memory entries.\n\n## Why\n\nRaw file writes bypass Obsidian's link tracking. obsidian cli keeps wikilinks and backlinks consistent across the vault.\n\n## Applies to\n\nAny agent creating, editing, or updating files in agent-memory/." \
  silent
```

> [!warning] Required frontmatter
> Every entry must have all five fields: `type`, `date`, `tags`, `status`, and `summary`. The `summary` field is what appears in the [[memory.base|memory.base]] Bases view — keep it concise and searchable.

**Filename format:** `YYYY-MM-DD-<short-slug>.md`

---

## 3. Entry Templates

### Context entry

```markdown
---
type: context
date: 2026-04-10
tags: [agent-memory, context]
status: active
summary: <one-line description>
---

## Current State

<What is true right now — active assumptions, in-progress work, vault state>

## Relevant Files

- [[Note Name]] — why it matters
```

### Decision entry

```markdown
---
type: decision
date: 2026-04-10
tags: [agent-memory, decision]
status: active
summary: <the decision in one line>
---

## Decision

<What was decided>

## Why

<The reasoning — what alternatives were considered, why this won>

## Applies to

<Scope — when should future agents follow this?>
```

### Mistake entry

```markdown
---
type: mistake
date: 2026-04-10
tags: [agent-memory, mistake]
status: active
summary: <what went wrong in one line>
---

## What Happened

<Description of the failure>

## Why It Failed

<Root cause>

## How to Avoid It

<Concrete guidance for future agents>
```

### Pattern entry

```markdown
---
type: pattern
date: 2026-04-10
tags: [agent-memory, pattern]
status: active
summary: <the pattern in one line>
---

## Pattern

<What the pattern is>

## When to Use

<Context where this applies>

## Example

<Concrete example or command>
```

---

## 4. Taking a Snapshot

Take a snapshot at the end of a session or before a major context switch. A snapshot is the most important entry type for continuity — it lets the next agent (or next session) resume without re-discovering context.

**A snapshot must include three sections:**

```markdown
---
type: snapshot
date: 2026-04-10
tags: [agent-memory, snapshot]
status: active
summary: <what was happening when this snapshot was taken>
---

## Task State

- **In progress:** <what was being worked on>
- **Done:** <what was completed this session>
- **Next:** <what to pick up next>

## Conversation Summary

<2-4 sentences: what was discussed, key decisions made, open questions>

## Files Touched

- `path/to/file.md` — created / edited / read
```

**Create the snapshot:**

```bash
obsidian create \
  path="agent-memory/snapshots/2026-04-10-session-end.md" \
  content="<snapshot content>" \
  silent
```

> [!example] When to snapshot
> - Before ending a session that isn't fully complete
> - Before switching to a very different task
> - After a large batch of changes the next agent needs to know about

---

## 5. Flagging Stale Entries (Pruning)

Entries become stale when they're superseded, incorrect, or no longer relevant. Agents flag them — you (the human) decide what to delete.

**Flag an entry as stale:**

```bash
obsidian property:set name="status" value="stale" file="2026-04-10-old-context"
```

Stale entries automatically appear in [[stale.base|stale.base]] and disappear from [[memory.base|memory.base]].

> [!warning] Never delete entries directly
> Set `status: stale` and let the human review. The `stale.base` view is the pruning queue — open it in Obsidian, review each entry, and delete the files you're happy to remove.

---

## 6. Browsing Memories in Obsidian

| View | What it shows |
|------|--------------|
| [[memory.base]] | All active entries, grouped by type |
| [[stale.base]] | Entries flagged for pruning |

Open either `.base` file in Obsidian to get a live table view. The `memory.base` view is the best starting point for a new session when you want to browse rather than search.

---

## 7. Quick Reference Card

```
SESSION START
  qmd search "topic" -c <your-collection-name>
  obsidian read file="<entry>"
  obsidian list path="agent-memory/snapshots"

WRITE AN ENTRY
  obsidian create path="agent-memory/<type>/YYYY-MM-DD-slug.md" content="..." silent

FLAG AS STALE
  obsidian property:set name="status" value="stale" file="<entry>"

BROWSE
  Open agent-memory/memory.base in Obsidian
  Open agent-memory/stale.base to review pruning queue
```

---

*Full protocol: [[AGENTS]] · Active memories: [[memory.base]]*
