---
name: bunosh-fundamentals
description: >-
  Core principles for authoring Bunosh task files. Use this skill whenever the
  user is writing, editing, reviewing, or debugging a Bunoshfile.js (or
  Bunoshfile.<namespace>.js), runs `bunosh <command>`, asks why a bunosh command
  isn't showing up or its arguments/options aren't parsed as expected, or wants
  to know how exported functions become CLI commands. Trigger this even when the
  user just says "add a task", "make a bunosh command", or pastes a Bunoshfile —
  Bunosh has non-obvious naming and argument rules that are easy to get wrong
  without this skill.
---

# Bunosh Fundamentals

Bunosh turns plain JavaScript functions into CLI commands. There is no DSL: a
`Bunoshfile.js` is an ES module, every exported function becomes a command, and
its parameters become arguments and options. Built-in helpers (`shell`, `fetch`,
`say`, `ask`, `task`, ...) are available as globals via `global.bunosh`.

The rules below are the ones that are *not* guessable. Get these right and the
rest is ordinary JavaScript.

## 1. Functions → commands (naming is non-obvious)

The command name is derived from the function name by splitting at the **first**
capital letter. The first segment becomes a namespace, the rest is kebab-cased
after a single colon. There is never more than one colon.

| Function | Command | Notes |
|----------|---------|-------|
| `build` | `bunosh build` | single word → lowercased |
| `deployApp` | `bunosh deploy:app` | first cap → `:` |
| `gitPush` | `bunosh git:push` | |
| `npmInstall` | `bunosh npm:install` | |
| `buildAndDeploy` | `bunosh build:and-deploy` | only first cap is the `:`, rest is `-` |
| `dbMigrateReset` | `bunosh db:migrate-reset` | |

Implication: to put several commands under one namespace, **prefix the function
names with the same first word** (`dbMigrate`, `dbSeed`, `dbReset` →
`db:migrate`, `db:seed`, `db:reset`). Only `export`ed functions become commands;
helpers can be plain non-exported functions in the same file.

## 2. Parameters → arguments and options

Positional parameters become CLI arguments. The presence and kind of a default
value decides whether the argument is required:

```js
export async function deploy(env, tag = 'latest', region = null) {}
```

| Parameter form | CLI behaviour |
|----------------|---------------|
| no default (`env`) | **required** argument: `bunosh deploy <env>` |
| default value (`tag = 'latest'`) | optional, falls back to the default |
| `= null` | optional, no default shown |

The **last** parameter, if it defaults to an object literal, becomes
`--options`. Each key is a flag:

```js
export async function deploy(env, options = { force: false, replicas: 3, tag: null }) {}
```

| Option default | CLI form | In function |
|----------------|----------|-------------|
| `false` or `null` | boolean flag `--force` | `options.force === true` when passed |
| any other value | `--replicas [value]` (default kept) | `options.replicas` is a string when passed on CLI |

Flag names are dasherized and map back to camelCase: CLI `--dry-run` →
`options.dryRun`. Values passed on the command line arrive as **strings** —
coerce them (`Number(options.replicas)`) when you need a number.

Usage of the example above:

```bash
bunosh deploy production --force --replicas 5 --tag v1.2.3
```

## 3. JSDoc → help text

The block comment directly above the function is its description. The first line
is shown in the command list; the whole comment shows in `bunosh <cmd> --help`.
Document params with `@param` so the generated help is useful:

```js
/**
 * Deploy the app to an environment.
 * @param {string} env - Target environment (staging|production)
 * @param {object} options
 * @param {boolean} [options.force=false] - Skip confirmation
 */
export async function deploy(env, options = { force: false }) {}
```

## 4. Built-in tasks

Pull helpers from `global.bunosh` at the top of the file (globals are used
instead of imports so the single-file binary works everywhere):

```js
const { shell, fetch, writeToFile, copyFile, task, ai } = global.bunosh;
const { say, ask, yell } = global.bunosh;
```

### `shell` — run commands (use this for everything shell)

`shell` is a tagged template. It streams output live, works cross-platform on
Bun, and falls back to Node `child_process` off Bun. `exec` and `$` are
**deprecated aliases of `shell`** kept for backward compatibility — prefer
`shell` in new code.

```js
await shell`npm ci`;
await shell`
  npm run build
  npm run bundle
`.env({ NODE_ENV: 'production' }).cwd('/srv/app');
```

`.env(obj)` and `.cwd(path)` are chainable.

### TaskResult — tasks don't throw

Every `shell`/`fetch`/`task` call resolves to a `TaskResult`. **It never throws
on a failed command** — you inspect it:

```js
const res = await shell`npm test`;
res.status        // 'success' | 'fail' | 'warning'
res.output        // combined output / returned value
res.hasFailed     // true when status === 'fail'
res.hasSucceeded
res.hasWarning
await res.json()  // shell: { stdout, stderr, exitCode, lines } · fetch: parsed JSON body
```

This is the single biggest behavioural difference from bash/node scripts: a
failing command does **not** abort the function by itself. See §6.

### `fetch`, file ops, `ai`

`fetch` returns a **`TaskResult`**, not a DOM `Response`. There is no `.ok`
or numeric `.status` on it — use `res.hasFailed` (a non-2xx response resolves
to a failed `TaskResult`). Read the body with **`await res.json()`** (parses
the JSON body) or `res.text()`; `res.output` is the raw streamed body as a
string. Do **not** call `.json()` on a raw `Response` you fetched yourself
through the helper — the body is consumed while streaming output.

```js
const r = await fetch('https://api.example.com/health');
if (r.hasFailed) { yell(`down: ${r.output}`); return; }
const body = await r.json();          // parsed JSON; res.output is the raw text

writeToFile('CHANGELOG.md', (line) => {
  line`# v${version}`;
  line``;
  line.fromFile('CHANGELOG.md');   // append previous contents
});

copyFile('template.env', '.env');

const out = await ai('Summarize these commits: ' + log.output, {
  summary: 'one paragraph',
  breaking: 'list of breaking changes',
});                                 // returns an object keyed by your schema
```

`ai` needs `AI_MODEL` plus a provider key (`OPENAI_API_KEY` /
`ANTHROPIC_API_KEY` / `GROQ_API_KEY`) in the environment.

### I/O

- `say(...)` — normal output.
- `yell(text)` — big ASCII-art banner for the one message that matters.
- `ask(question, default?, options?)` — prompt. Smart by type:
  - `ask('Name?', 'app')` → text with default
  - `ask('Proceed?', true)` → yes/no
  - `ask('Env?', ['dev','prod'])` → single select
  - `ask('Features?', [...], { multiple: true })` → multi select
  - `ask('Password?', { type: 'password' })`, `{ editor: true }`

### It's just an ES module — use real JavaScript

The built-in helpers cover the common cases, but a `Bunoshfile.js` is a normal
ES module. This matters most when rewriting `.js`/`.cjs`/`.mjs` scripts: you
don't have to shell out for everything.

- `import` any npm package or Node builtin: `import { S3 } from '@aws-sdk/client-s3'`,
  `import { readFile } from 'node:fs/promises'`, `import path from 'node:path'`.
- Under Bun, the `Bun` APIs are available: `Bun.file(p).text()/.json()/.exists()`,
  `await Bun.write(p, data)` (also copies: `Bun.write(dst, Bun.file(src))`).
- Define plain non-exported helper functions and module-level `const`s; only
  `export`ed functions become commands.
- Ordinary JS control flow, `async`/`await`, `Promise.all`, array methods —
  prefer these over piping through `shell`. Reach for `` shell`...` `` to invoke
  external tools, not to do logic.

Read external state with `Bun.file`/`fs`; parse with `JSON.parse` or
`await result.json()`; transform with `.map`/`.filter`. This is what makes the
JS version shorter and clearer than the bash/node original.

## 5. `task` — group and label work

`task(name, fn)` wraps work in a named, tracked unit. Use it to give a noisy
sequence one clear label and one success/failure line; nested tasks attach to
their parent.

```js
await task('Build image', async () => {
  await shell`docker build -t app .`;
  await shell`docker push app`;
});
```

`task.try(fn)` (or `task.try(name, fn)`) runs silently and returns a boolean —
ideal for probes:

```js
const dbUp = await task.try(() => shell`nc -z localhost 5432`);
if (!dbUp) { say('db down, using fallback'); }
```

Output control: `task.silent(fn)` runs one task without output;
`task.silence()` / `task.prints()` toggle output globally.

Run independent work in parallel with `Promise.all`:

```js
await Promise.all([
  shell`npm run build:web`,
  shell`npm run build:api`,
]);
```

## 6. Failure & exit-code model (read this before "fixing" error handling)

Because tasks don't throw, error handling is **result inspection + early
return**, not try/catch and not `process.exit`:

```js
const build = await shell`npm run build`;
if (build.hasFailed) { yell('build failed'); return; }
await shell`npm run deploy`;
```

Mode controls:

- **Default**: a failed task is recorded; execution **continues**; the process
  exits `1` at the end if anything failed.
- `task.stopOnFailures()` — abort immediately with exit `1` on the first
  failure. This is the "behave like `set -e` bash / a normal node script" knob;
  call it at the top of a function that should be all-or-nothing.
- `task.ignoreFailures()` — continue and exit `0` regardless. Good for
  best-effort cleanup commands.
- Under a test runner / `NODE_ENV=test`, exit code is forced to `0`.

`return` early to stop a command; never call `process.exit()` in a Bunoshfile —
let Bunosh own the exit code.

## 7. Project layout & invocation

- Default file: `Bunoshfile.js` in the working directory.
- `Bunoshfile.<ns>.js` registers **every exported function** in that file under
  the `ns:` namespace. The filename supplies the namespace, so name the
  functions plainly — don't prefix them again:
  - `Bunoshfile.db.js` with `export function migrate()` → `bunosh db:migrate`,
    `export function reset()` → `bunosh db:reset`.
  - Use single-word / simple function names here. A camelCase name in a
    namespaced file is still kebab-split and can drop its first word
    unexpectedly (`migrateReset` in `Bunoshfile.db.js` → `db:reset`, not
    `db:migrate-reset`). One word per function avoids the surprise.
  Use this to split a large `Bunoshfile.js` into per-area files.
- `bunosh` with no args lists all commands. `bunosh <cmd> --help` shows details.
- `bunosh --bunoshfile path/to/File.js <cmd>` or `BUNOSHFILE=...` to pick a file.
- `bunosh init` scaffolds a starter file; `bunosh edit` opens it;
  `bunosh export:scripts` mirrors commands into `package.json` `scripts`.
- `bunosh -e "say('hi')"` (or heredoc / stdin) runs inline JavaScript with all
  globals available — handy in CI without a committed file.

## Authoring conventions

When writing or editing a Bunoshfile, produce code that matches how Bunosh code
is written:

- No comments unless the user asks for them — function names and JSDoc carry
  intent.
- Prefer early `return` over `if/else` nesting (Bunosh's failure model is built
  around early returns).
- One exported function per logical command; factor shared logic into
  non-exported helpers.
- **No pass-through wrappers — commands are just functions, call them
  directly.** Don't create a `fooRun(...)`/`doFoo(...)` twin whose only job is
  to be called by the exported `foo` and by some other function. An exported
  command is an ordinary async function: call `foo()` straight from another
  command or helper. Only split out a helper when it has *its own* distinct
  responsibility — not to dodge "calling a command". This removes a whole layer
  of boilerplate and keeps the file compact.

  ```js
  // ❌ wrapper twin: releasePullRun exists only so two callers can reach it
  async function prepareContent() {
    await docsSyncRun(false);
    await docsUnifiedApiRun(false);
    await releasePullRun(5, true);
  }
  export async function releasePull(pages = 5) {
    stopOnFailures();
    await releasePullRun(Math.max(1, Number(pages) || 5), false);
  }
  async function releasePullRun(pages, soft) { /* real work */ }

  // ✅ the command IS the function — call it directly
  async function prepareContent() {
    await docsSync();
    await docsUnifiedApi();
    await releasePull();
  }
  /**
   * Pull releases from the CodeceptJS GitHub repo into release.md.
   * @param {number} [pages=5] - Pages to fetch (3 releases per page).
   */
  export async function releasePull(pages = 5) {
    stopOnFailures();
    const count = Math.max(1, Number(pages) || 5);
    await task(`Fetch releases from ${RELEASES_REPO}`, async () => {
      releases = await fetchReleases(count);
    });
  }
  ```
- **Commands at the top, helpers at the bottom — rely on hoisting.** Keep each
  exported command short and high-level — it should read as the *what*. Push the
  *how* (complex steps, parsing, retries) into non-exported helpers placed at
  the **very end** of the file, below all the commands. The top of the file then
  reads as a table of contents of what the project can do. Write those helpers
  as `function` **declarations**, not `const fn = () => {}` arrow expressions:
  declarations are hoisted, so commands at the top can call helpers defined at
  the bottom regardless of order. Arrow/`const` helpers are *not* hoisted and
  must appear before their first use, which forces helpers above commands and
  breaks the commands-first layout.

  ```js
  // top: exported commands, calling helpers defined far below
  export async function release(version) {
    const notes = await collectNotes(version);   // ok: hoisted declaration
    await publish(version, notes);
  }

  // bottom: non-exported helper declarations
  async function collectNotes(version) { /* ... */ }
  async function publish(version, notes) { /* ... */ }
  ```
- Add a JSDoc block to every exported function so `--help` is meaningful.
- Reach for `task()` to label multi-step sequences, not single commands.
- **Split when it gets long.** Past ~1000 lines, a single `Bunoshfile.js` stops
  being readable. Move cohesive groups into `Bunoshfile.<ns>.js` files (one per
  area: `Bunoshfile.db.js`, `Bunoshfile.deploy.js`, ...). Each file keeps the
  commands-top/helpers-bottom shape; commands there are automatically namespaced
  `ns:`. This is the primary tool for keeping a large task suite navigable.
