# claude-harness

Claude Code marketplace `big-damn-heroes`. Each plugin lives in `plugins/<name>/` and is listed in `.claude-plugin/marketplace.json`.

## Conventions

- **Skills hold knowledge and rules. Agents hold roles.** An agent that needs rules invokes the skill (`Skill` in its `tools`, `dev-harness:<skill> <mode>` in its body) — never copy rules into an agent body. Every rule lives in exactly one place.
- Prefer invoking over `skills:` preloading when the skill reads its own `references/`: the skill's `allowed-tools` grants those reads, which a subagent otherwise lacks (plugin files sit outside the working directory).
- Skills are general-purpose design and development practice. They must defer to project/org guidance on specifics (see the Precedence section in `box-design`), since they are installed alongside org-specific marketplaces.
- Heavy reference material goes in a skill's `references/` and is loaded on demand; keep `SKILL.md` focused.
- Reference bundled files with `${CLAUDE_SKILL_DIR}` in skills and `${CLAUDE_PLUGIN_ROOT}` in agents. Never hardcode paths.
- Plugin agents ignore `hooks`, `mcpServers`, and `permissionMode` frontmatter.
- **Never set `version`** in `plugin.json` or `marketplace.json`. The commit SHA is the version, so every push to `main` reaches installed machines.
- Entry `name` in `marketplace.json` must match `name` in the plugin's `plugin.json`.
- Don't add empty `hooks/hooks.json` or `.mcp.json` placeholders.
- `docs/` is source material and design intent. It is not shipped with any plugin.

## Checks

```sh
claude plugin validate .   # the "No version specified" warning is expected
```

Test changes without installing: `claude --plugin-dir ./plugins/dev-harness`, then `/reload-plugins` after edits.
