# nuxt-workspace

Claude Code plugin for working in Nuxt workspaces: skills plus a set of scoped coding rules.

## Layout

```
.claude-plugin/plugin.json        plugin manifest
.claude-plugin/marketplace.json   marketplace listing
skills/                           skills (interview, write-tests, quickfix)
rules/                            scoped coding rules (copied into your project, see below)
agents/                           subagent definitions (*.md)
hooks/hooks.json                  hook config
```

## Install

```
/plugin marketplace add theCodeD3vil/nuxt-workspace
/plugin install nuxt-workspace@nuxt-workspace
```

This installs the skills. Plugins do not load `rules/`, so copy the rules into your project (next step).

## Install the rules

Copy `rules/` into your project's `.claude/rules/`. Claude Code loads rules from there, and each rule only applies to files that match its `paths:` globs.

From the root of your Nuxt project:

```bash
npx degit theCodeD3vil/nuxt-workspace/rules .claude/rules
```

Or, from a clone of this repo:

```bash
git clone https://github.com/theCodeD3vil/nuxt-workspace /tmp/nuxt-workspace
mkdir -p .claude/rules
cp -R /tmp/nuxt-workspace/rules/. .claude/rules/
```

- Copy the whole folder. The rules link to each other (for example `errors.md`, `../css.md`), so keep the structure.
- `general.md` and `errors.md` have no `paths:`, so they load in every session (about 3.4k tokens). The others load only when matching files are touched.
- Re-run the copy to update. It overwrites your local edits, so keep project-specific changes in separate files.

## Skills

| Skill | Use |
|---|---|
| `interview` | Clarify requirements with focused questions before starting work. |
| `write-tests` | Write Vitest and `@nuxt/test-utils` tests for critical paths. |
| `quickfix` | Make a small, targeted fix with minimal changes. |

## Develop

```bash
claude --plugin-dir .
claude plugin validate .
```
