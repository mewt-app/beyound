---
name: feedback-agent-worktree-isolation
description: Spawning hmd:coder for work in this repo can isolate the agent into a worktree of a DIFFERENT repo, silently blocking all writes
metadata:
  type: feedback
---

Do not spawn `hmd:coder` for edits in `/Users/rj/Downloads/code/beyound` without
checking where it landed. Prefer a non-worktree agent, or make the edits directly.

**Why:** on 2026-08-14, three `hmd:coder` agents spawned for beyound work were
isolated into worktrees of `/Users/rj/Downloads/code/rallycheckout` — an unrelated
repo that merely happened to be read in the same session. Reads of beyound
succeeded; every Write/Edit was refused by the isolation guard with "Edit the
worktree copy of this file instead", but no such copy existed. Two agents burned
their full run and reported BLOCKED; one routed around it via a scratchpad + `cp`.
Roughly 20 minutes lost, and the failure is silent until the agent reports.

**How to apply:** if a spawned agent reports the isolation guard, do not respawn
blindly — either apply its planned edits yourself (they arrive as exact anchored
diffs, which replay cleanly) or copy its staged file out of the session scratchpad
at `/private/tmp/claude-501/<project>/<session>/scratchpad/`. Also note that a
background `sleep` returns immediately and does not actually wait; block with an
`until`-loop in a foreground Bash call instead. See [[project-beyound-checkout]].
