---
name: write-plan
description: Has a fresh read-only subagent research a task and write an implementation plan, through /sub in plan mode (opus, medium effort by default), then saves the plan where /execute-plan finds it. Use when the user asks to plan, or write a plan for, a change without filling this conversation with the research. Not for executing the plan.
---

`write-plan` in this document means you, the agent running this skill. You don't research or plan anything yourself. You hand the task to `/sub` in plan mode, save the plan that comes back, and show it to the user.

## Read the arguments

Leading words that `/sub` takes as a model or an effort pass through to it unchanged. Without them, use `opus medium`. The permission mode is always `plan`; if the user named another one, ignore it and say so. Whatever follows the settings is the task: a description, or a path to a file that holds it. If nothing follows, take the task from the conversation, and if there's none there either, ask for one.

## Hand it to /sub

Invoke the `sub` skill with the settings, then `plan`, then the task followed by the instructions below. `/sub` writes the task into a brief and adds what the subagent needs from this conversation, so pass the task in the user's words and don't research it yourself first. When the task is a file, pass its path and don't read it: the subagent reads the exact text, and nothing gets reworded on the way.

> Research the codebase for this task and write an implementation plan for it. Read the files the change touches and the code it should reuse, so the plan names real paths and functions. Don't ask questions: a gap you can't settle goes under `## Open Questions`.
>
> If the task comes with settled decisions, they bind the plan: follow them, don't reopen them. If it comes with an out-of-scope list, carry every item into the plan's `## Out of Scope`. If it says what done looks like, every one of those checks goes into `## Verification` with its expected result.
>
> Return the plan as your final message, in this shape and nothing after it:
>
> ```markdown
> # <short title>
>
> ## Context
>
> <why this change, what prompted it>
>
> ## Open Questions
>
> <gaps you couldn't settle. Omit the section when there are none>
>
> ## Approach
>
> <the recommended approach, a paragraph or two of prose. Not the alternatives you dropped>
>
> ## Out of Scope
>
> <what this change deliberately doesn't do>
>
> ## Work units
>
> | #   | Unit | Files | Reuse | Test seams | Depends on |
> | --- | ---- | ----- | ----- | ---------- | ---------- |
> | 1   | <what this unit does> | <paths> | <existing functions to call, with paths> | <the public boundaries it's tested at, or none> | none |
>
> ## Verification
>
> <how to check the whole change works end to end, with commands and expected results>
> ```

## Save it

The subagent can't write files, so you save the plan, verbatim. If the caller named a path for the plan, save it there, overwriting an earlier draft. Otherwise save it to `~/.claude/plans/<slug>.md`, where `<slug>` is the title in lowercase kebab-case, and add a number if the file already exists. If the subagent returned something that isn't a plan, such as a blocker or a question, don't save it: show it to the user instead.

## Report

Show the user the plan and the path you saved it to. Put the Open Questions first if there are any, since they decide whether the plan is ready. Then say the plan runs with `/execute-plan <path>` once they approve it. Don't execute it yourself.
