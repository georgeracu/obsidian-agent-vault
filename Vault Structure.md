---
title: Vault Structure
date: 2026-04-10
tags: [second-brain, reference, structure]
aliases:
  - How This Vault Is Organised
---

# Vault Structure

This vault uses **PARA + Zettelkasten** — a combination of GTD-style organisation (Projects, Areas, Resources, Archive) with atomic permanent notes for building knowledge over time.

---

## The PARA Structure

```
Inbox/                    ← Everything lands here first
  Daily/                  ← Daily capture files (YYYY-MM-DD.md)
  Fleeting-Notes/         ← Knowledge items awaiting processing

Projects/                 ← Active work with a clear outcome and deadline
  Example-Project-A/
  Example-Project-B/

Areas/                    ← Ongoing responsibilities (no end date)
  Career/
  Personal/
  AI & Tooling/
  Relationships/          ← One note per important person

Resources/                ← Reference material by topic
  AI & Agents/
  Architecture/
  Documents/
  Platform & Ops/
  Testing/
  Writing & Publishing/
  Reference-Notes/        ← Summaries of books, articles, external sources

Archive/                  ← Completed or inactive items from the above
```

---

## Knowledge (Zettelkasten)

```
Permanent Notes/          ← Synthesised, atomic ideas in your own words
  Assisting-User-Context  ← Your goals & context for AI agents
```

Permanent notes are the long-term knowledge base. Each note is:
- **Atomic** — one idea per note
- **In your own words** — not copy-paste
- **Linked** — connected to other notes via wikilinks

Fleeting notes (in `Inbox/Fleeting-Notes/`) are the raw material. Process them into permanent notes during weekly review.

---

## Daily Workflow

← Add as needed for your workflow

---

## Support & Utilities

```
Excalidraw/               ← Diagrams and visual notes
Mermaid/                  ← Mermaid diagram definitions
Templates/                ← Reusable note templates
agent-memory/             ← AI agent memory store (see AGENTS.md)
```

---

## How to Decide Where Something Goes

| Question | Answer → Location |
|----------|-----------------|
| Is it a raw thought or task? | `Inbox/Daily/` |
| Is it a knowledge item to process later? | `Inbox/Fleeting-Notes/` |
| Does it have a clear outcome and deadline? | `Projects/` |
| Is it an ongoing responsibility? | `Areas/` |
| Is it reference material? | `Resources/` |
| Is it a synthesised insight in my own words? | `Permanent Notes/` |
| Is it done or inactive? | `Archive/` |

---

## AI Agent Entry Point

All agents read [[AGENTS]] at session start. See [[agent-memory-tutorial]] for how the memory system works.

---

*Structure based on the second-brain skill — PARA + GTD + Zettelkasten*
