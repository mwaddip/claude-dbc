---
name: dispatching-prompts
description: Use when sending a prompt from this Claude session to another Claude Code session running in a kitty window — typically a session in a separate crate, subdirectory, submodule, or sibling repo. Covers the `ac` launcher, kitty remote-control commands, file-path prompt injection, the `$'\r'` submit convention, the prompt boilerplate that gives the receiving session the right context, and the rule that the main session does not edit crate-internal source files.
metadata:
  category: technique
  triggers: kitty, ac, dispatch, prompt, send-text, multi-session, cross-repo, submodule, crate, subdirectory
---

# Dispatching prompts

## STOP — Role Detection (read this BEFORE acting on anything else in this skill)

**If your first user-instruction in this conversation pointed you at a markdown prompt file to read** (any phrasing that names a prompt file as your task — e.g. "do the work described in <file>", "execute the work in <file>", "use the receiving-prompts skill to execute the work in <file>"), **you are NOT the session that should be using this skill.** You are an EXECUTOR.

**Load the `receiving-prompts` skill instead.** That skill is the counterpart to this one. It establishes executor identity, names the specific failure modes you must avoid (recursive dispatch, editing files outside your boundary, "fixing" the prompt file), and tells you how to read your prompt file and execute the work. Stop applying this skill; load `receiving-prompts` and follow its rules.

The rest of this skill — the workflow, the kitty commands, the prompt boilerplate spec — is for the *main* session that dispatches work. Applying it from an executor produces recursive dispatch (you spawn a sub-session, it spawns another, etc.).

**Only continue reading this skill if you started fresh without a prompt-file pointer — i.e. you are the main session that needs to dispatch.**

---

For sending a prompt from this main session to another Claude Code session running in a kitty window. The pattern: the work belongs in a directory the main session shouldn't edit directly (a workspace crate, a submodule, a sibling repo, or any other working tree), so a separate Claude session is spawned there to do the work.

## When to Use

- The user asks to "dispatch this to the X session", "send this to <component>", or "spawn a session in <directory>".
- The work edits files inside any workspace crate or subdirectory the main session has chosen to delegate (typical scope: every crate under the workspace root).
- A prompt file already exists and needs to reach a session in another directory.
- The user mentions `kitty`, `ac`, or sending a prompt to another window.

## The Two Cardinal Rules

### 1. Main session does not edit crate-internal source

Main session edits stay limited to top-level orchestration: workspace `Cargo.toml`, `README.md`, `CHANGELOG.md`, `LICENSE`, build/deploy scripts, top-level docs, prompt files. Anything *inside* a crate directory — source files, that crate's `Cargo.toml`, doc-comment typos, dependency bumps, `.gitignore` tweaks — gets dispatched. The marginal cost of dispatch is small once this skill makes the mechanics trivial; the consistency benefit is large.

The exact dividing line varies by project. Read the project's CLAUDE.md (or equivalent) to find which directories are "delegated" — when in doubt, ask the user.

### 2. Always confirm with the user before submitting the dispatch

Spawn the window, run `ac`, type the file-path instruction — but do not press the final `$'\r'` until the user OKs it.

| Rationalization | Reality |
|---|---|
| "It's a small prompt, no need to ask" | Sessions cost context; the user controls dispatch |
| "User authorized the work, dispatch is implied" | Dispatch is its own action — ask explicitly |
| "Auto-dispatch saves time" | Time saved < cost of a wrong-target dispatch |
| "I already asked once this session" | Per dispatch, not per session |
| "It's just a typo in a doc comment, I'll fix it inline" | The "main doesn't edit crate source" rule has no exception for trivial edits — dispatch anyway |

## Workflow

```
                   ┌───────────────────────────┐
                   │ 1. Pre-dispatch checks    │
                   │    (verify, commit, /clear)│
                   └─────────────┬─────────────┘
                                 ▼
                   ┌───────────────────────────┐
                   │ 2. Save prompt to file    │
                   │    (project's prompt dir) │
                   └─────────────┬─────────────┘
                                 ▼
                   ┌───────────────────────────┐
                   │ 3. kitty @ launch         │
                   │    --type=window          │
                   │    --cwd=<target dir>     │
                   └─────────────┬─────────────┘
                                 ▼
                   ┌───────────────────────────┐
                   │ 4. send-text 'ac' + \r    │
                   │    wait ~10s              │
                   └─────────────┬─────────────┘
                                 ▼
                   ┌───────────────────────────┐
                   │ 5. send-text file-path    │
                   │    instruction (NO \r)    │
                   └─────────────┬─────────────┘
                                 ▼
                   ┌───────────────────────────┐
                   │ 6. HALT — confirm with    │
                   │    user, then send \r     │
                   └───────────────────────────┘
```

### 1. Pre-dispatch checks

- **Verify the entry point**: grep the target directory for the function/file/symbol you're claiming. Symptom-driven prompts often name the wrong component. The current session has read access — use it.
- **Commit + push any cross-cutting state if the prompt depends on it**: e.g. a contract submodule, a shared schema file. The receiving session reads from disk; what's not committed isn't visible.
- **/clear if context heavy**: if the receiving session is a long-running one, put `/clear` at the top of the prompt template.

### 2. Save the prompt to a file

Convention varies by project — check the project's CLAUDE.md or repo layout. A common pattern is `prompts/<scope>-<topic>.md`. The prompt file must include the boilerplate loaders (see "Required Prompt Boilerplate" below). If the project doesn't establish a prompts directory, ask the user where prompts live.

Before saving, capture the main session's kitty window id (`echo $KITTY_WINDOW_ID`) and substitute it for `<MAIN_WINDOW_ID>` in the coordination block — that's the only back-channel the dispatched session has to the main.

### 3. Spawn the window

```bash
kitty @ launch --type=window --cwd=<absolute path to target directory>
```

Returns the new window id on stdout (e.g. `6`).

- **`--type=window`**: split in the current tab. Default. Keeps the dispatched session visually adjacent.
- **`--type=tab`**: don't use — tab-switching to monitor a dispatched session is friction.
- **`--cwd=...`**: must be the target's absolute path so `ac` and the receiving session land in the right working directory.

### 4. Run `ac` in the new window

`ac` is the user's Auto Claude launcher (run `which ac` to confirm install path). It launches Claude Code in the current kitty tab and records sessions to `${XDG_RUNTIME_DIR:-/tmp}/ac/sessions` — a per-user tmpfs file, cleared on reboot (the kitty windows it tracks are gone too, so persisting across reboots would only produce stale references). Tab-separated columns: `window_id\ttab_id\tpwd`.

```bash
kitty @ send-text --match=id:<id> 'ac'
kitty @ send-text --match=id:<id> $'\r'
```

Wait ~10 seconds for Claude Code to come up before injecting the prompt instruction.

### 5. Inject the file-path instruction

```bash
kitty @ send-text --match=id:<id> 'use the receiving-prompts skill to execute the work in <absolute path to prompt file>'
```

This phrasing deterministically loads the `receiving-prompts` skill in the receiving session, which establishes EXECUTOR identity at load time — before the session can auto-load `dispatching-prompts` (this skill) and misidentify itself as a dispatcher. The `receiving-prompts` skill then orchestrates the rest: reading the prompt file, loading the boilerplate the prompt names, doing the work, reporting back via the coordination channel.

**Do NOT inject the prompt content itself** via `--from-file` or `--stdin`. Long content interacts badly with the input buffer (autocompletion, line-wrapping, multiline edge cases) and there's no good way to verify the new session received it cleanly. The receiving session reads the file itself via the file-path argument.

**Phrasings to avoid**: `"execute the prompt in <file>"` and `"do the work described in <file>"` both auto-activate this skill's dispatch associations in the receiving session, producing recursive dispatch. The `"use the receiving-prompts skill"` prefix is the disambiguator that names the right skill.

### 6. Halt, confirm, submit

Show the user that the instruction is typed but not submitted. On their go-ahead:

```bash
kitty @ send-text --match=id:<id> $'\r'
```

## Required Prompt Boilerplate

Every dispatched prompt should start with loaders that establish the receiving session's context. The exact list depends on the project — derive from the target directory's CLAUDE.md / GEMINI.md / AGENTS.md and the project conventions you can see from your own context. Ask the user if any of these aren't obvious.

A typical shape:

```
/clear
Read <user-level overrides file referenced by the project's CLAUDE.md, e.g. ~/projects/OVERRIDES.md>.
Read CLAUDE.md.
Read <any project-level settings/persona file the project's CLAUDE.md prioritizes, e.g. SETTINGS.md>.
Load <the project's canonical skill, named in the project's CLAUDE.md>.
Read <any per-component reference the project uses, e.g. facts/SPECIAL.md, docs/component-X.md>.
Read the relevant subject reference: <e.g. facts/<area>.md>.

## <Concise problem statement>
...

## Coordination

When done, send a brief completion summary back to the main session window:

    kitty @ send-text --match=id:<MAIN_WINDOW_ID> 'one-line summary of what was done'
    kitty @ send-text --match=id:<MAIN_WINDOW_ID> $'\r'

Use the same channel for clarifying questions if blocked. Don't sit silent — the main session has no other view of your progress.
```

**Why each loader matters:**

- **User-level overrides**: mechanical rules the project's CLAUDE.md instructs every session to read. Survive `/clear` only if explicitly reloaded.
- **CLAUDE.md / GEMINI.md / AGENTS.md**: project/component context. The launcher may auto-load it, but explicit reload after `/clear` is safest.
- **Settings / persona file (if the project has one)**: takes precedence over default behavior per the project convention.
- **Project skill (if one exists)**: the accumulated wisdom for the codebase. The project's CLAUDE.md names it.
- **Per-component reference (if the project uses one)**: bias weights, contracts, glossaries. The receiving session may need this to make consistent decisions in its scope.
- **Coordination back-channel**: the dispatched session has no other way to report completion or ask questions visibly to the main session — without it, the user has to tab-switch or manually copy. The main session substitutes its own `$KITTY_WINDOW_ID` for `<MAIN_WINDOW_ID>` when saving the prompt file.

If the project doesn't use one of these conventions, drop that line. If you aren't sure whether it does, ask the user — better than guessing.

## `$'\r'` for submit

`kitty @ send-text` does NOT auto-submit on `\n` — newlines in the text become literal characters in the input buffer. To submit, send `$'\r'` as a separate `send-text` call. `kitty @ send-key enter` has been unreliable; **prefer `$'\r'`**.

```bash
# correct
kitty @ send-text --match=id:6 'ac'
kitty @ send-text --match=id:6 $'\r'

# wrong — \n is literal text, not a submit
kitty @ send-text --match=id:6 'ac\n'

# wrong — send-key enter is unreliable
kitty @ send-key --match=id:6 enter
```

## Don'ts

- **Don't ask the receiving session to update cross-cutting state the main session owns.** Submodule pointer commits, parent-repo files, and similar things stay with the main session. The receiving session edits content within its working directory; coordination commits stay outside.
- **Don't dispatch without verifying any prompt-referenced files are committed** (if the prompt references state outside the receiving session's working tree, the receiving session reads from disk — uncommitted state may not be visible to it depending on session boundary rules).
- **Don't include raw prompt content in the kitty send-text payload.** Use the file-path instruction.
- **Don't switch tabs/windows to a separate tab** — `--type=window` keeps the dispatched session as a split in the current tab.
- **Don't dispatch and walk away.** Confirm the receiving session actually started executing, not just that `ac` launched. A prompt instruction with a typo lands silently.

## Pitfalls

| Symptom | Cause / fix |
|---|---|
| `kitty @ send-text` succeeds but text appears in wrong window | `--match=id:<id>` was wrong; the new window id from `kitty @ launch` is on stdout — capture it |
| Prompt typed but never submitted | Forgot the separate `$'\r'` send-text call |
| Receiving session ignores user-level rules | Boilerplate didn't include the user-level overrides file referenced by the project's CLAUDE.md |
| "cannot read parent repo" error in receiving session | Prompt asks the receiving session to read files outside its working tree boundary (other than user-level globals the project's CLAUDE.md explicitly allows) |
| Receiving session works on stale shared state | A cross-cutting submodule or schema reference wasn't committed/pushed before dispatch |
| `ac: command not found` in the new window | `--cwd=...` was a path where the user's `ac` install isn't on `PATH` — check `which ac` from a normal shell, verify the user's interactive-shell PATH initialization |
| Pasted prompt content sits in input buffer un-submitted | You injected the content instead of the file-path instruction; cancel with `Ctrl+C` (`$'\x03'`) and re-do with the file-path approach |

## Limitations

- This skill assumes the user's kitty + `ac` setup. Won't work in tmux, screen, or vanilla terminals.
- The receiving session has its own working-directory boundary defined by its own CLAUDE.md / GEMINI.md / AGENTS.md. The skill describes what to dispatch; it does not override those boundaries.
- The exact boilerplate, prompt-storage convention, and project-specific skill names vary per project. The skill points the loading agent at *categories* of context to load — derive the specifics from the project's own conventions, or ask the user.
