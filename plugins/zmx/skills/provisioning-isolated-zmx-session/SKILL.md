---
name: provisioning-isolated-zmx-session
description: Use when an orchestrator session is about to dispatch one or more subagents that need an isolated, sandboxed environment — typically because the subagent will run untrusted commands, install dependencies, modify build state, or work on a feature where filesystem/process isolation from the parent is required. Triggers include explicit mention of "sandbox", "isolated subagent", a request to run work in a container, or invocation of this skill with a branch/feature slug argument.
---

# Provisioning an Isolated zmx Session

## Audience

**Orchestrator sessions only.** This skill prepares an isolated environment that subagents will then use via `zmx:using-zmx`. Subagents themselves never invoke this skill — they receive a session name from the orchestrator and route commands through it.

## Overview

An "isolated zmx session" is a zmx session whose foreground process is a container shell (podman/docker/devcontainer) running against a dedicated git worktree. Once provisioned, every `zmx run <session> <cmd>` lands inside that container — which is exactly the contract `zmx:using-zmx` already documents. Subagents need no special handling; they use `zmx:using-zmx` unchanged.

**Core principle:** orchestrator does setup *once per plan*; every subagent in that plan (implementer, spec reviewer, code reviewer, next task's implementer) shares the same session.

**Announce at start:** "I'm using the provisioning-isolated-zmx-session skill to set up a sandboxed environment for the subagent(s)."

## First action: load current zmx syntax

Before issuing any zmx command, run `zmx help` once. zmx flags evolve between versions — current help is the source of truth.

## Lifecycle: one session per plan, not per dispatch

Multiple subagents in a single plan share one session. Otherwise the spec reviewer can't see what the implementer wrote, the code reviewer can't see the commits, and the next task's implementer can't continue from where the previous left off. Cleanup is **orchestrator-triggered**, never automatic — see Cleanup section.

## Slug resolution

Check whether a branch/feature slug was passed as an argument (e.g., `add-cache-headers`).

- **Argument provided:** branch name = `<slug>`, session name = `claude-<slug>`.
- **No argument:** ask the user. Do not invent a slug. Same fallback policy as `using-git-worktrees`.

Announce the resolved session name to the user before issuing any commands so they can `zmx attach <name>` from another terminal to watch.

## Setup sequence

### 1. Create the worktree

**REQUIRED SUB-SKILL:** Use `superpowers:using-git-worktrees` to create the worktree. Do not duplicate its directory-selection or .gitignore safety logic here.

The worktree is required even when a container is also being used. The container isolates *processes and filesystem state inside the sandbox*; the worktree isolates *the git branch and on-disk source tree the container mounts*. Skipping the worktree means the container mounts the live working copy of whatever branch the host happens to be on — which corrupts the orchestrator's working directory and cross-contaminates with concurrent work.

After it returns, you have:
- `<worktree-path>` — absolute path to the worktree
- `<branch>` — the branch name (= the slug)

### 2. Resolve the container start command

Try in this order:

1. **`.devcontainer/devcontainer.json` exists** → use the `devcontainer` CLI. Verify the exact invocation by running `devcontainer --help` (the API has changed across versions); typical form is:
   ```
   devcontainer up --workspace-folder <worktree-path>
   devcontainer exec --workspace-folder <worktree-path> bash
   ```
   The second command (`exec ... bash`) is the foreground process you'll boot inside zmx.

2. **CLAUDE.md describes a sandbox container start command** → use it. Check with:
   ```
   grep -i -A 10 "sandbox\|container\|devcontainer" CLAUDE.md
   ```

3. **Neither** → ask the user:
   ```
   No .devcontainer/ found and CLAUDE.md does not document a sandbox container.
   What command should I use to start the sandbox container with the worktree
   (<worktree-path>) mounted as the working directory?

   Example: podman run -it --rm -v <worktree-path>:/workspace -w /workspace <image> bash

   Would you like me to save this to CLAUDE.md so future sessions don't need to ask?
   ```
   If they say yes, append a clearly labeled section to CLAUDE.md **after the container is verified working** (i.e., after the sanity checks in step 4 pass). Don't persist a command that hasn't been proven to work — you'd lock future sessions into the broken form.

**Do not guess an image name. Do not invent a default.** If the user has not specified one, ask.

### 3. Boot the session with the container as its foreground process

Choose `<session>` = `claude-<slug>`.

```
zmx run <session> -d <container-start-command>
```

`zmx run` auto-creates the session if it doesn't exist (verified empirically — no separate `zmx attach -d` is needed). The `-d` flag detaches so the call returns immediately while the container keeps running as the session's foreground process. **`-d` must come after `<session>`, not before** — `zmx run -d <session> ...` is parsed as session named `-d`.

Example with podman:
```
zmx run claude-add-cache-headers -d podman run -it --rm \
    -v /home/sam/repo/.worktrees/add-cache-headers:/workspace \
    -w /workspace ubuntu:24.04 bash
```

### 4. Sanity-check the session is talking to the container, not the host

This step is non-negotiable. If the container failed to start, or pulled the wrong image, or the mount didn't take, every subsequent `zmx run` falls through to the host's login shell silently — and a subagent dispatched into that session will modify the host filesystem while believing it's sandboxed.

Run all three:

```
zmx run <session> cat /etc/os-release          # expect container OS, not host
zmx run <session> command -v base64 printf     # zmx write requires both
zmx run <session> ls /workspace                # expect mounted worktree contents
```

**Inspect each output before proceeding.** If `/etc/os-release` shows the host OS, the container did not take over the foreground — diagnose with `zmx history <session> | tail -100` before going further. Do not dispatch any subagent into an unverified session.

### 5. Detect the container shell

If `command -v base64 printf` shows the prompt is `$` not the host's prompt, you're likely in `bash` or `sh`. If the container default is fish, every subsequent `zmx run` needs `--fish`. Check:

```
zmx run <session> echo $0
```

Tell the dispatched subagent which shell they're in if it's not bash.

### 6. Dispatch subagents into the session

When dispatching any subagent (implementer, spec reviewer, code quality reviewer) for this plan, **prepend** this to the subagent's prompt:

> First, invoke `zmx:using-zmx` with session-name argument `<session>`. Then proceed with: <task>.

The subagent then routes everything through `zmx run <session>` per `zmx:using-zmx`'s contract — which means everything lands in the container.

Same `<session>` for every subagent in the plan. Do not provision a new one per dispatch.

## Cleanup (orchestrator-triggered, never automatic)

After the plan is complete and you've moved on to integration:

1. (Optional) `zmx run <session> exit` — clean container shutdown.
2. `zmx kill <session>` — container exits with the session.
3. **Ask the user before removing the worktree.** They may want to inspect, merge, push, or keep it. Coordinate with `superpowers:finishing-a-development-branch` rather than reinventing.

Never auto-cleanup mid-plan. Subagents may need to be re-dispatched (review found issues), and re-creating the environment loses container state (installed deps, build caches).

## Quick reference

| Step | Command |
|---|---|
| Worktree | Delegate to `superpowers:using-git-worktrees` |
| Boot container in session | `zmx run <session> -d <container-start-command>` |
| Sanity: OS | `zmx run <session> cat /etc/os-release` |
| Sanity: zmx write deps | `zmx run <session> command -v base64 printf` |
| Sanity: mount | `zmx run <session> ls /workspace` |
| Detect shell | `zmx run <session> echo $0` |
| Dispatch subagent | Prepend "invoke `zmx:using-zmx` with session-name `<session>`" |
| Tear down | `zmx kill <session>`, then ask about worktree |

## Pitfalls

| Symptom | Cause / fix |
|---|---|
| Sanity check `/etc/os-release` shows host OS | Container failed to take over the foreground. `zmx history <session> \| tail -100`, fix root cause, kill session, retry. Do not dispatch subagent. |
| `zmx write` fails inside session | Container missing `base64` or `printf`. Caught by sanity check 2. Use a different image or install them. |
| Subagent's commands fail with cryptic shell errors | Container shell is fish or sh, not bash. Re-detect with `echo $0`; tell subagent which shell, or pass `--fish` in their `zmx run` calls. |
| Worktree-side files don't show up in container | Mount path mismatch between `-v <worktree-path>:/workspace` and what the subagent thinks is `/workspace`. Re-confirm with sanity check 3. |
| Container crashes mid-task; subsequent `zmx run` outputs look like host shell | Foreground container exited; session fell back to host login shell. Re-run sanity check after any error. Kill and re-provision; do not let subagent continue. |
| `zmx run -d <session> ...` creates session named "-d" | Flag order. Use `zmx run <session> -d ...`. |
| Two plans share one session | Cross-contamination of branches and deps. One session per plan; do not reuse. |

## Red flags — STOP

| Thought | Reality |
|---|---|
| "Podman started fine, I'll skip the sanity check" | Race conditions and image misconfig produce silent host-shell fall-through. The subagent will corrupt the host filesystem while believing it's sandboxed. Always run all three sanity checks. |
| "Let the subagent set up its own container" | Orchestrator does setup. Subagents use `zmx:using-zmx` against an already-provisioned session. Pushing setup down means each subagent re-provisions, loses state, and mis-shares with siblings. |
| "Containers are slow, let me reuse one across plans" | One session per plan. Different branches, different deps, different histories — sharing crosses isolation boundaries silently. |
| "The container is enough isolation, I'll skip the worktree" | Container isolates the sandbox; worktree isolates the source tree the container mounts. Skip it and the container mounts whatever branch the host is currently on, corrupting the orchestrator's working copy. |
| "I'll guess a sensible default image" | If the user hasn't specified, ask. A wrong base image silently misses required tooling and produces test failures the subagent will chase down rabbit holes. |
| "I'll dispatch the subagent now and verify the session in parallel" | Sequence matters. Verify first, dispatch second. Once the subagent starts, host-shell fall-through becomes destructive. |
| "I'll auto-clean up after the last subagent finishes" | Cleanup is orchestrator-triggered after the user is satisfied. Mid-plan auto-cleanup loses container state needed for review subagents and follow-up tasks. |

## Integration

**Required sub-skill:** `superpowers:using-git-worktrees` (worktree creation)

**Pairs with:**
- `zmx:using-zmx` — what every dispatched subagent invokes against the session this skill provisions
- `superpowers:subagent-driven-development` — the orchestrator pattern this skill plugs into
- `superpowers:finishing-a-development-branch` — coordinate worktree cleanup decisions
