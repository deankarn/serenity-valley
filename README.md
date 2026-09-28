# claude-harness

The **big-damn-heroes** Claude Code plugin marketplace.

## Plugins

### eng

General software design and development practices — how to design good software anywhere. Built to sit alongside org-specific plugins: where project or org guidance prescribes specifics, it wins.

| Component | Type | Use |
|---|---|---|
| `box-design` | skill | Decompose systems into boxes with explicit interface, error, and test contracts. Auto-triggers when designing or planning. `/box-design verify` audits a plan or codebase; `/box-design levels` pitches a design at the right altitude. The bare name works unless another command is also named `box-design`; `/eng:box-design` always works. |
| `retry-contracts` | skill | Errors carry retry advice (`shouldRetry`, `retryAfter`) through one shared interface; exactly one level retries per flow, and never against an upstream time budget. Auto-triggers on retry, backoff, and transient-error work and alongside `box-design`. `/retry-contracts verify` audits a plan or codebase. |
| `clean-code` | skill | Everyday conventions: container-relative names (`users.Get`), `get` returns an optional and `find` a collection, arguments ordered by variance and matched across calls, booleans as questions, distinct types where mix-ups are costly (optional), enums over flag arguments, early returns, comments that say why, and never fighting the project's formatter. Auto-triggers when writing or reviewing code. `/clean-code verify` audits a plan or codebase. |
| `design-auditor` | agent | Read-only audit of a plan, diff, or repo against the `box-design` and `retry-contracts` checklists, in an isolated context. Returns findings only. |

## Install

1. **Install the plugin** from your shell:

   ```sh
   claude plugin marketplace add deankarn/claude-harness
   claude plugin install eng@big-damn-heroes
   ```

   Inside Claude Code, the same commands work as `/plugin marketplace add …` and `/plugin install …`.

2. **Turn on auto-update**, so every push to `main` reaches this machine. In Claude Code: `/plugin` → **Marketplaces** → `big-damn-heroes` → **Enable auto-update**. This can't be set by the install commands.

3. **Optional — skip the `design-auditor` permission prompts** by adding this to `~/.claude/settings.json`:

   ```json
   { "permissions": { "allow": ["Skill(eng:box-design)", "Skill(eng:retry-contracts)"] } }
   ```

4. **Restart Claude Code**, or run `/reload-plugins`.

Check it worked: `claude plugin list` shows `eng@big-damn-heroes` as enabled.

Had `box-design` installed as a standalone skill before? Delete `~/.claude/skills/box-design`, or it loads twice.

### Alternative: settings file

For dotfiles, so a new machine needs no commands. Merge this into `~/.claude/settings.json` and start Claude Code. It installs the plugin, with auto-update on:

```json
{
  "extraKnownMarketplaces": {
    "big-damn-heroes": {
      "source": { "source": "github", "repo": "deankarn/claude-harness" },
      "autoUpdate": true
    }
  },
  "enabledPlugins": { "eng@big-damn-heroes": true },
  "permissions": {
    "allow": ["Skill(eng:box-design)", "Skill(eng:retry-contracts)"]
  }
}
```

## Update

With auto-update on, there's nothing to do. Claude Code picks up new commits within a few minutes of starting a session and asks you to run `/reload-plugins`.

To update right away:

```sh
claude plugin marketplace update big-damn-heroes
claude plugin update eng@big-damn-heroes
```

## Disable in one repo

Add this to that repo's `.claude/settings.local.json`:

```json
{ "enabledPlugins": { "eng@big-damn-heroes": false } }
```

## Develop

```sh
claude --plugin-dir ./plugins/eng   # shadows the installed copy for this session
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
