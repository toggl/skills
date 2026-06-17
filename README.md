# Toggl Skills

Public [Agent Skills](https://skills.sh) published by Toggl. Each skill is a
reusable instruction set that extends a coding agent (Claude Code, Cursor, Cline,
and 70+ others) with Toggl-specific capabilities.

## Install

Install every skill in this repo:

```sh
npx skills add toggl/skills
```

The CLI detects which agents you have installed and lets you pick where each skill
lands (it symlinks into e.g. `~/.claude/skills/` by default).

## Available skills

| Skill | What it does |
|-------|--------------|
| [`toggl-browser-integrations`](skills/toggl-browser-integrations/SKILL.md) | Author, post, and verify a Toggl 2.0 custom website integration (the in-page "Track" button) — build the integration JSON, find anchor/resolver CSS selectors, inject it on a live page, and confirm the button landed correctly. |

## Repository layout

```
skills/
  <skill-name>/
    SKILL.md          # YAML frontmatter (name, description) + instructions
```

Each skill lives in its own directory under `skills/`. The `skills` CLI walks
this directory one level deep, so adding a new skill is just a new folder with a
`SKILL.md`.

## Publishing a new skill

1. Create `skills/<skill-name>/SKILL.md` with valid frontmatter (`name`,
   `description` are required).
2. Add a row to the table above.
3. Open a PR. Once merged to the default branch, it's installable immediately —
   there is no separate publish step; `npx skills add toggl/skills` clones this
   repo at install time.

## License

TODO: add a license before announcing publicly (e.g. MIT) — confirm with Toggl
legal/ownership. Until then the default is "all rights reserved".
