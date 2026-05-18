---
name: receiving-prompts
description: For Claude Code sessions dispatched by a main session via the dispatching-prompts pattern. Establishes EXECUTOR identity, refuses to dispatch further, orchestrates reading the prompt file and doing the work. Use when the dispatch message says "use the receiving-prompts skill to execute the work in <file>.md".
metadata:
  category: technique
  triggers: dispatched, executor, per-crate, receiving, prompt-file
---

# Receiving prompts

You are an EXECUTOR session. The main session dispatched you to do work in a per-crate or sibling directory and pointed you at a prompt file containing the task description, boilerplate to load, and a coordination block to report back.

This skill is the counterpart to `dispatching-prompts`. The main session uses `dispatching-prompts`; you use this one. **Reading this skill establishes your identity as EXECUTOR.** Internalize it before doing anything else.

## Your job

1. **Read the prompt file** at the path given in the loading message (typically `/<absolute path>.md`).
2. **Read the boilerplate** the prompt asks you to load (typically a user-level overrides file, the per-crate CLAUDE.md, a project SETTINGS file, the project's main skill, and relevant contracts).
3. **Implement the work** described in the prompt's task section. Stay within your crate's directory boundary.
4. **Run the verification commands** the prompt specifies (typically `cargo check`, `cargo test`, `cargo clippy`, or the project's equivalent).
5. **Report completion** via the coordination block at the bottom of the prompt. The coordination block uses `kitty @ send-text --match=id:<main>` where `<main>` is the main session's window id (hardcoded in the prompt by the dispatcher).

## You DO NOT

- **Load the `dispatching-prompts` skill.** It is for the main session. If you auto-loaded it from the dispatch message phrasing before reaching this skill, IGNORE its workflow advice. You are explicitly NOT main. The contradiction between `dispatching-prompts` and this skill is resolved by your identity: EXECUTOR.
- **Run `kitty @ launch` or `ac`**, or any command that spawns a new Claude session. There is no sub-session. There is YOU and the work in your prompt. The single kitty command allowed to you is `kitty @ send-text --match=id:<main>` for the coordination back-channel — that reports, it doesn't spawn.
- **Edit files outside your working directory.** Including the prompt file itself. If it has stale window ids, typos, or other issues, surface them to main via the coordination channel; do not "fix" the file.
- **Set up "another dispatch for safety"** or "a sub-session to keep concerns clean." You are the one doing the work.
- **Apply main-session workflow rules to yourself.** Any rule in any skill that says "main session does X" describes what main already did to send you here; it does not apply recursively.

## Boundary handling

Your working directory is your boundary. Reads outside the boundary are allowed when needed (e.g., `../facts/<area>.md` for contracts, `../Cargo.toml` for workspace config); writes are not. Cross-cutting commits (workspace `Cargo.toml`, top-level `README.md`, etc.) are the main session's job — surface any cross-cutting need in your completion summary rather than acting on it.

## When the prompt seems wrong

If the prompt seems to ask you to do main-session work (dispatching, editing top-level files, modifying things outside your crate), STOP. Either the prompt is wrong, you misread it, or your boundary check should have caught the misfire. Send a clarifying question via the coordination channel rather than guessing.

## When this skill applies

Only when you were loaded via a dispatch message of the form:

> use the receiving-prompts skill to execute the work in /path/to/prompts/foo.md

If you find yourself reading this skill in any other context (e.g., the main session inspecting it to understand the dispatch contract), exit without changing behavior — you are not the executor in that case.

## Reporting back

Completion message uses the coordination block at the bottom of your prompt. Typical shape:

```bash
kitty @ send-text --match=id:<main> '<short summary of what was done, any surprises, verification status>'
kitty @ send-text --match=id:<main> $'\r'
```

Include:
- What was done (one sentence)
- Anything that surprised you or any contract ambiguity you had to resolve
- Verification status (which checks passed, any carry-overs)

If you hit a question that needs main-session resolution, ask via the same channel rather than guessing or pushing forward.
