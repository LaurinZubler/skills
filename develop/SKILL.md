---
name: develop
description: Builds a feature.
disable-model-invocation: true
---

`develop` in this document means you, the top-level agent running this skill. Everything the skill spawns is a subagent, and no subagent can reach the user, so every question for the user happens in one of your own turns.

Develop the current idea end to end. It calls out to two other skills: `understand` for phase 1, and `/code-review`, a Claude Code built-in, for the final pass. Everything else it needs is defined here, and the definitions too long to inline live once in the Reference section at the bottom, with the steps that need them pointing there.

Subagents can't read this file unless you tell them to. When a brief points at a Reference section, pass the subagent the absolute path of this file, which is `<skill-directory>/SKILL.md`. Claude Code gives you this skill's directory when the skill loads.

Only general-purpose subagents can spawn subagents of their own. `Explore` and `Plan` have no `Agent` tool, which is why every orchestrator below is general-purpose.

Every spawn below uses the same shape: a sentence telling you what to spawn, then the brief in a quote block. Everything inside the quote block goes into the subagent's prompt. Everything outside it is yours.

Every spawn also names its model, because an unnamed model is whatever the session happens to default to. The mapping lives once in **Model Selection** at the bottom, and each spawn points at the row it wants.

That table has a row for you too, and you know what you're running on. If it's below that row, say so in your first turn and offer the user `/model opus` before you go any further. You hold the whole run, you judge every summary that comes back to you, and a run held badly costs the rounds it takes to notice.

**The run folder.** From Implement onward, this skill keeps its working files in `.develop/<feature>/`, and briefs hand over paths into it rather than pasted text. `Set up the run folder` in step 3 creates it. Every path you pass to a subagent is absolute, since wave implementers run in worktrees where a repo-relative `.develop/` resolves to an empty folder in their own checkout.

**Small changes collapse the pipeline.** When the whole change fits one work unit in one or two files, and grilling already surfaced every fact the design needs, skip the Draft and Execute orchestrators: compose the plan yourself and run the rounds yourself, delegating only the implementation and the review. You give up a fresh-context pass over the plan, which is the right trade when the plan is short enough that you hold all of it. Say in your turn that you took this lane and why. The lane still sets up the run folder and still keeps the ledger, since you're then the one holding the whole run and the one who can lose their place. Anything larger uses the full pipeline below.

The lane only ratchets one way. If composing the plan yourself shows the change needs more than one work unit, or that grilling left a fact the design needs, stop and take the full pipeline from Draft. Nothing sends you back down: a change that looked small and grew is exactly the case where a fresh-context pass over the plan pays for itself.

# 1. Understand

Invoke the `understand` skill, passing on whatever the user gave `/develop` along with the fact that develop invoked you, so it knows not to add a stop of its own. It interviews the user, verifies the figures the input asserts, and returns a checkpoint summary under `## Context`, `## What Research found`, `## Settled Decisions` and `## Out of Scope`. That summary is Draft's input below.

It only returns once the user has agreed the summary, so don't re-open it. Keep it in your context verbatim: everything downstream is judged against it, and a paraphrase is a requirement rewritten.

Don't interview the user yourself, and don't skip the call because the idea reads clearly to you. The questions you'd skip are the ones whose answers the plan needs.

# 2. Plan

Call `EnterPlanMode` before spawning the draft agent, unless the session is already in plan mode. Get approval below exits plan mode with the finished plan, and that's the gate the whole rest of the skill hangs on.

## Draft the plan

Skip this spawn when the small-change lane from the preamble applies, and compose the plan yourself instead. Otherwise spawn a general-purpose subagent on the model **Model Selection** names for orchestration. It needs no worktree isolation, since this work is read-only. Give it the Checkpoint summary and the absolute path of this file. Its brief:

> Work read-only: no `Edit`, no `Write`, no state-mutating `Bash` command, no commits, no `gh` writes. This binds you and every agent you spawn.
>
> Name a model on every agent you spawn. **Model Selection** in the Reference section of the skill file whose path you were given holds the mapping: lookups for the `Explore` agents below, design for the `Plan` agents.
>
> **Understand**: spawn 1 to 3 `Explore` agents in parallel against the settled decisions. Default to 1; go higher only when the design touches several areas. Look for existing functions, utilities, and patterns to reuse. Don't propose new code where a suitable implementation already exists.
>
> **Design**: spawn 1 to 3 `Plan` agents in parallel to turn the exploration results into an implementation approach. Default to 1; go higher only for genuinely complex or multi-perspective work.
>
> When more than one `Plan` agent runs, you pick the winner rather than presenting a menu: take the approach that best fits the Settled Decisions, graft in anything better from the others, and note in one line why it won. The plan itself never mentions the alternatives.
>
> **Review**: read the critical files the Understand step found. This is the one place in the skill where an agent reads files rather than delegating the read, because review needs the whole file rather than an answer about it, and your context is disposable. Then check the design against the Settled Decisions and Out of Scope. Don't use `AskUserQuestion`. Nobody can answer at this depth, and that interview already happened during grilling. If review finds a genuine gap the checkpoint never settled, name it under `## Open Questions` instead of guessing, so the user catches it at approval time.
>
> **Compose the plan** in this shape. The sections above the table are what the user reads to decide whether to approve, so keep them prose and short. The table is what the implementation is sliced from, so keep it explicit.
>
> ```markdown
> ## Context
>
> <why this change, what prompted it>
>
> ## Open Questions
>
> <gaps the checkpoint never settled. Omit this section when there are none. It sits this high because it's what decides approval>
>
> ## Approach
>
> <the recommended approach, a paragraph or two of prose. Never the alternatives you considered and dropped>
>
> ## Out of Scope
>
> <carried over from the checkpoint, plus anything this design ruled out. The Spec review axis judges scope creep against this list, so an empty list makes that judgement guesswork>
>
> ## Work units
>
> | #   | Unit                  | Files                     | Reuse                        | Test seams                     | Depends on |
> | --- | --------------------- | ------------------------- | ---------------------------- | ------------------------------ | ---------- |
> | 1   | <what this unit does> | <paths, or one representative path plus "and N similar" for a repeated pattern> | <existing functions and utilities to call rather than rewrite, with paths> | <the public boundaries this unit is tested at> | none |
>
> ## Verification
>
> <how to check the whole change works end-to-end, with commands where possible>
> ```
>
> The `Test seams` column is the whole test plan: the user approves these along with the plan, and implementers test at these seams and nowhere else, so a seam you leave out goes untested. A unit can legitimately have no seam at all, when what it changes is data or configuration rather than behaviour. Write `none` there in that case and put an executable check in `## Verification` instead, stating the expected values, so a wrong result is visible rather than merely plausible. `Depends on` is what lets the orchestrator order the work, so name the unit number a unit waits on, or `none`. Units marked `none` with no overlapping files run together; a unit with a dependency runs once what it waits on has landed.
>
> Return the composed plan as your final message. Nothing else after it.

## Get approval

If the returned plan carries an `## Open Questions` section, those get answered before anything else. Put them to the user, then spawn a fresh Draft subagent with the answers as extra input. Approval must never be what closes an open question, since Execute would then implement a spec nobody settled.

Call `ExitPlanMode` with the plan.

If the user approves, continue to Execute. If they don't, ask what they want changed, then spawn a fresh Draft subagent with the rejected plan and their answer as extra input, and call `ExitPlanMode` again with the plan it returns. Repeat until a plan is approved. Stop only if the user rejects without saying what to change, since a second Draft on the same input would just produce the same plan.

# 3. Implement

## Get on a feature branch

Implementers commit to the current working branch, and the base pinning below needs a divergence to find. Neither works with `HEAD` on the trunk.

If the current branch is the trunk, create a feature branch named from the issue or idea, `<type>/<issue-number>-<short-slug>`, or a slug alone when there's no issue. Say which branch you made. Otherwise say which branch you're already on and move on.

## Pin the base

Both reviews later on compare the branch against the point it diverged from, so resolve that base branch now, once, and pass it down. Don't assume `main` or `master`. The trunk here might be `develop`, `dev`, a release branch, or anything else, and a wrong base makes both reviews look at the wrong range or at nothing at all.

List the candidates:

```bash
git for-each-ref --sort=-committerdate --count=30 --format='%(refname:short)' refs/heads refs/remotes/origin
```

Drop the current branch and its own remote ref. For each of the rest, compute `git merge-base HEAD <ref>`, and drop any ref whose merge-base equals `git rev-parse HEAD`: that ref already contains `HEAD` and hasn't diverged from it, so it would hand both reviews an empty diff. Rank what survives by merge-base commit date, newest first:

```bash
git log -1 --format=%ct <merge-base>
```

The top ref is the branch this one most likely grew out of. The 30-ref cap is deliberate, since a branch nobody has committed to in that long isn't a plausible parent.

If dropping those refs leaves nothing behind, the branch simply has no commits yet, which is why every ref still contains `HEAD`, the trunk included. The base is then the most recent of the refs you just dropped, since that's the point this branch will diverge from once Execute commits. Take it and move on rather than ranking what's left.

You're the only actor in this skill who can ask, so ask when the answer isn't clean: several refs tie, or the winner is itself a feature branch. Otherwise state the base you picked and move on.

## Set up the run folder

Everything below this point hands paths to subagents rather than pasted text, so the folder exists before Execute starts. It comes after base pinning because its first ledger line records the base.

`<feature>` is the branch name with the type prefix stripped and any remaining slashes flattened to dashes: `feat/42-oauth-refresh` gives `42-oauth-refresh`. A branch with no recognizable shape is slugified whole.

Create `.develop/<feature>/` and write the approved plan to `plan.md` inside it, verbatim. That file is what every later brief points at for `## Context`, `## Approach`, `## Out of Scope`, and the work units table.

Then start the ledger, `ledger.md` in the same folder, with one line:

```
run | branch <branch> | base <base> | plan <absolute path to plan.md>
```

Check whether the folder is ignored, with `git check-ignore -q .develop`. If it isn't, say so once in your turn and tell the user the entry to add is `.develop/`. Don't edit `.gitignore` and don't write `.git/info/exclude`: ignoring the folder is the user's call, made once per repo. Say it anyway rather than staying quiet, because implementers commit and one `git add -A` would put the run folder in the repo.

Pass the folder's absolute path to Execute.

## Execute

Skip this spawn when the small-change lane from the preamble applies, and run the rounds yourself against the brief below. Otherwise spawn a general-purpose subagent on the model **Model Selection** names for orchestration. It needs no worktree isolation, since the result has to land on the current working branch. Give it the run folder's absolute path, the base branch you pinned, and the absolute path of this file. The spec reaches it as `plan.md` in that folder, so don't paste the plan. Its brief:

> You orchestrate, you don't implement. Every line of this change is written by an agent you spawn, fixes included, so nothing you spawn ever reviews its own work.
>
> Nobody upstream can be reached while you run. Anything you can't resolve goes in your return, never into a question.
>
> Your spec is `plan.md` in the run folder you were given. Read it once. Everything else you write goes in that same folder, and every path you hand a subagent is absolute.
>
> Name a model on every agent you spawn. **Model Selection** in the Reference section of the skill file whose path you were given holds the mapping.
>
> You work in **rounds**. Round 1 implements the spec. Each later round implements fixes for what the previous round's review found. You get 3 rounds at most.
>
> **0. Read the ledger.** `ledger.md` in the run folder is the record of what has already happened, and it may be a previous life of this same run: your own memory does not survive a compaction, but that file does. Read it before anything else. A unit with a completion line is done, so never dispatch it again, and never re-run a round whose review is already recorded. Pick up from the first thing that isn't finished. On a fresh run the file holds only its `run |` line and you start at step 1.
>
> Append to the ledger as you go, one line per event, oldest first. Write the line for a dispatch *before* you dispatch, since the loss you're guarding against is redoing work; write the line for a result *after* it comes back, since that's when the fact exists. The shape:
>
> ```
> r1 baseline | suite pass | typecheck pass
> r1 scan | 3 pairs 2 units | clean
> r1 ruling <conflict> | <what you decided and why>
> r1 wave | units 1,2,3
> r1 unit 1 complete | commits a1b2c3d..e4f5a6b | report r1-unit-1-report.md
> r1 unit 2 retry | red tests | escalated
> r1 merged | suite pass | typecheck pass
> r1 review | standards 3 (1 hard) | spec 1 (1 hard) | head a1b2c3d | r1-review.md
> r1 fixed <finding> | left <finding> (judgement call, untouched file)
> r2 rereview | 2 addressed | 1 open | breakage 0 | head e4f5a6b | r2-rereview.md
> exit clean at r2
> ```
>
> **1. Preflight.** Check the ground before you build on it.
>
> In round 1, before the scan, run the full test suite and typecheck once and record the result in the ledger. That's your baseline. Start them with `run_in_background: true` and do the scan while they run, collecting before you dispatch anything: the scan reads `plan.md` and nothing else, so it never needs the result. A failure in step 5 that also appears there is inherited, not caused, and reporting an inherited failure as yours costs a round chasing code that was never yours to fix. Later rounds skip this, since the previous round's step 5 already recorded the status, and a baseline line already in the ledger is one you already ran: take it rather than running the suite twice.
>
> Then scan the work units table you're about to split, and write the scan to `r<R>-scan.md` in the run folder. It's a table, not a verdict:
>
> - One row per pair of units that share a file or an interface: the two units, what one produces against what the other consumes, and what you found.
> - One row per unit: whether its own text agrees with itself, meaning its seams against its files and its reuse targets against what it creates.
>
> "The scan is clean" without those rows is not a scan you ran. Skip the pair rows when the table holds one unit, since a single unit has no pairs.
>
> Rule on everything the scan surfaces before you dispatch anything, with `plan.md` as the binding authority, and record each ruling in the ledger. Your rulings reach the implementers through the same slot step 3 already uses for ambiguity you noticed in a unit's rows. Stop the run only when a conflict is load-bearing, meaning no unit can be built until somebody decides: return it unresolved with no code written, which is far cheaper than building on a guess and finding out at review.
>
> **2. Decide the split.** Work from a work units table: the spec's `## Work units` in round 1, the remediation table you composed in step 6 in every round after that. It names the units, their files, their reuse targets, their seams, and their ordering.
>
> Units whose `Depends on` is `none` and whose `Files` don't overlap run together as one wave. Once a wave lands, the units waiting on it become eligible and form the next wave. A table you can't read this way, because the dependencies are ambiguous or the file lists are vague, collapses to a single implementer for the whole round: a wrong split costs more in merge conflicts than it saves in tokens.
>
> Batch before you split. Units in the same wave that are the same shape of small edit, one repeated one-line change or constant or field addition across several files, go to one implementer as one brief listing every file and its change, and get reviewed as one diff. Five implementers each rebuilding context to make the same edit costs more than one making it five times. Keep one implementer per unit for work that needs its own tests or its own judgement. Batching also thins the wave, and a wave batched down to a single implementer needs no worktree.
>
> **3. Spawn the implementers.** One general-purpose subagent per unit in the wave, or per batch, or exactly one for the whole round when the table didn't decompose. You never write the code yourself.
>
> Write each one a brief first, at `r<R>-unit-<N>.md` in the run folder, `<R>` being the round and `<N>` the unit number, or the numbers joined by dashes for a batch. The brief holds that unit's rows from the work units table verbatim, and nothing you have paraphrased. Exact values, meaning paths, names, signatures, numbers and expected results, exist only there.
>
> Then dispatch, giving each implementer:
>
> - The absolute path of its brief, introduced as: read this first, it is your requirements, and the values in it are used verbatim.
> - The absolute path of `plan.md`, for `## Context`, `## Approach` and `## Out of Scope`.
> - The absolute path of the skill file.
> - The absolute path of its report file, `r<R>-unit-<N>-report.md`, named after its brief.
> - One line on where this unit sits in the change as a whole.
> - Interfaces and decisions from units that already landed, which its brief can't know.
> - Your resolution of any ambiguity you noticed in its rows.
>
> Nothing else. A dispatch prompt describes one unit, not the round's history, and anything you paste into it stays in your context for the rest of the run and is re-read on every later turn.
>
> Their instructions:
>
> - Implement with **Test-Driven Development**, defined in the Reference section of the skill file at the path you were given. Read it before you start. Test at the seams your rows list and nowhere else, and write no tests at all when those rows say `none`.
> - Run typechecking and single test files as you go. When a test stays red, fix the code until it passes. Driving the implementation from failing tests is the point of the loop, so nothing here hands red tests back up.
> - You may be running in a fresh worktree, which has none of the repo's installed dependencies and none of its generated code. A test or typecheck command that fails because a module, package, binary, or generated type can't be resolved is the worktree, not your code: run the repo's setup once and re-run. Setup is whatever that repo needs before its tests can run, so it's the install command, plus the codegen or workspace build steps in a repo that has them. Find it rather than guessing at it: `CLAUDE.md`, the README's setup section, a `setup` or `prepare` script in the package manifest, a Makefile target. Never treat a missing dependency as a red test to fix in code, and never count that failure as the one **Watch it fail, for the right reason** asks you to read.
> - Don't run the full test suite and don't run the round's review, whichever one it is. Both happen once per round, above you.
> - Nobody upstream can be reached while you run. Report, don't ask.
> - Committing is the repo's call, so read its git rules first: `CLAUDE.md`, `.claude/rules/*`, `CONTRIBUTING.md`. If they forbid committing unless the user asks for it, leave your finished work in the working tree and say so in your return. Otherwise commit it. Use `type(scope): short imperative message`, lowercase, no trailing period. `type` is one of `feat`, `fix`, `chore`, `docs`, `test`, `refactor`, `perf`, `style`, `ci`, `build`, `revert`. `scope` is the affected package or area, and you omit it when the change is repo-wide.
> - Write your full report to the report file whose path you were given: the branch, your commits with SHA and subject or the fact that the work is uncommitted and left in the working tree, the seams you tested and the commands you ran, and anything that deviates from the rows you were given. If the file already exists, a previous implementer attempted this unit, so append rather than overwrite. No file contents, no tool transcripts.
> - Then return four short things and nothing else: the branch, your commits or that the work is uncommitted, the seams you tested, and your deviations in one line each. The report file carries the detail, so keep the return to what the orchestrator needs to order the next step.
>
> Pass `isolation: "worktree"` when a wave has two or more implementers, since they mutate files in parallel and would otherwise conflict, and spawn them from the current working branch so each worktree branches off it rather than off the base. A lone implementer needs no worktree: it commits to the working branch directly, and there's nothing to merge. Implementers in a wave commit regardless of what the repo's git rules say, since a worktree's work only reaches the working branch through a merge, so when those rules forbid committing, the round runs a single implementer on the working branch instead of a wave.
>
> **Escalate rather than repeat.** An implementer that dies, returns nothing, or reports red tests gets one retry with a fresh subagent on the same rows, and the retry moves up to the model **Model Selection** names for escalation. Give it the failed implementer's report path with this framing: a prior implementer attempted this unit and failed, read the report for what was tried. Round 3 is escalated the same way for every implementer in it, since a unit that has survived two rounds usually means the implementer can't see its own problem, and fresh eyes and a capability bump belong in the same move. Note the escalation in the ledger.
>
> If the retry fails too, stop the run and report which unit failed and what it left behind. Never merge a failed branch, since half a change is worse than none.
>
> **4. Merge.** With two or more implementers, merge each returned branch into the working branch. The split guaranteed disjoint files, so expect no conflicts. If a merge conflicts anyway, stop and report it. Don't auto-resolve.
>
> **5. Verify.** Start the full test suite and typecheck with `run_in_background: true`, and don't wait on them. Nothing step 6 reads depends on the result: the review is built from committed history, and only the exit decision at the end of that step needs both. So run the spec's `## Verification` steps against the merged result while the suite works, then go to step 6 and collect the suite there. What this trades is that a red suite has already paid for the round's reviews, and since a red round needs those findings anyway, what's lost is spend rather than correctness. All of this runs every round, so the status you eventually report always describes the code as it stands after the last fix landed. Compare what fails against your round 1 baseline and say which failures are inherited, since a failure that was already there is not this change's to fix.
>
> **6. Review, then decide whether to loop.** Which review you run depends on the round.
>
> **Round 1** runs **Standards-and-Spec Review** (same Reference section) once against the total diff, using the base branch you were given as its fixed point. One pass over everything catches the cross-cutting issues no single implementer could see, which is why they skip it. Write its aggregated report to `r1-review.md` in the run folder and keep only the sorted findings below in your context.
>
> **Rounds 2 and 3** run **Scoped Re-Review** (also in the Reference section) against the fix diff alone, with the findings this round was dispatched to fix handed over verbatim. Write the report to `r<R>-rereview.md`. A fresh whole-diff review can't tell you whether a finding was fixed: it either mentions the finding again or it doesn't, and absence isn't a verdict. The re-review verdicts each one by name instead.
>
> Those findings are on disk whether or not you still remember them. Round 2's come from `r1-review.md`, round 3's from `r2-rereview.md`, in both cases the ones you sorted as hard.
>
> Either way, record the HEAD the review saw in the ledger's review line. The head you write now is the next round's fix base, so a missing one leaves the following round with nothing to diff against.
>
> Collect step 5's backgrounded suite and typecheck before you sort, and record them in the ledger. Findings sorted without a test status are findings you can't decide an exit on.
>
> Sort what round 1 returns:
>
> - **Hard**: a correctness bug, a spec requirement that's missing, partial, or implemented wrong, or a breach of a documented repo standard. These force another round.
> - **Judgement call**: anything from the smell baseline, and anything the review labelled as one. These ride along only when they sit in a file a forced unit is already touching. Otherwise they go in your return, unfixed, with the reason.
> - **Needs a human**: a finding that contradicts the approved plan, or that turns on a decision the plan never settled. No round can fix that without asking someone, so record it as a deviation and leave the code alone.
>
> A re-review has already done most of that sorting for you. A finding it returns NOT ADDRESSED stays open and stays hard. New breakage in the fix diff joins the open set at the severity it was given, and sorts by the same three rules. Out-of-scope observations go in the ledger and reach the user through your return: they never enter the loop, since a loop that accepts new findings from untouched code has no reason to end.
>
> If hard findings remain and you've run fewer than 3 rounds, compose a remediation work units table with the same columns as the spec's, one unit per fix, and go back to step 2. That table may only address findings this review reported, and the spec's `## Out of Scope` still binds it: a fix round is not an opening to redesign.
>
> Otherwise you're done. No hard findings is the clean exit, which for a re-review means every finding addressed and no new hard breakage in the fix diff. Hitting round 3 with hard findings still open is the capped exit. Say which one it was.
>
> Record the exit in the ledger, then return six things and nothing else: the commits landed with SHA and subject, or that the work is uncommitted and left in the working tree; the final test and typecheck status, with any inherited failures named as such; the review findings you fixed and the ones you left, with a reason for each; anything that deviates from the spec; every ruling you made, each with what it costs if you got it wrong; and the ledger's absolute path. No file contents, no tool transcripts.
>
> The rulings matter as much as the commits. Each one is a decision you took on the user's behalf because nobody upstream could be reached, and a clean run deletes the run folder, so a ruling that only ever existed in the ledger is a decision made in secret.

Keep only that returned summary in your context. The subagent's internal work, meaning file reads, edits, test runs, and review rounds, stays out of your conversation, and the run folder holds the detail if you ever need it.

# 4. Review

## Review the code

Spawn a new general-purpose subagent, not the Execute one, on the model **Model Selection** names for review. A fresh session reads the diff cold, while the agent that orchestrated the change carries every assumption it made into the review. It needs no worktree isolation, since it reads a diff and changes nothing. Give it the base branch you pinned, the run folder's absolute path, and what the Execute summary reported: the commit list, or that the work is uncommitted. Its brief:

> What needs reviewing depends on what the Execute summary reported. With commits, it's every commit this branch added on top of the base branch you were given. With the work left uncommitted, it's the working tree.
>
> Check your scope before reviewing. With commits, `git log <base-branch>..HEAD --oneline` has to list the commits you were given. Uncommitted, `git status --porcelain` has to come back non-empty. Either way, an empty scope means the target is wrong rather than the code being clean, so stop and report that instead of returning an empty table.
>
> Then run the review. With commits, `/code-review medium <base-branch>`, so the review covers the whole divergence rather than the working tree. Uncommitted, `/code-review medium` with no target, so it reviews the working tree. `medium` rather than `low`, because this is the only pass in the skill hunting correctness bugs and `low` trades recall for confidence. Don't pass `--comment`, since there's no PR to comment on, and don't pass `--fix`, since the user decides what to act on.
>
> This review is yours to run. Don't spawn a subagent to review part of the diff and don't spawn a second reviewer for another opinion: the skill already gives this diff every review seat it gets, so one you spawn duplicates a seat at full cost and its verdict counts for nothing. A diff too large for one pass gets several passes from you, and you say so.
>
> The review reports through the `ReportFindings` tool rather than as text, so transcribe what it reports into this table yourself, keeping its ordering, which is already most severe first:
>
> ```markdown
> | #   | Category            | Finding                               |
> | --- | ------------------- | ------------------------------------- |
> | 1   | <its category slug> | <finding, with a file:line reference> |
> ```
>
> Write the table to `findings.md` in the run folder you were given, then return it. Nothing else, and don't change any code.

This pass is not the Standards-and-Spec Review that ran inside Execute, or the re-reviews that verified its fixes. Those check conformance to the repo's standards and to the plan, and their hard findings were fixed in the loop. This one hunts correctness bugs, and what it finds goes to the user.

## Stop

Show the user the Execute summary and the findings table, then stop. develop is done. What to do about the findings is the user's call, and the numbered rows are there so they can point at the ones they want. When the work was left uncommitted, say that plainly and offer to commit it, rather than reporting as though it had landed.

Then resolve the run folder. A run is clean when the findings table is empty and Execute reported its clean exit rather than the capped one: delete `.develop/<feature>/` and say you did, since git history is the record from there. Anything else leaves something for the user to act on, so keep the folder and name its absolute path, pointing at `ledger.md` as the place that explains how the run went and at the report files behind each finding.

---

# Reference

The full definitions the steps above point to. Follow one completely whenever a step sends you here.

## Test-Driven Development

The red → green loop, plus what makes that loop produce tests worth keeping. Every part below applies on every cycle, so read it before the loop and consult it during, not after.

**What a good test is**: Tests verify behavior through public interfaces, not implementation details. Code can change entirely; tests shouldn't. A good test reads like a specification: "user can checkout with valid cart" tells you exactly what capability exists, and it survives refactors because it doesn't care about internal structure.

**Seams: where tests go**: A **seam** is the public boundary you test at, the interface where you observe behavior without reaching inside. Tests live at seams, never against internals.

The seams are already decided. They arrive with your spec in the `Test seams` column of the work units table, approved by the user along with the plan, so test at those and add none. A unit whose seams are `none` writes no tests at all, because what it changes isn't behaviour, and the orchestrator verifies it through the plan's `## Verification` instead. You can't test everything, and fixing the seams before implementation starts is how the effort lands on critical paths and complex logic instead of spreading over every edge case. If implementation shows that a listed seam is wrong, or that a critical one is missing, report it in your final summary as a deviation from the spec rather than quietly testing somewhere else.

**Anti-patterns**:

- **Implementation-coupled**: mocks internal collaborators, tests private methods, or verifies through a side channel (querying the database instead of using the interface). The tell: the test breaks when you refactor but behavior hasn't changed.
- **Tautological**: the assertion recomputes the expected value the way the code does (`expect(add(a, b)).toBe(a + b)`, a snapshot derived by hand the same way, a constant asserted equal to itself), so it passes by construction and can never disagree with the code. Expected values must come from an independent source of truth: a known-good literal, a worked example, the spec.
- **Horizontal slicing**: writing all tests first, then all implementation. Bulk tests verify _imagined_ behavior: you test the _shape_ of things rather than user-facing behavior, the tests go insensitive to real changes, and you commit to test structure before understanding the implementation. Work in **vertical slices** instead: one test, one implementation, repeat, each test a **tracer bullet** that responds to what the last cycle taught you.

**Rules of the loop**:

- **Red before green.** Write the failing test first, then only enough code to pass it. Don't anticipate future tests or add speculative features.
- **Watch it fail, for the right reason.** Run the new test before you write any implementation and read the failure. It has to fail because the behaviour is missing, not because of a typo, an import error, or a broken test setup, and it must not pass on the first run. A test whose first failure you never read is a test you can't claim exercises anything, and a test that passes immediately is testing behaviour that already existed.
- **Stub the target before you write the test.** A test against a module that doesn't exist yet fails on the import, and an import error is the one failure the rule above tells you not to count. So create the function, type, or endpoint first, returning a fixed value that is honestly wrong: `ok: true` where the case expects a rejection, an empty list where it expects rows. The test then runs, compares, and fails on the value, which is a failure about the behaviour being missing. Three lines of stub is what buys you the reading this loop depends on. Don't make the stub clever, though. A stub that guesses at the real logic can satisfy a case by accident, and a case that passes on its first run tells you nothing.
- **One slice at a time.** One seam, one test, one minimal implementation per cycle.
- **Refactoring is not part of the loop.** Standards-and-Spec Review reports what needs restructuring, and a later round implements those fixes once tests pass. Neither belongs inside the red → green cycle.

## Model Selection

Every spawn in this skill names its model, because an unnamed model is whatever the session defaults to, and that default is as likely to be too weak for a review as it is to be wasted on a one-file edit. Briefs name the row they want; this table is the mapping. The first row is the exception: nobody spawns `develop`, so that row is what the session should be set to before the skill starts.

| Work             | Where                                                                              | Model    |
| ---------------- | ---------------------------------------------------------------------------------- | -------- |
| `develop` itself | holding the run                                                                    | `opus`   |
| Lookups          | `Explore` agents in Draft                                                          | `sonnet` |
| Design           | `Plan` agents in Draft                                                             | `opus`   |
| Orchestration    | Draft and Execute subagents                                                        | `opus`   |
| Implementation   | every implementer, before escalation                                               | `sonnet` |
| Escalation       | a retry, and every implementer in round 3                                          | `opus`   |
| Review           | both Standards-and-Spec axes, the Scoped Re-Review, and the final correctness pass | `opus`   |

Four rules bind it:

- **Name it on every spawn**, including the ones a subagent makes of its own.
- **Implementation starts cheap.** Every implementer runs on `sonnet`, however many files its unit touches. The plan settled the design and the round's scan ruled on whatever it left ambiguous, so a unit that still needs an implementer to decide how things fit is a defect one layer up, and a bigger model there hides it rather than surfacing it. What this trades is the implementation that passes its own tests and is still wrong, which nothing can detect at dispatch time and which round 1's review is there to catch.
- **Escalate on failure**, never repeat. A retry or a round 3 moves up a tier, since the same model on the same rows produces the same result.
- **Never downgrade a review.** A review that misses a bug costs a whole round, which is more than the review ever saves. The Scoped Re-Review looks like the exception, since its findings are handed to it and its diff is small, but it's the last thing standing between a bad fix and the branch. Its diff being small is also why downgrading it saves so little.

## Standards-and-Spec Review

Two-axis review of the diff between `HEAD` and a fixed point:

- **Standards**: does the code conform to this repo's documented coding standards?
- **Spec**: does the code faithfully implement the plan it was given?

Both axes run as parallel subagents so they don't pollute each other's context, then you aggregate their findings.

**1. Pin the fixed point.** It's the base branch you were given, already resolved for you, so don't go looking for one and don't assume a trunk name. Confirm the ref resolves (`git rev-parse <fixed-point>`) and the diff is non-empty before going further, so a bad ref fails here instead of inside two parallel subagents.

Then write the diff to a file once, `r<R>-diff.txt` in the run folder, `<R>` being the round you're in. Redirect it, never print it: the whole point is that the diff reaches the reviewers without passing through you.

```bash
{
  echo "## Commits";       git log <fixed-point>..HEAD --oneline
  echo; echo "## Files changed"; git diff --stat <fixed-point>...HEAD
  echo; echo "## Not in the diff below"; echo "Lockfiles and go.sum. The stat above shows what they changed."
  echo; echo "## Diff";    git diff -U10 <fixed-point>...HEAD -- . \
    ':(exclude)*.lock' ':(exclude)package-lock.json' \
    ':(exclude)pnpm-lock.yaml' ':(exclude)npm-shrinkwrap.json' ':(exclude)go.sum'
} > <run folder>/r<R>-diff.txt
```

Three-dot on the diffs so the comparison runs against the merge-base. `-U10` rather than the default three lines of context because that's what lets a reviewer judge a hunk without opening the file, and opening the file is the expensive path: one `Read` per hunk, with the whole file landing in the reviewer's context. Then check its length with `wc -l`. Past 1500 lines, regenerate it at `-U3` and say so in your aggregation. The budget is lines rather than bytes because a `Read` returns 2000 of them and stops, and the reviewer briefs below tell the reviewer not to go looking anywhere else, so a package over that gets reviewed in part by an agent that believes it saw all of it. 1500 leaves room for the commit list and the stat above the diff.

The exclusion list is files no human wrote. A lockfile has no documented standard to conform to and no spec requirement to verify, and one dependency bump can add thousands of lines that both axes then read for nothing. Test snapshots are deliberately not on that list and must never be added to it: an implementer that satisfies a failing test by rewriting its snapshot is exactly what the Spec axis is there to catch.

**2. The spec is `plan.md`** in the run folder you were given. There's nothing to look up and nobody to ask, and it's never the remediation table, which is derived from the plan and so can't be the thing that judges it. This review runs in round 1; fix rounds run **Scoped Re-Review** instead.

**3. Identify the standards sources.** Anything in the repo that documents how code should be written, such as `CODING_STANDARDS.md` or `CONTRIBUTING.md`.

On top of whatever the repo documents, the Standards axis always carries the **smell baseline** below: a fixed set of Fowler code smells (_Refactoring_, ch.3) that applies even when a repo documents nothing. Two rules bind it:

- **The repo overrides.** A documented repo standard always wins. Where it endorses something the baseline would flag, suppress the smell.
- **Always a judgement call.** Each smell is a labelled heuristic ("possible Feature Envy"), never a hard violation. Like any standard here, skip anything tooling already enforces.

Each smell reads _what it is_ → _how to fix_; match it against the diff:

- **Mysterious Name**: a function, variable, or type whose name doesn't reveal what it does or holds. → rename it; if no honest name comes, the design's murky.
- **Duplicated Code**: the same logic shape appears in more than one hunk or file in the change. → extract the shared shape, call it from both.
- **Feature Envy**: a method that reaches into another object's data more than its own. → move the method onto the data it envies.
- **Data Clumps**: the same few fields or params keep travelling together (a type wanting to be born). → bundle them into one type, pass that.
- **Primitive Obsession**: a primitive or string standing in for a domain concept that deserves its own type. → give the concept its own small type.
- **Repeated Switches**: the same `switch`/`if`-cascade on the same type recurs across the change. → replace with polymorphism, or one map both sites share.
- **Shotgun Surgery**: one logical change forces scattered edits across many files in the diff. → gather what changes together into one module.
- **Divergent Change**: one file or module is edited for several unrelated reasons. → split so each module changes for one reason.
- **Speculative Generality**: abstraction, parameters, or hooks added for needs the spec doesn't have. → delete it; inline back until a real need shows.
- **Message Chains**: long `a.b().c().d()` navigation the caller shouldn't depend on. → hide the walk behind one method on the first object.
- **Middle Man**: a class or function that mostly just delegates onward. → cut it, call the real target direct.
- **Refused Bequest**: a subclass or implementer that ignores or overrides most of what it inherits. → drop the inheritance, use composition.

**4. Spawn both subagents in parallel**, both on the model **Model Selection** names for review. Two rules bind what you write into either brief:

- **Don't steer the review.** Nothing in a brief may tell the reviewer what not to flag, how severe something may be at most, or that the plan chose something deliberately. If what you're writing contains "don't flag", "at most a judgement call", or "the plan chose this", delete it: you're pre-judging to spare yourself a round. Let the review raise it and sort it in step 5 below, where the labels are yours to assign anyway.
- **Neither axis spawns anything.** Say so in both briefs. A reviewer a reviewer spawns is a duplicate seat at full cost, and its verdict counts for nothing.

Standards subagent brief:

> <the absolute path of the diff package> is your view of the change: the commit list, the files changed, and the full diff with ten lines of context around every hunk. Read it start to finish: if it runs longer than one `Read` returns, keep reading with `offset` until you reach the end, say how many passes it took, and never review a partial package. Its context lines are the changed files, so don't re-run git and don't open a changed file unless a hunk you have to judge is cut off, in which case say which one in your report.
>
> Do this review yourself. Don't spawn a subagent for any part of it and don't spawn a second reviewer for another opinion.
>
> <the standards-source files found in step 3>
>
> Read the **smell baseline** under `## Standards-and-Spec Review` in the skill file at <the absolute path you were given>. It's a fixed set of code smells the Standards axis carries on top of whatever the repo documents.
>
> Report, per file or hunk where relevant, (a) every place the diff violates a documented standard, citing the standard by file and rule, and (b) any baseline smell you spot, naming it and quoting the hunk. Distinguish hard violations from judgement calls: a documented-standard breach can be hard, baseline smells are always judgement calls, and a documented repo standard overrides the baseline. Skip anything tooling enforces. Under 400 words.

Spec subagent brief:

> <the absolute path of the diff package> is your view of the change: the commit list, the files changed, and the full diff with ten lines of context around every hunk. Read it start to finish: if it runs longer than one `Read` returns, keep reading with `offset` until you reach the end, say how many passes it took, and never review a partial package. Its context lines are the changed files, so don't re-run git and don't open a changed file unless a hunk you have to judge is cut off, in which case say which one in your report.
>
> Do this review yourself. Don't spawn a subagent for any part of it and don't spawn a second reviewer for another opinion.
>
> <the absolute path of `plan.md`> is the spec. Read it first, and judge the diff against it.
>
> Report (a) requirements the spec asked for that are missing or partial, (b) behaviour in the diff that nothing asked for, which is scope creep, and (c) requirements that look implemented but where the implementation looks wrong. Quote the spec line for each finding. Under 400 words.

**5. Aggregate.** Assemble both reports under `## Standards` and `## Spec` headings, verbatim or lightly cleaned. Don't merge or rerank findings across the axes, for the reason below.

Label every finding **hard** or **judgement call**, since that label is what whoever ran this acts on. A breach of a documented repo standard is hard, and so is every Spec finding: a missing requirement, unasked-for behaviour, and a requirement implemented wrong all are. Every smell-baseline finding is a judgement call.

End with a one-line summary: total findings per axis, how many of those are hard, and the worst issue _within each axis_ if there is one. Don't pick a single winner across axes, since that's the reranking the separation exists to prevent.

**Why two axes**: A change can pass one axis and fail the other:

- Code that follows every standard but implements the wrong thing → **Standards pass, Spec fail.**
- Code that does exactly what the plan asked but breaks the project's conventions → **Spec pass, Standards fail.**

Reporting them separately stops one axis from masking the other.

## Scoped Re-Review

A fix round's gate, run in place of Standards-and-Spec Review in every round after the first. It isn't a fresh review: the whole-diff standards and spec judgement already happened in round 1, and this pass answers a narrower question. Did the fixes fix the findings, and did they break anything on the way?

**1. Build the fix diff package.** The fix base is the HEAD the previous round's review saw, recorded in the ledger. Confirm it resolves and that `git diff <fix-base>..HEAD` is non-empty: an empty fix diff means the fix round produced nothing, which is a failed round, not a clean one. Then write the package exactly as Standards-and-Spec Review's step 1 does, to `r<R>-diff.txt`, over `<fix-base>..HEAD`.

**2. Spawn one subagent** on the model **Model Selection** names for review. One agent, not two axes: the findings are handed to it rather than hunted, so there's nothing for a second axis to see that the first would miss. The same two rules from Standards-and-Spec Review's step 4 bind this brief too, so don't steer it and don't let it spawn.

> <the absolute path of the diff package> is the fix diff: the fix commits, the files they changed, and the full diff with ten lines of context. Read it start to finish: if it runs longer than one `Read` returns, keep reading with `offset` until you reach the end, say how many passes it took, and never review a partial package. Don't re-run git, and don't open a changed file unless a hunk you have to judge is cut off.
>
> <the absolute path of `plan.md`> is the spec. A fix that satisfies a finding by breaking a requirement is still a failure.
>
> <the absolute paths of the report files for the units fixed this round>: what the implementers say they did. Treat every line of it as an unverified claim and check it against the diff. A stated rationale is the implementer grading its own work, so "left it deliberately" or "kept it simple" never lowers a finding's severity.
>
> The findings under verification, in order:
>
> <the previous round's open findings, copied verbatim, one per line>
>
> Do this review yourself. Don't spawn a subagent for any part of it and don't spawn a second reviewer for another opinion.
>
> Return three sections and nothing else.
>
> **Finding verdicts**: every finding above, in the order given, each **ADDRESSED** or **NOT ADDRESSED** with a `file:line`. Attempted is not addressed. The specific defect has to be gone, and a fix that moves the defect somewhere else in the diff is NOT ADDRESSED with the new location named.
>
> **New breakage in the fix diff**: anything the fix itself broke or introduced, each labelled hard or judgement call with a `file:line`, judged against the spec and the repo's documented standards. Write "none" when it's clean.
>
> **Out-of-scope observations**: anything you noticed outside the fix diff. Name it in one line each and don't investigate further. These don't block the round.
>
> Under 400 words.

**3. Aggregate.** Keep the three sections as they came back, and end with a one-line summary: how many findings were addressed, how many are still open, and how much new breakage there is. Don't rerank an open finding to a judgement call, since a finding the previous round called hard stays hard until the defect is gone.

**Why scoped**: a fresh whole-diff review can't answer the question a fix round asks. It reports whatever it finds on a diff that grew, and a finding it doesn't mention this time reads as fixed whether it was fixed, missed, or simply crowded out. Absence is not a verdict. Restricting the scope to the fix diff also keeps the loop finite, since a review free to range over untouched code can always find one more reason to run another round.
