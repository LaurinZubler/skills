---
name: execute-plan
description: Hands an approved plan to a fresh subagent through /sub (opus, medium effort, auto mode by default) that implements it, so the work runs with nothing in its context but the plan. Use when the user asks to execute, run, or implement a plan that already exists, such as the plan file from plan mode. Not for writing the plan.
---

`execute-plan` in this document means you, the agent running this skill. You don't implement anything. You find the plan, hand it to `/sub` with the brief below, and relay what comes back.

## Read the arguments

Leading words that `/sub` takes as settings (a model, an effort, or a permission mode) pass through to it unchanged. Without them, use `opus medium auto`. Don't pass `plan`: a read-only subagent can't implement anything. Whatever follows the settings is the plan, a path to it, or nothing.

## Find the plan

Take the first of these that exists:

1. A path or plan the user passed as the argument.
2. The plan file this conversation wrote or approved in plan mode, under `~/.claude/plans/`.
3. The most recently modified file in `~/.claude/plans/`. Say which file you picked, since it's a guess.

Read the whole file. If it's empty or isn't an implementation plan, stop and say so. If the plan refers to the conversation for something it needs, that's a gap in the plan: tell the user and let them fix the plan before you hand it on.

## Hand it to /sub

Invoke the `sub` skill with the settings, then this brief. Fill in the plan's absolute path. If the plan was passed inline rather than as a file, replace the first sentence with "Implement the approved implementation plan below." and append the plan verbatim under a `## Plan` heading.

> Implement the approved implementation plan in `<absolute path to the plan file>`. Read the whole file first. Implement it completely, as written.
>
> - The plan is the spec. Do what it says, in the order its dependencies imply, and nothing it doesn't ask for. Its Out of Scope section binds you.
> - Where the plan names tests, write them and watch them fail before you make them pass.
> - Commit only if the plan or the repo's rules say to. Otherwise leave the work in the working tree. Never push.
> - When something blocks a unit, finish the units that don't depend on it, then report the blocker rather than working around the plan.
> - Run the plan's Verification section at the end and report the real results, failures included.
>
> Return, briefly: the files changed (or commits made), the verification results, each deviation from the plan with its reason, and anything left undone.

The brief stands on its own, so `/sub` passes it through as it is. Don't add a summary, the conversation's history, or your own notes on the plan. Anything you add costs the subagent tokens and colours how it reads the plan.

## Report

`/sub` shows the subagent's report: what changed, the verification results, and any deviations from the plan. Don't re-run its checks and don't continue its work unless the user asks.
