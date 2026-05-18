# claude-dbc

Skills for AI coding agent harnesses (Claude Code, Codex) that apply Bertrand Meyer's **Design by Contract** at the architecture level: a shared-contracts directory holds the source of truth, the main session writes contracts and prompts (never component code), and per-component subagent sessions implement against the contracts in their own boundaries.

Three skills:

- **`design-by-contract-for-ai-agents`** — multi-component, multi-agent orchestration. Architecture, roles, workflow, contract format, prompt format, anti-patterns, and a Bootstrap Discovery flow that fires when you start (or join) a project that doesn't have contracts yet.
- **`dispatching-prompts`** — main-session companion: how to send a prompt from the main session to a component session running in another terminal window. Covers the file-path injection pattern and the kitty + `ac` setup the author uses.
- **`receiving-prompts`** — executor-side counterpart to `dispatching-prompts`. Establishes EXECUTOR identity at load time and forbids the dispatched session from re-dispatching (running `kitty @ launch` / `ac` / spawning sub-sessions). Loaded by the main session's send-text phrasing — `use the receiving-prompts skill to execute the work in <file>.md` — so it activates *before* the receiving session can auto-load `dispatching-prompts` and accidentally start a recursive dispatch chain. The `dispatching-prompts` skill's top section now defers to this one for any session whose first user-instruction is a prompt-file pointer.

## Install

Direct from GitHub, no npm publish in the loop:

```
npx github:mwaddip/claude-dbc
```

That runs the installer interactively and asks which harness to target.

Non-interactive forms:

```
npx github:mwaddip/claude-dbc install --harness=claude-code
npx github:mwaddip/claude-dbc install --harness=codex
npx github:mwaddip/claude-dbc install --harness=claude-code --skill=design-by-contract-for-ai-agents
npx github:mwaddip/claude-dbc install --harness=claude-code --project
npx github:mwaddip/claude-dbc list
```

| Flag | Effect |
|------|--------|
| `--harness=<name>` | `claude-code` or `codex` |
| `--skill=<name>` | install one skill instead of all |
| `--project` | install to `./.claude/skills/` or `./.agents/skills/` (project-level) instead of user-level |

After install, restart your harness so it picks up the new skills.

## Updating

Just rerun `npx github:mwaddip/claude-dbc` — `npx` re-clones the repo each invocation. The installer overwrites the skill files in place.

## Optional: `ac` (kitty session launcher)

The `dispatching-prompts` skill assumes you can launch agent sessions in named terminal windows that other sessions can find. The author uses kitty + a small bash script called `ac` (Auto Claude), shipped at the top of this repo. `ac` runs `claude` in the current kitty tab and writes a row to a flat session-registry file — so a parent Claude session can `grep <component> ${XDG_RUNTIME_DIR:-/tmp}/ac/sessions` instead of parsing the kilobytes of JSON `kitty @ ls` returns. Token savings add up across a long session.

### Requirements

- [kitty](https://sw.kovidgoyal.net/kitty/) with remote control enabled (`allow_remote_control yes` in `~/.config/kitty/kitty.conf`)
- [Claude Code](https://claude.com/claude-code) CLI on `PATH` as `claude`
- bash, python3 (one inline `kitty @ ls` JSON parse), GNU coreutils

### Install

If you're going to use `npx` to install the skills anyway, the simplest setup is to clone this repo once and symlink `ac` from somewhere on your `PATH`:

```
git clone https://github.com/mwaddip/claude-dbc.git ~/projects/claude-dbc
ln -s ~/projects/claude-dbc/ac ~/.local/bin/ac     # or anywhere on PATH
```

To update: `cd ~/projects/claude-dbc && git pull`. The symlink follows.

### Usage

In any kitty tab:

```
ac                # start a new Claude Code session in this tab
ac -r             # resume the previous Claude Code session
ac -o "<opts>"    # pass extra flags to claude (must be the last argument)
```

`ac` registers `(window_id, tab_id, pwd)` in the sessions file before launching, and removes its row on Claude Code exit.

### Sessions file

```
${XDG_RUNTIME_DIR:-/tmp}/ac/sessions
```

Per-user tmpfs path. Tab-separated, no headers: `<window_id>\t<tab_id>\t<pwd>`. Cleared on reboot — the kitty windows the entries refer to are also gone after a reboot, so persisting them would only produce broken references.

### Caveats

- Kitty-only. Won't work in tmux, screen, or vanilla terminals (relies on `KITTY_WINDOW_ID`).
- Multi-user safety in the `/tmp` fallback: if `XDG_RUNTIME_DIR` is unset on your system (rare on modern Linux desktops, possible on macOS or minimal containers), two users on the same machine collide on `/tmp/ac/sessions`. Set `XDG_RUNTIME_DIR` per-user, or don't use the fallback.

If you're not on kitty, the `dispatching-prompts` skill still applies as a *concept* (file-path prompt injection between agent sessions in any terminal multiplexer that supports cross-window text injection) but the kitty-specific commands won't work as-is.

## Skill format

Both skills follow the [agentskills.io specification](https://agentskills.io/specification): YAML frontmatter (`name`, `description`, optional `metadata`) followed by a markdown body, in `<harness-skills-dir>/<skill-name>/SKILL.md`. This is the format Claude Code, Codex, and other compliant harnesses load.

Adapters for other harnesses (Cursor's `.cursor/rules/*.mdc`, Aider's `CONVENTIONS.md`-style references) are not implemented; PRs welcome.

## License

MIT.
