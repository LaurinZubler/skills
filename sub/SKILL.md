---
name: sub
description: Hands a task to a fresh subagent so the work happens outside this conversation's context, then shows its report. Optional leading arguments pick the model (default opus), effort (default medium), and permission mode (default auto, or plan for a read-only run that returns a plan), e.g. `/sub sonnet high fix the flaky login test`. Use when the user asks to run something in a subagent, to send or delegate a task to a sub, or invokes /sub.
---

`sub` in this document means you, the agent running this skill. You don't do the task. You turn the request into a brief, hand it to a subagent, and relay what comes back. The point is that the work, and every file it reads, fills the subagent's context instead of this one.

## Read the arguments

Leading words that match one of these lists are settings, in any order. Everything after them is the task.

- **Model:** `opus` (default), `sonnet`, `haiku`, `fable`.
- **Effort:** `low`, `medium` (default), `high`, `xhigh`, `max`.
- **Permission mode:** `auto` (default), `edits`, `ask`, `plan`. `plan` is read-only: the subagent researches and returns a plan or an answer, and changes nothing.

`/sub sonnet high ask fix the flaky login test` runs on sonnet at high effort, asking for permission. `/sub fix the flaky login test` runs on opus at medium effort, in auto mode. If no task is left after the settings, ask for one.

## Write the brief

The subagent sees nothing of this conversation. Its prompt is all it gets, so the brief has to stand on its own:

- **Goal:** what to do and why, in the user's terms.
- **Context:** only the facts from this conversation the task needs: file paths, decisions already made, constraints the user stated, things already ruled out. Write them as facts, not as a history of the chat.
- **Done:** what finished looks like, and any check that proves it.

When the request already stands alone, such as a plan or a precise instruction, pass it verbatim. When the task lives in a file, give the path and tell the subagent to read it. Don't read the file yourself: that would pull it into this context, which is what the skill avoids. Leave out your own opinions on how to do the task unless the user settled them.

If the request leans on something you can't pin down, like "that bug" with no clear referent, ask the user before you spawn.

## Spawn

Call `Agent` with:

- `subagent_type`: `sub` for `auto`, `sub-edits` for `edits`, `sub-ask` for `ask`, `sub-plan` for `plan`.
- `model`: the chosen model.
- `effort`: the chosen effort.
- `prompt`: the brief.

Run it in the foreground and wait for it.

The agent's permission mode only takes effect when this session runs in `default` or `dontAsk` mode. In auto, `acceptEdits` or `bypassPermissions` the subagent inherits this session's mode, and in plan mode it can't edit at all, so leave plan mode before you spawn anything but `plan`. `sub-plan` has no edit tools whatever mode it ends up in, but it still has Bash, so its read-only rule is an instruction there, not a lock.

## Report

Show the user the subagent's report. For a `plan` run that's the plan itself. Otherwise it's what it did, what changed, the check results, and any choices it made where the brief was unclear. Don't redo its work or re-run its checks unless the user asks.
