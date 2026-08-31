---
name: wrap-up
description: Use at the end of a vault session, or at a mid-session checkpoint, to write a snapshot, promote durable lessons into knowledge entries, flag superseded snapshots stale, refresh hot.md and the qmd index, and surface open items to the user.
---

# Session Wrap-Up

End every session with a clean handoff. Six steps, in order.

## When to run this

Do not wait for the user to type `/wrap-up`. The sessions that lose the most work are the long ones that never reach a tidy ending, as context runs out or the laptop closes mid-task. Run at least steps 1, 3 and 4 when any of these hits, whichever comes first:

- roughly 100 tool calls into the session
- a context compaction has just occurred
- the work is about to switch topic or repo
- a long unattended run is about to start
- the user signals the end ("that's it for today", "closing the laptop")

An interim snapshot is cheap. Write it, then flag it stale when the session-end one lands.

**This applies to repo-work sessions too.** When the session was engineering in an external repo and touched no notes, still write the snapshot: the reasoning, the dead ends and the open questions are exactly what the next session needs, and they exist nowhere else once the transcript scrolls away.

## Step 1: Write session snapshot

Create `agent-memory/snapshots/YYYY-MM-DD-session-end-<topic-slug>.md` — today's date plus a 2-4 word slug for the session's main thread, e.g. `2026-07-17-session-end-inbox-triage.md`. Date-only names collide on multi-session days, and a syncing filesystem (OneDrive, Dropbox, iCloud Drive) turns the collision into conflict copies. If the target path already exists, extend the slug rather than overwrite.

```bash
obsidian vault="My Vault" create path="agent-memory/snapshots/YYYY-MM-DD-session-end-<topic-slug>.md" content="<content>" silent
```

For anything longer than a couple of lines, do not fight shell quoting in `content=`. Write the payload to a temporary file first, then create the note from it in one `eval`:

```bash
obsidian vault="My Vault" eval code='(async()=>{const fs=require("fs");const c=fs.readFileSync("/tmp/payload.md","utf8");await app.vault.create("agent-memory/snapshots/<name>.md",c);return "created";})()'
```

Pass `path=` only. Do **not** add `name=`: supplying both mints a duplicate note, and a `name=` containing `/` fails outright with `name cannot contain "/"`.

Required frontmatter:

```yaml
type: snapshot
date: YYYY-MM-DD
tags:
  - agent-memory
  - snapshot
status: active
summary: <one-line description of session>
```

Required content sections:

- **Task state** — what was done, what is in progress, what comes next
- **Conversation summary** — key decisions made, open questions
- **Files touched** — list of files created, edited, or moved
- **Open items** — anything the user needs to act on before next session

## Step 2: Promote durable lessons out of the snapshot

A snapshot goes stale within a day. Anything learned that will still be true next month belongs in the knowledge layer, or it is lost the moment this snapshot is flagged.

Before moving on, ask what this session learned that a future session would otherwise repeat, and write each as its own entry under `agent-memory/`:

- `mistakes/` — a pitfall hit, what failed and why
- `patterns/` — an approach that worked and would work again
- `decisions/` — a choice future agents should respect
- `context/` — durable project state or a working assumption

Each needs the standard frontmatter plus a `## Related` section of `[[wikilink]]` bullets (AGENTS.md requires cross-linking). Verify every link resolves before finishing, and never link an external-memory slug such as `[[project_...]]`, as those files live outside the vault and always dangle.

Most sessions yield nothing durable, and writing nothing is the right answer then. A session that hit a tool footgun, changed an approach, or corrected a wrong assumption almost always yields one.

**Memory routing.** Vault-specific facts go in `agent-memory/`, never in the external auto-memory. A tool behaviour that applies beyond this vault (an `obsidian` CLI or `qmd` footgun that would bite in any project) may be recorded in both.

## Step 3: Mark superseded snapshots as stale

Snapshots are a chronological log, so only the latest stays active in `memory.base`. Flag every earlier snapshot, including interim ones from this same session and the previous session-end entry.

Do **not** use `search query="type:snapshot"` — `type:` is not a supported search operator, so this has never worked. Use one guarded `eval` pass instead:

```bash
obsidian vault="My Vault" eval code='(async()=>{
  const keep = "agent-memory/snapshots/<TODAYS-SNAPSHOT>.md";
  const files = app.vault.getMarkdownFiles().filter(f => f.path.startsWith("agent-memory/snapshots/"));
  let flagged = 0, skipped = [];
  for (const f of files) {
    if (f.path === keep) continue;
    const fm = app.metadataCache.getFileCache(f)?.frontmatter;
    if (!fm || fm.type !== "snapshot") { skipped.push(f.path); continue; }
    if (fm.status === "stale") continue;
    await app.fileManager.processFrontMatter(f, w => { w.status = "stale"; });
    flagged++;
  }
  return "flagged " + flagged + ", skipped " + skipped.length;
})()'
```

Gate on `type: snapshot` and skip already-stale files so nothing outside the log is touched and no file is rewritten needlessly.

**Verify rather than trust the return value.** This `eval` sometimes returns nothing when another Claude Code session holds the Electron single-instance lock. Confirm the counts independently:

```bash
obsidian vault="My Vault" eval code='(async()=>{const c={};for(const f of app.vault.getMarkdownFiles()){if(!f.path.startsWith("agent-memory/snapshots/"))continue;const s=(app.metadataCache.getFileCache(f)?.frontmatter||{}).status||"none";c[s]=(c[s]||0)+1;}return JSON.stringify(c);})()'
```

Expect exactly one `active`. Then spot-check one rewritten file: `processFrontMatter` reserialises the YAML, and block-list `tags:` must survive as block-list, never a flow array or a quoted literal.

For a single file, `property:set` is simpler:

```bash
obsidian vault="My Vault" property:set file="<snapshot-name>" name="status" value="stale"
```

Do NOT mark the new session-end snapshot stale.

**Retention.** Stale snapshots older than 90 days have served their purpose. Roughly monthly, list them (path + date) and propose deletion as a batch; deletion is destructive, so it happens only with explicit approval, via the CLI. Knowledge entries are exempt — they stay until the human prunes them through `stale.base`.

## Step 4: Refresh hot.md

`agent-memory/hot.md` is the sub-500-word orientation cache AGENTS.md reads at session start. Rewrite it in place — replace the whole body, never append; it is a cache, not a log:

- `## Now` — active work threads, three to six bullets
- `## Standing` — the few constraints worth re-reading cold
- `## Pointers` — a wikilink to today's snapshot plus two or three key entries

Set `date:` to today. Keep the body under 500 words; cut the oldest Now bullets before anything else. Use the payload pattern from step 1 with `app.vault.modify` (or recreate the file if missing):

```bash
obsidian vault="My Vault" eval code='(async()=>{const fs=require("fs");const c=fs.readFileSync("/tmp/hot.md","utf8");const f=app.vault.getAbstractFileByPath("agent-memory/hot.md");if(f){await app.vault.modify(f,c);}else{await app.vault.create("agent-memory/hot.md",c);}return "refreshed";})()'
```

## Step 5: Update qmd index

Refresh the search index so future sessions have up-to-date embeddings. Run after any vault notes were created, updated, or deleted:

```bash
qmd update && qmd embed
```

## Step 6: Surface open items to the user

End with a short, scannable message covering:

- Any files in `_from-migration/` needing manual review
- Empty folders requiring Finder deletion
- Broken wikilinks reminder (if files were moved)
- Any decisions deferred to next session
- Anything the user must do outside the vault (post a link, approve a deletion)

Keep it terse. No trailing summary of work the user just watched.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Waiting for `/wrap-up` on a long session | Checkpoint at ~100 tool calls, after a compaction, or before a topic switch |
| Skipping the snapshot because it was "repo work, not vault work" | The reasoning is the artefact; write it anyway |
| Burying a durable lesson in the snapshot body | Promote it to `mistakes/`, `patterns/`, `decisions/` or `context/` in step 2 |
| Using `search query="type:snapshot"` | Not a supported operator; use the `eval` pass in step 3 |
| Passing both `name=` and `path=` to `create` | `path=` alone; both together mint a duplicate note |
| Trusting a silent `eval` return as success | Verify with the status-count query, then spot-check one file |
| Writing a knowledge entry with no `## Related` | Cross-linking is required; verify each link resolves |
| Putting vault facts in the external auto-memory | Vault context lives in `agent-memory/`; only cross-project tool footguns go in both |
| Appending to `hot.md` | It is a cache, not a log — replace the whole body |
| Skipping the `hot.md` refresh | Session start reads it; a stale hot misleads the next session |
| Skipping `qmd update && qmd embed` after vault changes | Future sessions will search stale content |
