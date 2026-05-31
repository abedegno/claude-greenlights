# claude-greenlights

A pre-approved Bash command allowlist for [Claude Code](https://docs.claude.com/en/docs/claude-code) — drop it into a project's `.claude/settings.local.json` and Claude stops pausing to ask permission for routine commands.

The name is the metaphor: every entry is a *green light* — a command you've already decided is safe enough to run without a per-invocation prompt.

## Why this exists

Claude Code has a sandbox feature that, when on, auto-allows Bash calls *that successfully execute inside the sandbox*. The config most people start with looks like this:

```json
{
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true
  }
}
```

Reasonable. But you'll still get prompts, and the reason is subtle:

- **The sandbox blocks network and certain filesystem writes.** Many useful commands need those (anything hitting `github.com`, package installs, `git push`, `curl`). When Claude tries to run them under the sandbox, the sandbox refuses, Claude retries with `dangerouslyDisableSandbox: true`, and *that* path requires an explicit permission rule. `autoAllowBashIfSandboxed` does not cover it.
- **First-party tools other than Bash** (Read, Edit, Write, MCP tool calls) aren't sandbox-gated at all and follow normal permission rules.

So in practice the auto-allow shortcut covers a narrower slice than it sounds like. To stop being prompted on `git push`, `pip install`, `curl`, `gh api`, and friends, you need actual `permissions.allow` entries.

## What's in the template

`settings.local.json` allowlists the commands a typical day's coding session leans on:

- **git** — read-only inspection (`status`, `diff`, `log`, `show`, `rev-parse`, `branch`) plus the mutating commands you've usually decided to trust (`add`, `commit`, `push`, `checkout`, `stash`, `restore`)
- **python** — `python`, `python3`, `python3.12`, and common venv paths (`.venv/bin/python`, `./.venv/bin/python`)
- **pip** — install via every common spelling (`pip`, `pip3`, `python -m pip`, `python3 -m pip`, `python3.12 -m pip`)
- **pytest** — `pytest`, `python -m pytest`, `python3 -m pytest`
- **file/text inspection** — `ls`, `cat`, `head`, `tail`, `awk`, `grep`, `rg`, `find`, `jq`, `wc`, `sort`, `uniq`
- **network/GitHub** — `curl`, plus read-only `gh api`, `gh run list/view`, `gh pr list/view`

It does **not** allowlist:

- `rm`, `mv`, or anything else that destroys data
- `git push --force`, `git reset --hard`, `git branch -D`, `git rebase -i` — irreversible or interactive
- `gh pr merge`, `gh pr close`, `gh issue close` — actions visible to others
- `npm install`, `brew install`, `apt`, etc. — supply-chain side effects (add per-project if you want them)
- Anything under `sudo`

The principle: greenlight commands whose worst-case outcome is "wasted time" or "noisy git history," not "lost work" or "someone else gets a notification."

## Install

### Per-project (recommended)

Most teams want the allowlist scoped to one repo, since `git push` to one project isn't an endorsement to push to another.

```sh
mkdir -p /path/to/your/project/.claude
cp settings.local.json /path/to/your/project/.claude/settings.local.json
```

If your project already has a `.claude/settings.local.json`, merge the `permissions.allow` arrays by hand — don't overwrite. (Claude Code merges settings from multiple sources, but only at the top level; the `allow` array itself isn't deep-merged.)

Make sure `.claude/settings.local.json` is in your `.gitignore` if you don't want to commit it — it's local-by-convention and may differ per teammate.

### Global

If you want the same allowlist everywhere, put it in `~/.claude/settings.json` instead. This applies to every project on your machine, so be more conservative about what you greenlight — a `git push` rule there fires on every repo you ever open.

## Reload

Edits to `.claude/settings.*.json` don't always hot-reload. After changing the file:

- Run `/config` inside Claude Code to reopen the settings UI (this reloads the file), or
- Restart the session

## Customizing

Add patterns one at a time as you notice the same prompt repeating. The syntax:

| Form | Matches |
|---|---|
| `"Bash(npm test)"` | exact command |
| `"Bash(npm test:*)"` | `npm test` and any suffix (`npm test --watch`, etc.) |
| `"Bash(git *)"` | any git subcommand (broader than per-subcommand; convenient but stops you from selectively denying `git push --force`) |
| `"Read"` | every Read tool call |
| `"mcp__<server>__<tool>"` | a specific MCP tool |

The colon-wildcard form (`:*`) is usually what you want — pins the verb, allows flags.

To find which patterns to add, the [`/fewer-permission-prompts`](https://docs.claude.com/en/docs/claude-code/skills) skill (if installed) scans your transcripts and suggests them.

## License

MIT — see `LICENSE`.
