# cbulloss-skills — repo guide

## Mission

A growing collection of Claude skills covering:
- **Dev tooling** — review workflows, repo automation, productivity helpers.
- **PM/dev/review workflows in the AI era** — patterns and templates for working with Claude on planning, specs, code review, and similar.
- **Personal automation** — browser-driven skills for everyday tasks (e.g., grocery shopping).

Skills here are designed for two distribution paths:
1. **Plugin install** — `/plugin install <git-url>` makes every skill discoverable.
2. **Copy-out** — Each `skills/<name>/` directory is fully self-contained, so it can be lifted into another repo's `.claude/skills/` (project) or `~/.claude/skills/` (user) without modification.

## Repo conventions

### One skill per directory
- Path: `skills/<kebab-case-name>/`.
- Required: `SKILL.md`. Recommended: standalone `README.md` so the directory still makes sense after being copied out.
- Optional: `schemas/`, `examples/`, `scripts/` — all inside the skill's own directory. **No cross-skill imports.** Skills must not depend on files in other skills.

### `SKILL.md` frontmatter

```yaml
---
name: <kebab-case-name>           # must match directory name
description: <one paragraph>      # MUST start with or include "use when ..." so dispatch is reliable
user-invocable: true              # set if the skill should be invokable as /<name>
allowed-tools:                    # optional whitelist
  - Read
  - Write
  - Bash(git *)
  - mcp__playwright__*
---
```

The `description` is what other agents (and Claude itself) read to decide whether to invoke the skill. Lead with the trigger condition, not a feature description.

### External state lives outside the repo
Anything stateful — browser profiles, auth cookies, cached scrapes, credentials — goes under `~/.claude/skills-data/<skill-name>/`. **Never** commit state into this repo, and never write state inside the skill directory. This is what keeps skills copy-friendly and the repo clean.

`.gitignore` blocks `skills-data/` as a safety net in case someone symlinks state in by mistake.

### Hands-off discipline (for automation skills)

Skills that perform autonomous work (browser automation, batch operations, etc.) MUST follow this contract:
1. Gather **all** clarifying input from the user UPFRONT, in a single batch.
2. Run to completion without further prompts.
3. Report results at the end.

Mid-task `AskUserQuestion` calls break the user contract — the whole point of these skills is that the user can walk away. If a skill genuinely cannot proceed mid-run (captcha, expired session), it must FAIL with a clear actionable message, not silently wait for input.

If a skill deviates from this rule, it must document the deviation explicitly in its `SKILL.md` under a "Known interruption points" heading.

### Required MCP servers
Each `SKILL.md` lists the MCP servers it needs (e.g., `playwright`) and how to install them. Skills should fail fast at the top with a clear error if a required MCP is missing — don't try to limp along.

## Adding a new skill

1. Create `skills/<kebab-case-name>/`.
2. Write `SKILL.md` with proper frontmatter.
3. Write a standalone `README.md` for the skill directory.
4. Add a one-line entry to the repo `README.md` skill index.
5. Commit. Bump `.claude-plugin/plugin.json` `version` if this is a release.

## Distribution

**As a plugin:**
```
/plugin install <git-url-or-local-path>
```
All skills become available. Updates pull on the next install.

**As a single skill (copy-out):**
```
cp -r skills/<name>/ <target-repo>/.claude/skills/<name>/
# or for user-scope across all projects:
cp -r skills/<name>/ ~/.claude/skills/<name>/
```
The skill is self-contained, so this just works — no edits required.

## What does NOT belong here
- Project-specific code or one-off scripts.
- Anything tied to credentials or private data.
- Skills that depend on external repos at fixed paths.
- Generated artifacts or build output.
