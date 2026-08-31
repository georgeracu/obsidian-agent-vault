---
name: memory-lint
description: Use when auditing agent-memory health in this vault — checking for schema drift, dangling wikilinks, orphaned or under-linked entries, stale-but-active entries, sync-conflict duplicate files, ambiguous basenames, a stale hot.md, and provenance coverage. Read-only; produces a report the human acts on.
---

# Memory Lint

## Overview

A read-only health check over `agent-memory/`. It surfaces problems; the human acts on them via `stale.base` and approved edits. The lint itself never mutates the vault.

## When to use

- Periodic memory hygiene, or after a heavy capture session
- When the graph feels disconnected, or entries look stale/duplicated
- Before crystallising or reorganising memory

## Scope

Knowledge layer only: `context/`, `decisions/`, `mistakes/`, `patterns/`. **Snapshots are a chronological log, not graph nodes** — schema-check them at most, never lint them for links or orphans. `hot.md` gets its own staleness check and nothing else.

## The checks

1. **Schema drift** — required frontmatter present and well-formed: `type` matches its subdirectory and is one of context|decision|mistake|pattern|snapshot; `status` is active|stale; `summary` and `date` present; tags are block-list YAML — flag both the broken quoted form `tags: "[...]"` and the inline flow form `tags: [a, b]`, since AGENTS.md and wrap-up both require block-list. Flag entries on the external `name`/`description`/`type: feedback` schema.
2. **Dangling wikilinks** — `[[links]]` whose target resolves nowhere in the vault.
3. **Under-linking / orphans** — knowledge entries with no *working* inter-memory link. An entry whose only links dangle or point outside `agent-memory/` counts as orphaned. Report the aggregate (N/total) and the list.
4. **Stale-source candidates** — `context`/`decision` entries with `status: active`, older than ~45 days, that also carry a completion/TODO signal in the body (TODO, pending, draft, published, finalised, ready-for-review, past-due follow-up). The `## Related` section is stripped before matching so cross-link reasons never trigger it; `pattern`/`mistake` entries are treated as evergreen and skipped, as is any entry carrying `evergreen: true` in frontmatter.
5. **Duplicates / sync-conflicts** — space-numbered conflict copies (` 1.md`, ` 2.md`) left by a syncing filesystem (OneDrive, Dropbox, iCloud Drive) and near-empty files.
6. **Ambiguous basenames** — the same filename in more than one `agent-memory/` subdirectory. Unqualified `[[wikilinks]]` to a shared basename resolve by shortest path, which may be the wrong entry; links to these must be path-qualified.
7. **Hot cache staleness** — `agent-memory/hot.md` missing, over 500 words, or dated before the newest active snapshot (meaning the last wrap-up skipped its refresh step).
8. **Provenance coverage** (informational) — count of `status: active` entries carrying a `source:` field, plus the list of those without. Not a hard flag: an entry whose only origin is the work session legitimately has no source.
9. **Index freshness** (shell, after the eval) — run `qmd update` and read the <your-collection-name> line. Anything other than `0 new, 0 updated` means the index was stale and a previous session skipped its wrap-up refresh.

## How to run

Enumerate and read the corpus with qmd (`qmd ls <your-collection-name>/agent-memory`, `qmd multi-get`), then run the deterministic lint inside Obsidian. The script below reports every check in one pass via `metadataCache` (links + frontmatter) and `cachedRead` (body signals). Paste it into a single-quoted heredoc and run `obsidian vault="My Vault" eval code="$(cat /tmp/lint.js)"`:

```js
(async () => {
  const KNOW = { context:"context", decisions:"decision", mistakes:"mistake", patterns:"pattern" };
  const STALE_TYPES = { context:1, decision:1 };
  const all = app.vault.getMarkdownFiles().filter(f => f.path.startsWith("agent-memory/"));
  const know = all.filter(f => Object.keys(KNOW).some(d => f.path.startsWith("agent-memory/"+d+"/")));
  const ALLOWED = { context:1, decision:1, mistake:1, pattern:1, snapshot:1 };
  const STALE_DAYS = 45, NOW = Date.now();
  const SIGNAL = /\b(TODO|pending|draft|published|finalised|finalized|ready-for-review|follow-?up)\b/i;
  const drift=[], dangling=[], orphans=[], stale=[], dupes=[], noSource=[], hot=[];
  let activeKnow = 0;
  for (const f of all) if (/ \d+\.md$/.test(f.path)) dupes.push(f.path);
  const byName = {};
  for (const f of all) if (!f.path.startsWith("agent-memory/snapshots/")) (byName[f.name] = byName[f.name] || []).push(f.path);
  const ambiguous = Object.values(byName).filter(a => a.length > 1).map(a => a.join("  <->  "));
  for (const f of know) {
    const c = app.metadataCache.getFileCache(f) || {};
    const fm = c.frontmatter || {};
    const sub = f.path.split("/")[1];
    const body = await app.vault.cachedRead(f);
    const issues = [];
    if (!fm.type) issues.push("missing type");
    else if (!ALLOWED[fm.type]) issues.push("type '"+fm.type+"' not allowed");
    else if (KNOW[sub] && fm.type !== KNOW[sub]) issues.push("type != subdir ("+fm.type+")");
    if (!fm.date) issues.push("missing date");
    if (!fm.summary) issues.push("missing summary");
    if (fm.status && fm.status !== "active" && fm.status !== "stale") issues.push("status '"+fm.status+"'");
    const fmBlock = (body.match(/^---\n[\s\S]*?\n---/) || [""])[0];
    if (/^tags:\s*"\[/m.test(fmBlock)) issues.push("quoted-array tags");
    else if (/^tags:\s*\[/m.test(fmBlock)) issues.push("flow-array tags");
    if (issues.length) drift.push(f.path + " — " + issues.join(", "));
    let working = 0;
    for (const l of (c.links || [])) {
      const dest = app.metadataCache.getFirstLinkpathDest(l.link, f.path);
      if (!dest) dangling.push(f.path + " — [[" + l.link + "]]");
      else if (dest.path.startsWith("agent-memory/")) working++;
    }
    if (working === 0) orphans.push(f.path);
    const text = body.replace(/\n#{1,6} Related[\s\S]*$/, "");
    const d = fm.date ? Date.parse(fm.date) : NaN;
    const age = isFinite(d) ? Math.round((NOW - d) / 86400000) : null;
    if (fm.status === "active" && !fm.evergreen && STALE_TYPES[fm.type] && age !== null && age > STALE_DAYS && SIGNAL.test(text))
      stale.push(f.path + " — " + age + "d, completion/TODO signal");
    if (fm.status === "active") { activeKnow++; if (!fm.source) noSource.push(f.path); }
  }
  const hf = app.vault.getAbstractFileByPath("agent-memory/hot.md");
  if (!hf) hot.push("hot.md missing");
  else {
    const hfm = app.metadataCache.getFileCache(hf)?.frontmatter || {};
    const hb = await app.vault.cachedRead(hf);
    const words = hb.replace(/^---[\s\S]*?---/, "").trim().split(/\s+/).length;
    if (words > 500) hot.push("hot.md over 500 words (" + words + ")");
    const newest = all.filter(f => f.path.startsWith("agent-memory/snapshots/"))
      .map(f => app.metadataCache.getFileCache(f)?.frontmatter || {})
      .filter(m => m.status === "active").map(m => String(m.date)).sort().pop();
    if (hfm.date && newest && String(hfm.date) < newest) hot.push("hot.md dated " + hfm.date + ", newest active snapshot " + newest);
  }
  const sec = (t, a) => [t + " (" + a.length + ")", ...a, ""].join("\n");
  return [
    sec("SCHEMA DRIFT", drift),
    sec("DANGLING LINKS", dangling),
    sec("DUPLICATES / CONFLICTS", dupes),
    sec("AMBIGUOUS BASENAMES", ambiguous),
    sec("STALE-SOURCE CANDIDATES", stale),
    sec("HOT CACHE", hot),
    "NO WORKING INTER-MEMORY LINK (" + orphans.length + "/" + know.length + ")", ...orphans, "",
    "PROVENANCE COVERAGE: " + (activeKnow - noSource.length) + "/" + activeKnow + " active entries have source",
    ...noSource.map(p => "  no source: " + p)
  ].join("\n");
})()
```

Then check index freshness from the shell:

```bash
qmd update 2>&1 | grep -A2 "<your-collection-name>"
```

## Report format

Group by check, one line per finding as `path — reason`, end with the orphan aggregate, the provenance coverage line, and the index-freshness result. Write nothing to the vault.

## Acting on findings (separate, approved steps)

- **Stale** — human flags via `stale.base`, or set `status: stale` with per-entry approval (`processFrontMatter`/`property:set`). Never auto-flag.
- **Duplicates** — destructive; propose keep/drop per cluster and get approval before trashing. Some space-numbered files have no base copy and carry unique content — rename those (strip the suffix) rather than delete.
- **Under-linking** — backfill a `## Related` section of `[[wikilink]]` bullets (required by AGENTS.md). Verify every new link resolves with `getFirstLinkpathDest` before finishing.
- **Ambiguous basenames** — path-qualify the links (`[[agent-memory/decisions/<name>|alias]]`); renaming an entry is destructive and needs approval.
- **Flow-array tags** — convert to block-list with `processFrontMatter` (it serialises block-list by default), per-entry approval as with any bulk frontmatter rewrite.
- **Stale hot.md** — rebuild it at the next wrap-up (step 4 of the wrap-up skill) rather than patching it standalone, so the refresh stays coupled to a snapshot.
- **Provenance** — add a `source:` list only where the entry actually states an origin (AGENTS.md rule). Never invent a source.

## Common mistakes

- **Letting inline-flow tags pass** `[a, b]` — the vault convention is block-list everywhere (AGENTS.md, wrap-up); both the quoted `"[...]"` form and the flow form are drift.
- **Linting snapshots for links** — they are an append-only log; exempt them. `hot.md` likewise gets only its staleness check.
- **Counting a dangling link as linked** — the orphan check counts *working* inter-memory links only; an entry whose every link dangles is an orphan.
- **Forcing links or sources** onto standalone entries — omit rather than invent a weak edge or a fabricated origin.
- **Treating provenance coverage as a hard flag** — it is informational; not every entry has a recoverable source.
- **Stale false-positives** — the heuristic is type-scoped (patterns/mistakes skipped as evergreen), honours an explicit `evergreen: true`, and strips `## Related` before matching, so cross-link reasons never trigger a flag. Keep all three guards if editing the script.
- **Re-flagging a standing constraint** — a platform limit or a routing decision is not a task, so it never "completes" and will resurface on every run. Set `evergreen: true` on it once rather than flagging it stale, which would hide a rule that is still in force.
- **Mutating during the lint** — it is read-only; all fixes are separate, approved steps.
