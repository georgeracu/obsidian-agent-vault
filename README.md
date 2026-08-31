# obsidian-agent-vault

An opinionated Obsidian vault template combining PARA+Zettelkasten organisation with a shared agent memory protocol for Claude, Gemini, and any AI tool that can read a markdown file.

Any AI agent starts every session by reading `AGENTS.md`, which tells it exactly how to load prior context, write new memories, and leave the vault in a better state than it found it.

## Prerequisites

- **Obsidian v1.9+** — Bases requires v1.9. Enable under Settings > Core Plugins > Bases
- **obsidian CLI** — ships with the Obsidian app. On macOS, add to PATH:
  ```bash
  ln -s /Applications/Obsidian.app/Contents/MacOS/obsidian /usr/local/bin/obsidian
  ```
  On Linux/Windows: locate the Obsidian binary and add its directory to your PATH.
- **qmd** — local semantic search for your vault:
  ```bash
  npm install -g @tobilu/qmd
  ```

## Setup

1. Click **"Use this template"** on GitHub, then clone your new repo
2. Open the cloned folder as a vault in Obsidian: **File > Open folder as vault**
3. Enable Bases if not already on: **Settings > Core Plugins > Bases**
4. Set up the `obsidian` CLI symlink (see Prerequisites above)
5. Decide on a collection name for qmd (e.g. `my-vault`), then index the vault:
   ```bash
   qmd collection add /path/to/your/vault --name <your-collection-name>
   qmd embed
   ```
   > `qmd embed` downloads ~2 GB of models on first run and may take several minutes.
   > Re-run `qmd embed` after adding significant numbers of new notes.
6. Find-and-replace the following placeholders in `AGENTS.md` and `Resources/AI & Agents/agent-memory-tutorial.md`:

   | Placeholder | Replace with |
   |-------------|-------------|
   | `<your-vault-name>` | Your Obsidian vault name (the folder name you opened in step 2) |
   | `<your-collection-name>` | The name you chose in step 5 |

## Verify your setup

- Open `agent-memory/memory.base` in Obsidian — it should render as an empty table
- Run `qmd search "agent" -c <your-collection-name>` — should return results from this vault

## What's included

- **`AGENTS.md`** — the protocol every agent reads at session start. One file, every agent, same protocol.
- **Five memory types**: `context`, `decision`, `mistake`, `pattern`, `snapshot` — stored as plain markdown files in `agent-memory/`
- **`memory.base`** — a live Obsidian Bases table of all active memories, grouped by type, no index to maintain
- **`stale.base`** — pruning queue: agents flag outdated entries; you approve deletions
- **PARA+Zettelkasten vault structure** — see `Vault Structure.md` for the full layout and decision tree
- **`Permanent Notes/Assisting-User-Context.md`** — fill this in so agents understand who you are and how to help you

## Included skills

Agent skills are stored in `.agents/skills/` and extend what AI agents can do in this vault. Skills are loaded on demand — agents read them when they need guidance on a specific tool or task.

| Skill | What it does |
|-------|-------------|
| `obsidian-cli` | Interact with Obsidian vaults from the command line — read, create, search, and manage notes |
| `obsidian-bases` | Create and edit Obsidian Bases (`.base` files) with filters, formulas, and database-style views |
| `obsidian-markdown` | Write Obsidian Flavored Markdown — wikilinks, embeds, callouts, properties, and more |
| `qmd` | Search your vault semantically and by keyword using `qmd` |
| `defuddle` | Extract clean markdown from web pages, stripping clutter and navigation |
| `json-canvas` | Create and edit JSON Canvas files (`.canvas`) for visual maps and flowcharts |

## Using with AI agents

Point your agent at `AGENTS.md` at the start of every session. For Claude Code, `CLAUDE.md` does this automatically. For other tools, add the equivalent instruction to their config file (e.g. `GEMINI.md` for Gemini CLI).

The agent reads the protocol, searches for relevant memories using `qmd`, loads prior context, does the work, and writes a snapshot before stopping. The next session — whether the same agent or a different one — picks up where it left off.

## Links

- Blog post: [AI Agents That Remember: Building a Second Brain for Claude and Gemini](https://open.substack.com/pub/georgeracu/p/ai-agents-that-remember-building?utm_campaign=post-expanded-share&utm_medium=web)
- qmd: [github.com/tobi/qmd](https://github.com/tobi/qmd)
- Obsidian: [obsidian.md](https://obsidian.md)
- Install Obsidian skills: `npx skills add git@github.com:kepano/obsidian-skills.git`
- Install qmd skill: `npx skills add https://github.com/tobi/qmd`
- Install second-brain skill: `npx skills add https://github.com/sean-esk/second-brain-gtd`

## Licensing

This template is MIT licensed (see [LICENSE](LICENSE)). Bundled third-party skills keep their upstream licences and copyright notices — see [THIRD-PARTY-LICENSES.md](THIRD-PARTY-LICENSES.md), with per-skill provenance and content hashes in [skills-lock.json](skills-lock.json).
