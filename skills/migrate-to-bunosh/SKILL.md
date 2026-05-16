---
name: migrate-to-bunosh
description: >-
  Convert existing automation — bash/sh scripts, package.json npm/yarn/pnpm
  scripts, Makefile targets, Justfiles, or standalone Node.js scripts — into a
  single Bunosh `Bunoshfile.js`. Use this skill whenever the user wants to
  migrate, port, consolidate, or "move to bunosh" their scripts; mentions
  replacing a pile of shell scripts or npm scripts with bunosh; asks to turn a
  Makefile/Justfile into bunosh commands; or hands over one or more scripts and
  asks for the bunosh equivalent. Trigger it even for a single script — the
  output is always one consolidated Bunoshfile.
---

# Migrate to Bunosh

Goal: collapse scattered automation (shell scripts, `package.json` scripts,
Makefile/Justfile targets, node scripts) into **one `Bunoshfile.js`** where each
former script is an exported function = a `bunosh` command.

This skill assumes the naming/argument/failure rules from the
`bunosh-fundamentals` skill. If that skill's content isn't already in context,
read it first — the conversion below depends on it (especially the
function-name → command mapping and the "tasks don't throw" model).

## Workflow

Follow these steps in order. Don't skip the inventory — picking command names
and namespaces well is the part users care about most.

### 1. Inventory the sources

Find everything that should become a command:

- `package.json` → `scripts` block
- `*.sh`, `scripts/`, `bin/`, `tools/` shell scripts
- `Makefile` / `makefile` targets, `Justfile`
- standalone `*.js` / `*.mjs` automation scripts
- CI files (`.github/workflows`, `.gitlab-ci.yml`) — these often reveal the
  *real* entry points and the env vars each step needs

For each, note: what it does, its arguments/flags, env vars it reads, its
working directory, and whether it must abort on first error.

If there are many sources, briefly list the planned commands and namespaces and
confirm with the user before writing the whole file — renaming later is cheap
but agreeing up front avoids churn.

### 2. Design the command surface

Map each source to a function name, remembering Bunosh's first-capital-splits
rule:

- Group related scripts under a shared first word so they share a namespace:
  `db:migrate`, `db:seed`, `db:reset` come from `dbMigrate`, `dbSeed`,
  `dbReset`.
- npm scripts usually map 1:1 (`"build"` → `export function build()`).
- A script that takes positional args → function parameters; a script parsing
  `--flags` (getopts / `process.argv` / `minimist`) → an `options = {}` object
  as the last parameter.
- Very large task sets: split into `Bunoshfile.<ns>.js` files per namespace.

### 3. Translate each script body

Rewrite the logic as JavaScript. The detailed source-construct → bunosh
mapping (bash conditionals/loops/`set -e`/`$?`, curl, jq, fs, `child_process`,
npm script chaining, Makefile `.PHONY`, etc.) lives in:

**`references/conversion-cheatsheet.md`** — read it before translating; it has
the line-by-line equivalents and the common mistakes.

Core principles while translating:

- All shell goes in `` shell`...` `` template literals (multiline is fine).
  `.env({...})` and `.cwd(path)` replace `export VAR=` and `cd`.
- Tasks **don't throw**. Replace `set -e` / `&&` chaining /
  `if [ $? -ne 0 ]` / try-catch-exit with either `task.stopOnFailures()` at the
  top of the function (the faithful "abort on first error" port) or explicit
  `if (res.hasFailed) return;` checks.
- Replace `exit N` / `process.exit` with early `return`. Bunosh owns the exit
  code.
- `echo` → `say()`; prominent banners → `yell()`; prompts → `await ask()`.
- `curl` → `fetch()`; reading/writing files → `Bun.file()` / `Bun.write()` or
  `writeToFile()`; `cp` → `copyFile()`.
- Preserve behaviour, don't "improve" it silently. If the original aborts on
  error, the port must too (`task.stopOnFailures()`).

### 4. Assemble one Bunoshfile.js

Structure:

```js
const { shell, fetch, writeToFile, copyFile, task, ai } = global.bunosh;
const { say, ask, yell } = global.bunosh;

/**
 * <first line of help, from what the original script did>
 * @param {string} env - ...
 */
export async function deploy(env = 'staging', options = { force: false }) {
  task.stopOnFailures();            // only if the original aborted on error
  // ...translated body...
}
```

- One exported function per former script.
- JSDoc on every function (first line → command list, full block → `--help`).
  Reconstruct it from comments/usage text in the original script.
- Shared logic → non-exported helper functions in the same file.
- No explanatory code comments unless the user asks — JSDoc + names carry
  intent. Prefer early `return` over `if/else` nesting.

### 5. Verify

Don't claim success without checking the file loads and the commands register:

```bash
bunosh                 # lists all generated commands — every source accounted for?
bunosh <command> --help   # args/options/description correct?
```

A syntax error or a non-object last parameter will show up here. Spot-check one
representative command if it's safe to run (dry-run / `--help` only for
destructive ones).

### 6. Wire it up & report

- Offer to run `bunosh export:scripts` so the old `npm run <x>` calls still work
  (they become `bunosh <x>` under the hood) — useful for a gradual migration.
- Tell the user which old files are now redundant (don't delete them unless
  asked).
- Summarize the command surface: a short table of
  `old script → bunosh command`.

## Output contract

The deliverable is a single valid `Bunoshfile.js` that:

- has one exported function per migrated script,
- registers cleanly (`bunosh` lists them, `--help` works),
- preserves the original error/exit behaviour,
- carries JSDoc-derived help,
- contains no leftover bash/process-exit/try-catch-exit patterns.

Plus a brief mapping table and a note on redundant files. Never delete or
overwrite the original scripts unless the user explicitly asks.
