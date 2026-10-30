---
name: develop
description: Builds a simple feature end to end, without test-driven development. Interviews you with /understand, has /write-plan draft a plan you approve, has /execute-plan build it, then runs a final bug review. For work that needs tests written first, use /develop-tdd.
disable-model-invocation: true
---

`develop` in this document means you, the top-level agent running this skill. The skills it calls run their work in subagents, and no subagent can reach the user, so every question for the user happens in one of your own turns.

Develop the current idea end to end. It calls out to four other skills: `understand` for step 1, `write-plan` for step 3, `execute-plan` for step 4, and `sub` for the final review in step 5, which runs `/code-review`, a Claude Code built-in.

Run this skill on `opus`. You hold the whole run and judge everything that comes back to you. If you're on something weaker, say so in your first turn and offer the user `/model opus` before you go any further.

Don't do the steps' work yourself: no reading the codebase, no drafting the plan, no writing code. Each step hands its work to a skill so that work stays out of this conversation.

# 1. Understand

Invoke the `understand` skill, passing on whatever the user gave `/develop` along with the fact that develop invoked it, so it knows not to add a stop of its own. It interviews the user and returns a context, a summary under `## Background`, `## What Research found`, `## Settled Decisions`, `## Out of Scope` and `## Done when`. That context is the plan's input below.

It only returns once the user has agreed the context, so don't re-open it. Keep it word for word: everything downstream is judged against it, and a paraphrase is a requirement rewritten.

Don't interview the user yourself, and don't skip the call because the idea reads clearly to you. The questions you'd skip are the ones whose answers the plan needs.

# 2. Set up

## Get on a feature branch

The run folder below is named after the branch, and the final review compares the branch against the point it diverged from. Neither works with the work on the trunk.

If the current branch is the trunk, create a feature branch named from the issue or idea, `<type>/<issue-number>-<short-slug>`, or a slug alone when there's no issue. Say which branch you made. Otherwise say which branch you're already on and move on.

## Pin the base

Resolve the branch this one grew out of now, once, for the final review. Don't assume `main` or `master`. The trunk here might be `develop`, `dev`, a release branch, or anything else, and a wrong base makes the review look at the wrong range or at nothing at all.

List the candidates:

```bash
git for-each-ref --sort=-committerdate --count=30 --format='%(refname:short)' refs/heads refs/remotes/origin
```

Drop the current branch and its own remote ref. For each of the rest, compute `git merge-base HEAD <ref>`, and drop any ref whose merge-base equals `git rev-parse HEAD`: that ref already contains `HEAD`, so it would hand the review an empty diff. Rank what survives by merge-base commit date, newest first (`git log -1 --format=%ct <merge-base>`). The top ref is the base.

If nothing survives, the branch has no commits yet, which is why every ref still contains `HEAD`. The base is then the most recent of the refs you dropped. When the answer isn't clean, because several refs tie or the winner is itself a feature branch, ask. Otherwise state the base you picked and move on.

## Set up the run folder

`<feature>` is the branch name with the type prefix stripped and any remaining slashes flattened to dashes: `feat/42-oauth-refresh` gives `42-oauth-refresh`. A branch with no recognizable shape is slugified whole.

Create `.develop/<feature>/` in the repo root and write the context to `context.md` inside it, verbatim. From here on, the context and the plan travel as absolute paths into this folder, never as pasted text, so nothing rewords them on the way.

Check whether the folder is ignored, with `git check-ignore -q .develop`. If it isn't, say so once in your turn and tell the user the entry to add is `.develop/`. Don't edit `.gitignore` and don't write `.git/info/exclude`: ignoring the folder is the user's call, made once per repo. Say it anyway rather than staying quiet, because the build may commit, and one `git add -A` would put the run folder in the repo.

# 3. Plan

## Draft the plan

Invoke the `write-plan` skill. Its task is the absolute path of `context.md`, introduced as the agreed summary to plan from. Tell it to save the plan to `plan.md` in the run folder.

`write-plan` shows the plan when it's done.

## Get approval

If the plan has an `## Open Questions` section, those get answered before anything else. Put them to the user, add each answer to `## Settled Decisions` in `context.md`, then invoke `write-plan` again exactly as above. Approval must never be what closes an open question, since the build would then implement a spec nobody settled.

Then ask the user with `AskUserQuestion` whether to build this plan, naming its path.

If they approve, continue. If they don't, ask what they want changed, then invoke `write-plan` again as above, adding the path of the rejected `plan.md` and their answer. Repeat until a plan is approved. Stop only if the user rejects without saying what to change, since a second draft on the same input would just produce the same plan.

# 4. Execute

## Build

Invoke the `execute-plan` skill with the absolute path of `plan.md`. Leave plan mode first if the session is in it. Keep only the report it returns: the files changed or commits made, the verification results, and the deviations from the plan.

# 5. Review

## Review the code

Invoke the `sub` skill with `opus medium` and this brief, filling in the base branch you pinned and whether the build committed its work or left it in the working tree. A fresh agent reads the diff cold, while anything that built the change carries its assumptions into the review.

> Review the change on the current branch for bugs. Don't change any code. Base branch: `<base>`. The build <committed its work: list the commits | left its work uncommitted in the working tree>.
>
> Check your scope first. With commits, `git log <base>..HEAD --oneline` has to list them. Uncommitted, `git status --porcelain` has to come back non-empty. Either way, an empty scope means the target is wrong rather than the code being clean, so stop and report that instead of returning an empty table.
>
> Then run `/code-review medium --max-findings all <base>` with commits, or `/code-review medium --max-findings all` with no target when the work is uncommitted. Don't pass `--comment` or `--fix`. Run it yourself, and don't spawn other reviewers.
>
> The review reports through the `ReportFindings` tool, so transcribe what it reports into this table, keeping its order, which is most severe first:
>
> ```markdown
> | #   | Category            | Finding                               |
> | --- | ------------------- | ------------------------------------- |
> | 1   | <its category slug> | <finding, with a file:line reference> |
> ```
>
> Return the table and nothing else.

## Stop

Show the user the build report and the findings table, then stop. develop is done. What to do about the findings is the user's call, and the numbered rows are there so they can point at the ones they want. When the work was left uncommitted, say that plainly and offer to commit it, rather than reporting as though it had landed.

Then resolve the run folder. A run is clean when the findings table is empty and the build reported no deviations and nothing left undone: delete `.develop/<feature>/` and say you did, since git history is the record from there. Otherwise keep the folder and name its absolute path, so the user can find the context and plan behind the findings.
