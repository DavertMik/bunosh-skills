# Bunosh Skills

Agent skills for working with [Bunosh](https://buno.sh) — a task runner that turns
JavaScript functions into CLI commands.

| Skill | Use it when |
|-------|-------------|
| [`bunosh-fundamentals`](./bunosh-fundamentals) | Writing, editing, or debugging a `Bunoshfile.js`; you need to know how functions become commands, how args/options map, or how tasks behave. |
| [`migrate-to-bunosh`](./migrate-to-bunosh) | Converting existing bash scripts, npm/package.json scripts, Makefiles, or Node.js scripts into a single `Bunoshfile.js`. |

## Installation

These skills are plain folders containing a `SKILL.md`. Install them by copying
into a directory the agent reads skills from.

**Claude Code (project-level):**

```bash
git clone <this-repo> bunosh-skills
cp -r bunosh-skills/bunosh-fundamentals  .claude/skills/
cp -r bunosh-skills/migrate-to-bunosh    .claude/skills/
```

**Claude Code (user-level, available in every project):**

```bash
cp -r bunosh-skills/* ~/.claude/skills/
```

Restart the session (or run `/doctor`) so the skills are picked up.

## Layout

```
skills/
├── bunosh-fundamentals/
│   └── SKILL.md
└── migrate-to-bunosh/
    ├── SKILL.md
    ├── references/
    │   └── conversion-cheatsheet.md
    └── evals/
        └── evals.json
```

`migrate-to-bunosh` depends conceptually on `bunosh-fundamentals` — install both.
