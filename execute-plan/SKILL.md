---
name: execute-plan
description: Hands an approved plan to a fresh subagent (opus, medium effort, auto mode) that implements it, so the work runs with nothing in its context but the plan. Use when the user asks to execute, run, or implement a plan that already exists, such as the plan file from plan mode. Not for writing the plan.
---

`execute-plan` in this document means you, the agent running this skill. You don't implement anything. You find the plan, hand it to the `plan-executor` subagent, and relay what comes back.

## Find the plan

Take the first of these that exists:

1. A path or plan the user passed as the argument.
2. The plan file this conversation wrote or approved in plan mode, under `~/.claude/plans/`.
3. The most recently modified file in `~/.claude/plans/`. Say which file you picked, since it's a guess.

Read the whole file. If it's empty or isn't an implementation plan, stop and say so.

## Spawn the executor

Call `Agent` with:

- `subagent_type`: `plan-executor`
- `model`: `opus`
- `effort`: `medium`
- `prompt`: the plan file's contents, verbatim, and nothing else.

Don't add a summary, the conversation's history, or your own notes on the plan. The point of the subagent is a context holding only the plan, and anything you add costs it tokens and colours how it reads the plan. If the plan refers to the conversation for something it needs, that's a gap in the plan: tell the user and let them fix the plan before you spawn.

The `plan-executor` definition sets `permissionMode: auto`. That only applies when this session runs in `default` or `dontAsk` mode. In auto, `acceptEdits` or `bypassPermissions` the subagent inherits this session's mode, and in plan mode it can't edit at all, so leave plan mode before you spawn.

## Report

When the subagent returns, show the user its report: what changed, the verification results, and any deviations from the plan. Don't re-run its checks and don't continue its work unless the user asks.
