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

## Why migrate (read this first — it changes how you port)

Migration has two goals, and they decide every judgement call below:

1. **One place.** Every script — bash, `.sh`, `.js`, `.cjs`, `.mjs`, npm
   scripts, Makefile/Justfile targets — ends up as an exported function in a
   single `Bunoshfile.js`. No more hunting through `scripts/`, `bin/`, `tools/`,
   and `package.json` to find what does what.
2. **More readable.** The point is not to *wrap* the old scripts — it's to
   **rewrite them as clean, compact JavaScript** using the Bunosh API. Bash
   gets unreadable fast once there's branching, loops, JSON, or error handling;
   the equivalent JS is shorter and clearer. A migration that just buries a
   50-line bash blob inside one `` shell`...` `` template has failed goal 2.

So the default action is **rewrite into idiomatic JS**, not transliterate.

### The rewrite rule

- **Small/medium scripts (< ~100 LOC), bash or JS:** rewrite the *logic* in
  JavaScript using the Bunosh API. Bash conditionals/loops/`case`/`$?`,
  argument-parsing, `jq`, `sed`/`awk` data wrangling, `curl` → become JS:
  `if`/`for`/`fetch`/object access/array methods. `.js`/`.cjs`/`.mjs` scripts
  are ported to use Bunosh helpers (`shell`, `fetch`, `say`, `ask`, `task`,
  `Bun.file`) instead of `child_process`, `console.log`, `fs`, `node-fetch`.
- **Keep `` shell`...` `` only for what it's good at:** invoking real external
  tools — `git`, `docker`, `kubectl`, `npm`, `rsync`, compilers. The *control
  flow and data handling around* those calls belongs in JS, not in the shell
  string.
- **Large/complex scripts (> ~100 LOC, or heavy domain logic):** still expose
  them as a `bunosh` command, but don't force a risky line-by-line rewrite. Port
  the entrypoint/orchestration to JS and either (a) split the body into
  helper functions in the Bunoshfile, or (b) if it's genuinely big and stable,
  keep it as a script the command calls — and tell the user it was left as-is
  and why. Flag these explicitly rather than silently producing a sketchy port.

Goal: collapse scattered automation into **one `Bunoshfile.js`** where each
former script is an exported function = a `bunosh` command, written so a
newcomer can read it.

This skill assumes the naming/argument/failure rules from the
`bunosh-fundamentals` skill. If that skill's content isn't already in context,
read it first — the conversion below depends on it (the function-name → command
mapping, the "tasks don't throw" model, and the fact that **a Bunoshfile is a
normal ES module**: you can `import` npm packages and use Node/Bun APIs).

## Workflow

Follow these steps in order. Don't skip the inventory — picking command names
and namespaces well is the part users care about most.

### 1. Inventory the sources

Find everything that should become a command:

- `package.json` → `scripts` block
- `*.sh`, `scripts/`, `bin/`, `tools/` shell scripts
- `Makefile` / `makefile` targets, `Justfile`
- standalone `*.js` / `*.cjs` / `*.mjs` automation scripts (note each one's
  rough LOC — it decides rewrite vs keep-as-is, see "The rewrite rule")
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

- **Rewrite logic as JS; reserve `` shell`...` `` for external tools.** Branching,
  loops, arg parsing, JSON/text manipulation → JavaScript. `git`/`docker`/`npm`/
  `kubectl`/`rsync` invocations → `` shell`...` `` with `.env({...})` /
  `.cwd(path)` replacing `export VAR=` / `cd`. The result should read like a
  small program, not a shell script in disguise.
- **Port `.js`/`.cjs`/`.mjs` to the Bunosh API:** `child_process` → `shell`;
  `console.log` → `say`/`yell`; `fs` → `Bun.file()`/`Bun.write()`; `node-fetch`/
  `axios` → global `fetch`; `process.argv` parsing → function params + options.
  It is a normal ES module, so keep using real `import`s for genuine libraries
  (date math, AWS SDK, etc.) — don't reimplement those.
- Tasks **don't throw**. Replace `set -e` / `&&` chaining /
  `if [ $? -ne 0 ]` / try-catch-exit with either `task.stopOnFailures()` at the
  top of the function (the faithful "abort on first error" port) or explicit
  `if (res.hasFailed) return;` checks.
- Replace `exit N` / `process.exit` with early `return`. Bunosh owns the exit
  code.
- `echo` → `say()`; prominent banners → `yell()`; prompts → `await ask()`.
- `curl` → `fetch()`; `jq`/`sed`/`awk` on output → parse in JS
  (`await res.json()`, `.split`, `.map`, `.filter`); `cp` → `copyFile()`.
- Preserve behaviour, don't change what it does — but **do** make it more
  readable: flatten nesting with early returns, name things, drop the bash
  arg-parsing boilerplate. If the original aborts on error, the port must too
  (`task.stopOnFailures()`).
- Compactness check: if a ported function is materially longer or harder to
  follow than the original, you're transliterating, not rewriting — step back
  and express it the way you'd write it fresh in JS.

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

### 6. Export commands back into package.json

This is a standard part of the migration, not an optional extra: a migrated
project should still respond to `npm run <x>` and whatever CI/teammates already
call. Bunosh ships a command for exactly this:

```bash
bunosh export:scripts
```

It rewrites the `scripts` section of `package.json` so every generated command
gets an entry pointing at Bunosh:

```jsonc
{
  "scripts": {
    "build": "bunosh build",
    "test": "bunosh test",
    "ci": "bunosh ci",
    "deploy": "bunosh deploy",
    "backup:db": "bunosh backup:db"
  }
}
```

So `npm run build` (or `yarn build` / `pnpm test`) now dispatches into the
Bunoshfile. Notes and caveats to apply:

- Run it **from the directory containing both `Bunoshfile.js` and
  `package.json`** — it needs both; it errors out otherwise.
- It is destructive to the `scripts` block: it removes prior `bunosh`-prefixed
  entries and merges the regenerated set. Before running it on a project that
  has hand-written non-bunosh scripts, confirm with the user and/or note that
  those entries map 1:1 to the new commands (they were just migrated, so the old
  `node scripts/foo.js` lines are now redundant and being replaced by
  `bunosh foo`).
- If there is no `package.json` (pure shell/Make project), skip this step and
  say so — `export:scripts` only makes sense for npm projects.
- Namespaced commands keep their colon (`backup:db` → `"backup:db": "bunosh
  backup:db"`), which is a valid npm script name and is run as
  `npm run backup:db`.

Run it as part of the migration unless the user opted out, then show the
resulting `scripts` block.

### 7. Report

- Tell the user which old files are now redundant (don't delete them unless
  asked) — old `.sh`, `scripts/*.js`, Makefile targets superseded by commands.
- Summarize the command surface: a short table of
  `old script → bunosh command` (and the matching `npm run` alias if
  `export:scripts` was run).

## Output contract

The deliverable is a single valid `Bunoshfile.js` that:

- has one exported function per migrated script,
- registers cleanly (`bunosh` lists them, `--help` works),
- preserves the original error/exit behaviour,
- carries JSDoc-derived help,
- contains no leftover bash/process-exit/try-catch-exit patterns.

Plus, for npm projects, a `package.json` whose `scripts` were updated via
`bunosh export:scripts` so existing `npm run <x>` / CI invocations keep working.

Plus a brief mapping table and a note on redundant files. Never delete or
overwrite the original scripts unless the user explicitly asks.
