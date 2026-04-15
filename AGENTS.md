# First-Time Setup

Before using this vault with any AI agent, find-and-replace the following placeholders throughout this file and `Resources/AI & Agents/agent-memory-tutorial.md`:

| Placeholder | Replace with |
|-------------|-------------|
| `<your-vault-name>` | Your Obsidian vault name (the folder name, e.g. `My Vault`) |
| `<your-collection-name>` | The name you used when running `qmd collection add` |

---

## Agent Memory Protocol

This file is the single entry point for all AI agents working in this vault.
Read it at the start of every session.

---

## Hard Constraints

**CRITICAL:**

1. **Scope:** You are ONLY permitted to modify files and directories within "<your-vault-name>". Do NOT change anything outside of this vault.
2. **Vault Targeting:** All `obsidian` CLI commands MUST target the vault "<your-vault-name>" by using the `vault="<your-vault-name>"` parameter.
3. **Tooling:** Use ONLY `obsidian` CLI and `qmd` for all vault interactions. Do NOT use raw file I/O, direct Bash file commands (`cat`, `echo`, `sed`, `awk`, `rm`, etc.), or any file system tools outside of `obsidian` CLI. Every read, write, create, and update must go through `obsidian` CLI.

---

## Session Start

1. Use `qmd` to search `agent-memory/` for context relevant to your current task
2. Read matching entries with `obsidian` CLI
3. If resuming prior work, check `agent-memory/snapshots/` for a recent snapshot

---

## Writing Memories

Write a new entry any time you discover something worth preserving for future sessions.

| Type | Subdirectory | When to write |
|------|-------------|---------------|
| `context` | `agent-memory/context/` | Active project state, working assumptions, vault structure notes |
| `decision` | `agent-memory/decisions/` | A choice made that future agents should respect |
| `mistake` | `agent-memory/mistakes/` | A pitfall you hit — what failed and why |
| `pattern` | `agent-memory/patterns/` | A reusable approach that worked well |
| `snapshot` | `agent-memory/snapshots/` | End of session or before a major context switch |

**Entry filename format:** `YYYY-MM-DD-<short-slug>.md`

**Required frontmatter for every entry:**

```yaml
---
type: context|decision|mistake|pattern|snapshot
date: YYYY-MM-DD
tags: [agent-memory, <type>]
status: active
summary: One-line description of this entry
---
```

Use `obsidian` CLI to create entries. Do not use raw file I/O.

---

## Snapshots

A snapshot captures full working memory at a point in time. It must include:

1. **Task state** — what was in progress, what is done, what comes next
2. **Conversation summary** — what was discussed, key decisions made, open questions
3. **Files touched** — list of files created, edited, or read during the session

---

## Pruning (Flagging Stale Entries)

If an entry is outdated, superseded, or no longer relevant:

1. Update its frontmatter: set `status: stale` using `obsidian` CLI
2. Do NOT delete the file — the human reviews stale entries via the `stale.base` view in Obsidian and approves deletions

---

## Directory Reference

```bash
agent-memory/
├── memory.base       ← Live index of all active entries (Obsidian Bases)
├── stale.base        ← Pruning queue — entries with status: stale
├── context/          ← Active project/vault context notes
├── decisions/        ← Architectural and technical choices
├── mistakes/         ← Known pitfalls to avoid
├── patterns/         ← Reusable approaches that worked well
└── snapshots/        ← Session checkpoints (full working memory)
```

---

## Session End

When the user says to wrap up, end, or close the session, use the **wrap-up** skill. It handles:

1. Writing a session snapshot to `agent-memory/snapshots/`
2. Marking superseded snapshots as stale
3. Refreshing the `qmd` search index (`qmd update && qmd embed`)
4. Surfacing open items to the user

The `qmd update && qmd embed` step must also be run during a session after any batch of vault note changes (create, update, delete) to keep the search index current.

---

## Tooling Reference

| Task                                 | Tool                                       |
| ------------------------------------ | ------------------------------------------ |
| Search memory by topic/keyword       | `qmd`                                      |
| Read a specific entry                | `obsidian` CLI read                        |
| Create a new memory entry            | `obsidian` CLI create note                 |
| Update frontmatter (e.g. flag stale) | `obsidian` CLI update properties           |
| Refresh search index after changes   | `qmd update && qmd embed`                  |
| End a session                        | **wrap-up** skill                          |
| Browse all active memories           | Obsidian — open `agent-memory/memory.base` |
| Review pruning candidates            | Obsidian — open `agent-memory/stale.base`  |
