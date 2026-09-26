# claude-harness

The **big-damn-heroes** Claude Code plugin marketplace.

## Plugins

### dev-harness

General software design and development practices — how to design good software anywhere. Built to sit alongside org-specific plugins: where project or org guidance prescribes specifics, it wins.

| Component | Type | Use |
|---|---|---|
| `box-design` | skill | Decompose systems into boxes with explicit interface, error, and test contracts. Auto-triggers when designing or planning. `/box-design verify` audits a plan or codebase; `/box-design levels` pitches a design at the right altitude. The bare name works unless another command is also named `box-design`; `/dev-harness:box-design` always works. |
| `design-auditor` | agent | Read-only composability audit of a plan, diff, or repo in an isolated context. Returns findings only. |

## Install

The marketplace installs straight from this GitHub repo. No clone is needed.

### Option 1 — commands (recommended)

From your shell:

```sh
claude plugin marketplace add deankarn/claude-harness   # GitHub owner/repo shorthand
claude plugin install dev-harness@big-damn-heroes
```

Or inside a session:

```
/plugin marketplace add deankarn/claude-harness
/plugin install dev-harness@big-damn-heroes
```

The full URL works too: `claude plugin marketplace add https://github.com/deankarn/claude-harness.git`.

Then turn on auto-update: `/plugin` → **Marketplaces** → `big-damn-heroes` → **Enable auto-update**. Without it, you only get updates when you pull them yourself (see below). The install commands can't turn auto-update on, and a marketplace can't default it on, so this toggle is a required step.

### Option 2 — settings file

Useful if you sync dotfiles and want new machines set up with no commands. Add this to `~/.claude/settings.json`, merging with any existing keys:

```json
{
  "extraKnownMarketplaces": {
    "big-damn-heroes": {
      "source": { "source": "github", "repo": "deankarn/claude-harness" },
      "autoUpdate": true
    }
  },
  "enabledPlugins": {
    "dev-harness@big-damn-heroes": true
  },
  "permissions": {
    "allow": ["Skill(dev-harness:box-design)"]
  }
}
```

Start Claude Code. It clones the marketplace from GitHub in the background and installs `dev-harness`, then shows `Plugins changed. Run /reload-plugins to activate.` From then on it keeps the plugin updated.

### How updates arrive

The plugin has no `version`, so every commit to `main` counts as a new version. With auto-update on, Claude Code checks for changes within about 10 minutes after the first message of an interactive session. It updates the plugin on disk and prompts `Run /reload-plugins to apply`; the next launch picks it up either way.

To update immediately:

```sh
claude plugin marketplace update big-damn-heroes
claude plugin update dev-harness@big-damn-heroes
```

Setting `DISABLE_AUTOUPDATER`, `DISABLE_UPDATES`, or `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` turns auto-update off, unless `FORCE_AUTOUPDATE_PLUGINS=1` is also set.

### Migrating from a standalone skill

If you previously installed `box-design` as a standalone skill, remove `~/.claude/skills/box-design`. Otherwise it loads twice and takes the bare `/box-design` name.

### Permissions

`design-auditor` loads its checklist by invoking the `box-design` skill, which prompts once. Approve the prompt, or add `"Skill(dev-harness:box-design)"` to `permissions.allow` in `~/.claude/settings.json`. Option 2 already includes it.

### Disable in a specific repo

Add to that repo's `.claude/settings.local.json`:

```json
{ "enabledPlugins": { "dev-harness@big-damn-heroes": false } }
```

## Develop

```sh
claude --plugin-dir ./plugins/dev-harness   # shadows the installed copy for this session
claude plugin validate .
```

Run `/reload-plugins` in the session after editing. See [CLAUDE.md](CLAUDE.md) for conventions.

## License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or <http://www.apache.org/licenses/LICENSE-2.0>)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or <http://opensource.org/licenses/MIT>)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the work by you, as defined in the Apache-2.0 license, shall be dual licensed as above, without any additional terms or conditions.
