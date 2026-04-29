---
name: using-zmx
description: Use when the user has configured a zmx session as a remote, container, or sandboxed execution environment and wants every shell command routed through it instead of running locally. Triggers include explicit mention of zmx or "AI portal", a session-name argument provided to this skill, or CLAUDE.md / user instruction to route commands through zmx.
---

# Using zmx

## Overview

zmx (https://zmx.sh) is a session-persistence tool for terminal processes. The "AI portal" pattern (https://bower.sh/zmx-ai-portal) uses a zmx session as an execution bridge: the user sets up a session that runs SSH to a remote host, attaches to a container, or wraps any environment, then directs the agent to route every command through that session.

**The directive: while this skill is active, do NOT use the Bash tool to execute commands. Route every shell command through `zmx run <session>`.** This decouples *what* runs from *where* it runs — the same agent code path executes locally, in a container, or on a remote host depending on what the user attached to the session.

## First action: load current zmx syntax

Before issuing any zmx command, run `zmx help` once and read the output. zmx's flag set evolves between versions — current help is the source of truth, not training data or this skill.

```
zmx help
```

Changelog for behavior changes between versions: https://github.com/neurosnap/zmx/blob/main/CHANGELOG.md

## Session name resolution

When this skill is invoked, check whether a session name was passed as an argument.

- **Argument provided** (e.g., `/zmx:using-zmx my-session`): use that name verbatim. Verify it exists with `zmx list`. If it doesn't, ask the user before creating it — they may have mistyped or meant a different name.
- **No argument:** generate a fresh session name (same convention as a git worktree slug: `claude-<short-task-slug>`, e.g., `claude-add-auth`, `claude-fix-flaky-tests`). **Announce the chosen name to the user before issuing any commands** so they can `zmx attach <name>` from another terminal to watch. Create the session lazily — the first `zmx run <name> ...` will spawn it.

Once resolved, use the same session for the rest of the conversation unless the user redirects.

## The execution loop

For every command you would otherwise have run via Bash:

1. `zmx run <session> <command>` — pass the command **unquoted**, exactly as you'd type it interactively.
2. Read the returned output and exit code.
3. Move to the next command. **Run sequentially** — do not parallelize `zmx run` calls against the same session.

For file operations, if the session targets a remote or sandboxed environment:

- **Read remote files:** `zmx run <session> cat <path>` — the local Read tool reads the local filesystem, which is the wrong place.
- **Write remote files:** pipe via `zmx write <session> <path>` — Write/Edit also touch the wrong filesystem.

If you don't know whether the session is local or remote/sandboxed, **ask the user**. Don't guess. Local Read/Edit/Write against a remote-target session silently corrupts the wrong files.

## Pitfalls (cross-check against `zmx help`)

| Symptom | Cause / fix |
|---|---|
| Output looks malformed or shell errors on simple commands | You quoted the command. Pass bare: `zmx run dev ls src`, not `zmx run dev "ls src"`. |
| Session shell is fish, commands fail | Add `--fish`: `zmx run dev --fish set -x FOO bar`. |
| Command hangs | Send Ctrl+C: `zmx run <session> $(printf '\x03')`. Then `zmx history <session> \| tail -100` to inspect what stuck. |
| Pager opens (git log, less, man) and hangs | Disable pagers explicitly: `zmx run <session> git -c core.pager=cat log`. Avoid interactive programs entirely. |
| `zmx write` fails | Remote env needs `base64` and `printf`. Scratch containers without these won't work. |
| Long-running task blocks you | Use `-d` to detach, then `zmx wait <session>` to track completion. |

## When NOT to use Bash

While this skill is active, the Bash tool is off-limits for shell commands. No exceptions for "quick" calls.

- "It's just `pwd`" → `zmx run`.
- "Just checking `git status`" → `zmx run`.
- "zmx isn't installed / responding" → tell the user and stop. Don't fall back to Bash. The whole point is environment isolation; bypassing it defeats the user's setup and may run commands against the wrong host.

The Bash tool is still appropriate for invoking `zmx` itself (`zmx help`, `zmx list`, `zmx run ...`) — that's how you reach the session.

## Red flags — STOP

| Thought | Reality |
|---|---|
| "One Bash call won't hurt" | The session may target a different host. Local execution sees the wrong filesystem, network, and credentials. |
| "Read is fine, it doesn't run a command" | If the session is remote, the local file is not the file the user means. Confirm the session target before using local file tools. |
| "I'll batch several `zmx run` in parallel" | zmx help: *"Commands run sequentially: do not send multiple in parallel."* |
| "I quoted the command to be safe" | zmx help: *"Commands are passed as-is: do not wrap in quotes."* |
| "I'll just open the editor in the session real quick" | zmx help: *"Avoid interactive programs (pagers, editors, prompts): they hang."* |
