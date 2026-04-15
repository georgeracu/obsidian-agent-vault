---
name: wrap-up
description: Use when the user says to wrap up, end, or close a session in <your-vault-name>. Covers snapshot, stale-flagging, and open-item surfacing.
---

# Session Wrap-Up — <your-vault-name>

End every vault session with a clean handoff. Four steps, in order.

All agent memory lives in `agent-memory/` inside the vault. Do NOT write to or update any external memory files.

## Step 1: Write session snapshot

Create `agent-memory/snapshots/YYYY-MM-DD-session-end.md` (use today's date):

```bash
obsidian vault="<your-vault-name>" create name="YYYY-MM-DD-session-end" path="agent-memory/snapshots/YYYY-MM-DD-session-end.md" content="<content>" silent
```

Required frontmatter:
```
type: snapshot
date: YYYY-MM-DD
tags: [agent-memory, snapshot]
status: active
summary: <one-line description of session>
```

Required content sections:
- **Task state** — what was done, what is in progress, what comes next
- **Conversation summary** — key decisions made, open questions
- **Files touched** — list of files created, edited, or moved
- **Open items** — anything the user needs to act on before next session

## Step 2: Mark superseded snapshots as stale

Search for earlier snapshots from the same session:

```bash
obsidian vault="<your-vault-name>" search query="type:snapshot" limit=20
```

For every interim snapshot written during the session (task checkpoints, interrupted snapshots), set status to stale:

```bash
obsidian vault="<your-vault-name>" property:set file="<snapshot-name>" name="status" value="stale"
```

Do NOT mark the new session-end snapshot stale. Only interim ones.

## Step 3: Update qmd index

Refresh the search index so future sessions have up-to-date embeddings. Run this after any vault notes were created, updated, or deleted during the session:

```bash
qmd update && qmd embed
```

## Step 4: Surface open items to the user

End with a short, scannable message covering:
- Empty folders requiring Finder deletion
- Broken wikilinks reminder (if files were moved)
- Any decisions deferred to next session

## Common Mistakes

| Mistake | Fix |
|---|---|
| Updating any external memory file | All memory lives in vault `agent-memory/` only — never write outside it |
| Skipping stale-flagging of interim snapshots | Stale entries clutter the memory.base view — always flag them |
| Writing snapshot but not surfacing open items | The user needs to know what to do before closing Obsidian |
| Skipping `qmd update && qmd embed` after vault changes | Future sessions will search stale content — always refresh the index |
