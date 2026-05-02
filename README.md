# cbulloss-skills

A collection of Claude skills for dev tooling, PM/dev/review workflows, and personal automation. Each skill is self-contained and can be installed all at once via the Claude Code plugin system, or copied individually into another project.

## Install

### As a plugin (all skills)

In any Claude Code session:
```
/plugin install <git-url-or-local-path-to-this-repo>
```
The skills below become discoverable to Claude immediately.

### As a single skill (copy one out)

Each `skills/<name>/` directory is self-contained. To use just one skill in another project:
```bash
cp -r skills/<name>/ <other-project>/.claude/skills/<name>/
# or for user-scope (available in every project):
cp -r skills/<name>/ ~/.claude/skills/<name>/
```

## Skill index

| Skill | Purpose | Requires |
|---|---|---|
| [`instacart-shopper`](skills/instacart-shopper/) | Build an Instacart cart from a grocery list (text, markdown, or JSON). Hands-off after upfront clarification. | `playwright` MCP server |

## Conventions for contributors

See [`CLAUDE.md`](CLAUDE.md) for the full conventions. Highlights:

- One skill per directory under `skills/<kebab-case-name>/`.
- `SKILL.md` is required; `README.md` per skill is strongly recommended so a copied directory still documents itself.
- Skill directories must be **self-contained** — no cross-skill imports.
- Stateful runtime data (browser profiles, caches, credentials) lives at `~/.claude/skills-data/<skill-name>/`, never in the repo.
- Automation skills must be **hands-off**: gather all clarifying input upfront, then run to completion.

## Adding a new skill

1. `mkdir skills/<kebab-case-name>` and write `SKILL.md` with proper frontmatter (see `CLAUDE.md`).
2. Add a one-line row to the skill index above.
3. Bump `.claude-plugin/plugin.json` `version` if releasing.
4. Commit.

## License

[MIT](LICENSE).
