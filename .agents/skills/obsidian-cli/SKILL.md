---
name: obsidian-cli
description: Interact with Obsidian vaults using the Obsidian CLI to read, create, search, and manage notes, tasks, properties, and more. Also supports plugin and theme development with commands to reload plugins, run JavaScript, capture errors, take screenshots, and inspect the DOM. Use when the user asks to interact with their Obsidian vault, manage notes, search vault content, perform vault operations from the command line, or develop and debug Obsidian plugins and themes.
license: MIT
metadata:
  upstream: https://github.com/kepano/obsidian-skills
  upstream_license: "MIT (c) 2026 Steph Ango (@kepano)"
  modified_by: George Racu
---

# Obsidian CLI

Use the `obsidian` CLI to interact with a running Obsidian instance. Requires Obsidian to be open.

## Command reference

Run `obsidian help` to see all available commands. This is always up to date. Full docs: https://help.obsidian.md/cli

## Syntax

**Parameters** take a value with `=`. Quote values with spaces:

```bash
obsidian create name="My Note" content="Hello world"
```

**Flags** are boolean switches with no value:

```bash
obsidian create name="My Note" silent overwrite
```

For multiline content use `\n` for newline and `\t` for tab.

## File targeting

Many commands accept `file` or `path` to target a file. Without either, the active file is used.

- `file=<name>` — resolves like a wikilink (name only, no path or extension needed)
- `path=<path>` — exact path from vault root, e.g. `folder/note.md`

## Vault targeting

Commands target the most recently focused vault by default. Use `vault=<name>` as the first parameter to target a specific vault:

```bash
obsidian vault="My Vault" search query="test"
```

## Common patterns

```bash
obsidian read file="My Note"
obsidian create name="New Note" content="# Hello" template="Template" silent
obsidian append file="My Note" content="New line"
obsidian search query="search term" limit=10
obsidian daily:read
obsidian daily:append content="- [ ] New task"
obsidian property:set name="status" value="done" file="My Note"
obsidian tasks daily todo
obsidian tags sort=count counts
obsidian backlinks file="My Note"
```

Use `--copy` on any command to copy output to clipboard. Use `silent` to prevent files from opening. Use `total` on list commands to get a count.

## Plugin development

### Develop/test cycle

After making code changes to a plugin or theme, follow this workflow:

1. **Reload** the plugin to pick up changes:
   ```bash
   obsidian plugin:reload id=my-plugin
   ```
2. **Check for errors** — if errors appear, fix and repeat from step 1:
   ```bash
   obsidian dev:errors
   ```
3. **Verify visually** with a screenshot or DOM inspection:
   ```bash
   obsidian dev:screenshot path=screenshot.png
   obsidian dev:dom selector=".workspace-leaf" text
   ```
4. **Check console output** for warnings or unexpected logs:
   ```bash
   obsidian dev:console level=error
   ```

### Additional developer commands

Run JavaScript in the app context:

```bash
obsidian eval code="app.vault.getFiles().length"
```

Inspect CSS values:

```bash
obsidian dev:css selector=".workspace-leaf" prop=background-color
```

Toggle mobile emulation:

```bash
obsidian dev:mobile on
```

Run `obsidian help` to see additional developer commands including CDP and debugger controls.


## Common pitfalls and patterns

These have bitten in prior sessions. Apply pre-emptively.

### Pre-flight: confirm Obsidian is running

Run `pgrep -l Obsidian` once per session before the first `obsidian` call. If empty, ask the user to open Obsidian before proceeding — the `obsidian` binary blocks indefinitely with no error when no instance is running. Wrap calls in `timeout 15` while uncertain.

### Named parameters only

Every command takes named parameters (`path=`, `file=`, `content=`, `name=`, …). Positional arguments are silently ignored, and most commands fall back to the **active file**. `obsidian delete "folder/note.md"` trashes whatever is currently open in Obsidian, not the path you typed.

Correct: `obsidian delete path="folder/note.md"`.

### \`\n\` in `content=` becomes a real newline

`obsidian create content="..."` and `obsidian append content="..."` interpret `\n` and `\t` as escape sequences and write literal newline/tab characters. This corrupts JS, JSON, Python, regex, or shell content whose string literals use `\n` at the *language* level. Dataview surfaces this as `SyntaxError: Invalid or unexpected token`.

For code or any escape-sensitive content, write via `app.vault.create(path, text)` through `obsidian eval` — the JS API takes strings verbatim.

### Dot-prefixed paths need the adapter

Paths starting with `.` (`.agents/`, `.obsidian/`, `.claude/`) are hidden from Obsidian's indexed vault state. As a result:

- `obsidian read path=".agents/..."` returns *File not found*
- `app.vault.getAbstractFileByPath(".agents/...")` returns `null`
- `app.vault.getFiles()` does not list them

Use the lower-level adapter inside `obsidian eval`:

```js
await app.vault.adapter.exists(path);
await app.vault.adapter.read(path);
await app.vault.adapter.write(path, content);
await app.vault.adapter.list(folder);   // returns { files, folders }
```

The adapter operates on the raw filesystem and bypasses Obsidian's index.

### `eval` scripting: three things have to line up

1. **Backtick template literals** for JS strings, not single or double quotes. Apostrophes (`don't`, `it's`) and quotation marks both survive. Inside a backtick literal, escape a literal backtick as \`\` and a literal \`${...}\` as \`\\${...}\`.
2. **IIFE wrapper** — `eval` rejects top-level `await`. Use `(async () => { … })()` with the trailing `()` invocation.
3. **Single-quoted heredoc** for the script file — `cat > /tmp/script.js <<'JSEOF' … JSEOF` so zsh leaves `$` and backticks alone. Outer double quotes on `code="$(cat /tmp/script.js)"` preserve newlines.

### Apostrophe escape inside `eval`

When the JS body has literal apostrophes (English prose with contractions and possessives), don't use the shell quote-stacking recipe — it visually misaligns and silently produces broken JS. Inject from inside the JS:

```js
const APOS = String.fromCharCode(39);
const content = "subagent" + APOS + "s task description";
```

Now the shell sees no literal apostrophes in the `code=` argument. Concatenate `APOS` into any string that needs one.

### Diagnosing a stuck `eval` plugin

If `obsidian eval` returns `Command 'eval' not found` mid-session, an earlier wedged CLI call has locked the plugin. Find the pair:

```bash
ps -ef | grep -i obsidian | grep -v grep
```

The wedged session appears as a `/bin/zsh -c` wrapper plus its child `obsidian vault=…` process. Kill **both PIDs**, never the app processes under `/Applications/Obsidian.app/`. Exit 144 confirms SIGTERM landed. `eval` comes back without restarting Obsidian.

### Commands that do not exist

There is no `cat`, `grep`, `list`, `write`, `find` or `rm` subcommand. The near-misses waste a call each and the suggestion line is unhelpful. Use:

| Wanted | Actual |
|---|---|
| `cat` / read a note | `read path="..."` |
| `grep` / content search | `search query="..."` or `qmd search` |
| `list` a folder | `files path="..."`, or `eval` over `app.vault.getMarkdownFiles()` |
| `write` | `create path="..." overwrite` |
| `rm` | `delete path="..."` (trashes; needs user approval) |

`obsidian help` lists the real set and is always current. Bare `help` with no argument **hangs** and wedges the single-instance lock — always `obsidian help <command>`.

### `search` has no frontmatter operators

`search query="type:snapshot"` fails with `Operator "type" not recognized`, consistently. The search index does not expose frontmatter fields. To select notes by frontmatter, iterate `app.metadataCache.getFileCache(f).frontmatter` inside `eval`:

```bash
obsidian vault="My Vault" eval code='(async()=>app.vault.getMarkdownFiles().filter(f=>(app.metadataCache.getFileCache(f)?.frontmatter||{}).type==="snapshot").map(f=>f.path).join("\n"))()'
```

`search` also requires the named parameter: a positional `search "term"` fails with `Missing required parameter: query=<text>`.

### `create`: `path=` alone, never `name=` with a slash

`name=` takes a bare note name and rejects any `/` with `name cannot contain "/". Use path for a full file path.` Passing **both** `name=` and `path=` mints a duplicate note. For anything in a folder, pass `path=` only.

### `read` resolves fuzzily and can return a different note

Beyond the positional-argument trap, `read` with an explicit `file=` or `path=` has still returned a *different* note whose name shares a long prefix with the requested one. Reads also intermittently hang past 90s.

Never treat a single `read` as verification. Cross-check with `files`, an `eval` over the metadata cache, or `qmd` once reindexed. Identical content returned for two different filenames is a calling-convention bug until proven otherwise, not duplicate notes.

### Large payloads: `eval` + `readFileSync`, not chunked append

`create` with a large `content=` argument fails on big payloads — observed hanging at 54 KB and exiting 143/144 at ~56 KB with the file left untouched, while 2.5 KB writes through the identical command succeeded minutes earlier. Chunked `create` + `append` works but is fragile and leaves a partial note if a chunk fails.

The reliable path keeps argv small and does the read inside the app:

```bash
obsidian vault="My Vault" eval code='(async () => {
  const fs = require("fs");
  const text = fs.readFileSync("/path/to/staged.md", "utf8");
  const f = app.vault.getAbstractFileByPath("Folder/Note.md");
  if (!f) return "note not found";
  await app.vault.modify(f, text);
  return "written " + text.length + " chars";
})()'
```

Use `app.vault.adapter.write(path, text)` instead of `modify` for dot-prefixed paths, which `getAbstractFileByPath` cannot see. Both return a char count, so the write is self-verifying. The returned char count sits **below** the file's byte count when the note holds em-dashes, `×` or arrows: that is UTF-8, not truncation.

### `eval`: top-level `return` is as illegal as top-level `await`

Two distinct errors, one cause. `await is only valid in async functions` and `Illegal return statement` both mean the body was not wrapped. The IIFE must be an expression that is *invoked*, with no leading `return`:

```js
(async () => { ... })()      // correct
return (async () => {...})() // Illegal return statement
```

### `processFrontMatter` resolves before the write reaches disk

`await app.fileManager.processFrontMatter(f, cb)` returns once the callback has run, not once the file is written. A verification read issued in the same `eval` reports the **old** content, through `metadataCache`, `cachedRead` and `adapter.read` alike. Observed 2026-08-08: a `type`/`date` repair read back as `undefined` immediately, and as correct a few seconds later in a separate call.

This misreads as a silent no-op and invites a pointless retry. Verify in a **separate CLI call**, or `await new Promise(r => setTimeout(r, 500))` before reading back. The same applies to bulk passes: check the resulting counts afterwards rather than inside the writing `eval`.

Note also that `processFrontMatter` appends new keys to the **end** of the block, so a repaired `type:` will not appear where the schema lists it. Match on the whole frontmatter block, never on the first N characters.

### Concurrent Claude Code sessions contend on the single-instance lock

A second session running its own `obsidian eval` makes this session's call time out, and the error output may echo back *the other session's* command. An `eval` may also complete successfully while returning no visible output.

A failed or silent CLI call is not proof the vault is broken. Check `pgrep -fl "obsidian vault"` for another session's invocation before diagnosing anything else, and verify the write landed by reading it back rather than trusting the return value.

### Dataview index desync after external writes

After modifying a Dataview-indexed file via `app.vault.modify` or `adapter.write`, Dataview's cached page row stays stale even though disk and `app.metadataCache` are correct. `dv.index.reload`, no-op modifies, and delete+create do **not** fix it. Only `Cmd+P → Reload app without saving` does.

For verification, read `app.metadataCache.getFileCache(f).frontmatter` rather than `dv.pages()`. If a Bases or Dataview view looks wrong after an external write, ask the user to reload before chasing phantom bugs in the query.
