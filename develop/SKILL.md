---
name: develop
description: Builds a feature.
disable-model-invocation: true
---

`develop` in this document means you, the top-level agent running this skill. Everything the skill spawns is a subagent, and no subagent can reach the user, so every question for the user happens in one of your own turns.

Develop the current idea end to end. The only skill it calls out to is `/code-review`, a Claude Code built-in. Everything else it needs is defined here, and the definitions too long to inline live once in the Reference section at the bottom, with the steps that need them pointing there.

Subagents can't read this file unless you tell them to. When a brief points at a Reference section, pass the subagent the absolute path of this file, which is `<skill-directory>/SKILL.md`. Claude Code gives you this skill's directory when the skill loads.

Only general-purpose subagents can spawn subagents of their own. `Explore` and `Plan` have no `Agent` tool, which is why every orchestrator below is general-purpose.

Every spawn below uses the same shape: a sentence telling you what to spawn, then the brief in a quote block. Everything inside the quote block goes into the subagent's prompt. Everything outside it is yours.

# 1. Understand

## Get context

Figure out what the user wants from whatever they gave `/develop`: a written prompt, a GitHub issue (number or link), or a linked file. If nothing was given, detect a GitHub issue from the branch name, recent commits, or session history. If that finds nothing, ask directly what they want to do.

When the input is a GitHub issue, read it with `gh issue view <number> --comments` and carry its body into the Checkpoint `## Context`. Read the comments too, since that's usually where an issue gets amended.

## Research

Look up only what the issue or idea names explicitly: a file, module, or function. This bounds how far you go into the codebase, not how much of the input you read. Leave open questions and possibilities for grilling.

Spawn an `Explore` subagent for the lookup instead of reading files yourself, so the raw file contents stay out of your context and you keep only the answer.

## Grill

Interview the user about the context and what Research found until you reach a shared understanding. Map decisions as a **design tree**: each decision branches into the decisions that depend on it. Treat scope boundaries as a normal branch, since "what's out of scope" is a question like any other.

Work the tree one question at a time. The **frontier** is every decision whose prerequisites are already settled: the questions you can ask _now_ without guessing at answers you haven't heard yet. Pick the single most important open frontier question and ask it.

Ask through `AskUserQuestion`, one question per call, recommended answer first. Shape it like so:

```
## <question title>
<question body, might be multiple paragraphs>
- <recommended answer first>
- <other suggestions>
```

Wait for the answer before asking anything else. Each answer reshapes the tree: settled decisions unblock the questions that depended on them. Recompute the frontier and ask the next single question. A question that depends on another still-open question comes later, not now.

Finding facts is your job, not the user's. When a frontier question needs a fact from the environment, spawn an `Explore` subagent instead of asking. Don't block on it: treat a running exploration as an unsettled prerequisite and keep asking other frontier questions while it runs. The decision itself belongs to the user, so put it to them and wait.

Continue until the frontier is empty and you have visited every branch.

## Checkpoint

Write a structured summary with these sections:

```markdown
## Context

<the issue or idea that started this>

## What Research found

<the facts your Explore agents came back with, so the next step doesn't re-derive them>

## Settled Decisions

<what grilling settled, one item per decision>

## Out of Scope

<what was explicitly excluded>
```

Show it to the user and ask if they want to continue.

- **Yes**: continue to Draft the plan.
- **No**: ask what's missing or wrong, add it to the tree as a new branch, and work the frontier again.

# 2. Plan

Call `EnterPlanMode` before spawning the draft agent, unless the session is already in plan mode. Get approval below exits plan mode with the finished plan, and that's the gate the whole rest of the skill hangs on.

## Draft the plan

Spawn a general-purpose subagent. It needs no worktree isolation, since this work is read-only. Give it the Checkpoint summary and the absolute path of this file. Its brief:

> Work read-only: no `Edit`, no `Write`, no state-mutating `Bash` command, no commits, no `gh` writes. This binds you and every agent you spawn.
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
> The `Test seams` column is the whole test plan: the user approves these along with the plan, and implementers test at these seams and nowhere else, so a seam you leave out goes untested. `Depends on` is what lets the orchestrator order the work, so name the unit number a unit waits on, or `none`. Units marked `none` with no overlapping files run together; a unit with a dependency runs once what it waits on has landed.
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

You're the only actor in this skill who can ask, so ask when the answer isn't clean: several refs tie, or the winner is itself a feature branch. Otherwise state the base you picked and move on.

## Execute

Spawn a general-purpose subagent. It needs no worktree isolation, since the result has to land on the current working branch. Give it the approved plan as its spec, the base branch you pinned, and the absolute path of this file. Its brief:

> You orchestrate, you don't implement. Every line of this change is written by an agent you spawn, fixes included, so nothing you spawn ever reviews its own work.
>
> Nobody upstream can be reached while you run. Anything you can't resolve goes in your return, never into a question.
>
> You work in **rounds**. Round 1 implements the approved spec. Each later round implements fixes for what the previous round's review found. You get 3 rounds at most.
>
> **1. Decide the split.** Work from a work units table: the spec's `## Work units` in round 1, the remediation table you composed in step 5 in every round after that. It names the units, their files, their reuse targets, their seams, and their ordering.
>
> Units whose `Depends on` is `none` and whose `Files` don't overlap run together as one wave. Once a wave lands, the units waiting on it become eligible and form the next wave. A table you can't read this way, because the dependencies are ambiguous or the file lists are vague, collapses to a single implementer for the whole round: a wrong split costs more in merge conflicts than it saves in tokens.
>
> **2. Spawn the implementers.** One general-purpose subagent per unit in the wave, or exactly one for the whole round when the table didn't decompose. You never write the code yourself.
>
> Give each one the absolute path of the skill file, the spec's `## Context`, `## Approach` and `## Out of Scope` sections, its own rows from the work units table, and these instructions:
>
> - Implement with **Test-Driven Development**, defined in the Reference section of the skill file at the path you were given. Read it before you start. Test at the seams your rows list and nowhere else.
> - Run typechecking and single test files as you go. When a test stays red, fix the code until it passes. Driving the implementation from failing tests is the point of the loop, so nothing here hands red tests back up.
> - Don't run the full test suite and don't run Standards-and-Spec Review. Both happen once per round, above you.
> - Nobody upstream can be reached while you run. Report, don't ask.
> - Commit your finished work. Use `type(scope): short imperative message`, lowercase, no trailing period. `type` is one of `feat`, `fix`, `chore`, `docs`, `test`, `refactor`, `perf`, `style`, `ci`, `build`, `revert`. `scope` is the affected package or area, and you omit it when the change is repo-wide.
> - Return four things and nothing else: the branch you committed to, your commits with SHA and subject, the seams you tested, and anything that deviates from the rows you were given. No file contents, no tool transcripts.
>
> Pass `isolation: "worktree"` when a wave has two or more implementers, since they mutate files in parallel and would otherwise conflict, and spawn them from the current working branch so each worktree branches off it rather than off the base. A lone implementer needs no worktree: it commits to the working branch directly, and there's nothing to merge.
>
> An implementer that dies, returns nothing, or reports red tests gets one retry with a fresh subagent on the same rows. If the retry fails too, stop the run and report which unit failed and what it left behind. Never merge a failed branch, since half a change is worse than none.
>
> **3. Merge.** With two or more implementers, merge each returned branch into the working branch. The split guaranteed disjoint files, so expect no conflicts. If a merge conflicts anyway, stop and report it. Don't auto-resolve.
>
> **4. Verify.** Run the full test suite and typecheck, then the spec's `## Verification` steps, once against the merged result. This runs every round, so the status you eventually report always describes the code as it stands after the last fix landed.
>
> **5. Review, then decide whether to loop.** Run **Standards-and-Spec Review** (same Reference section) once against the total diff, using the base branch you were given as its fixed point. One pass over everything catches the cross-cutting issues no single implementer could see, which is why they skip it.
>
> Sort what it returns:
>
> - **Hard**: a correctness bug, a spec requirement that's missing, partial, or implemented wrong, or a breach of a documented repo standard. These force another round.
> - **Judgement call**: anything from the smell baseline, and anything the review labelled as one. These ride along only when they sit in a file a forced unit is already touching. Otherwise they go in your return, unfixed, with the reason.
> - **Needs a human**: a finding that contradicts the approved plan, or that turns on a decision the plan never settled. No round can fix that without asking someone, so record it as a deviation and leave the code alone.
>
> If hard findings remain and you've run fewer than 3 rounds, compose a remediation work units table with the same columns as the spec's, one unit per fix, and go back to step 1. That table may only address findings this review reported, and the spec's `## Out of Scope` still binds it: a fix round is not an opening to redesign.
>
> Otherwise you're done. A review with no hard findings is the clean exit, and hitting round 3 with hard findings still open is the capped exit. Say which one it was.
>
> Return four things and nothing else: the commits landed with SHA and subject; the final test and typecheck status; the review findings you fixed and the ones you left, with a reason for each; and anything that deviates from the spec. No file contents, no tool transcripts.

Keep only that returned summary in your context. The subagent's internal work, meaning file reads, edits, test runs, and review rounds, stays out of your conversation.

# 4. Review

## Review the code

Spawn a new general-purpose subagent, not the Execute one. A fresh session reads the diff cold, while the agent that orchestrated the change carries every assumption it made into the review. It needs no worktree isolation, since it reads a committed diff and changes nothing. Give it the base branch you pinned and the commit list from the Execute summary. Its brief:

> Everything is already committed, so there's no uncommitted diff to review. What needs reviewing is every commit this branch added on top of the base branch you were given.
>
> Check your scope before reviewing: `git log <base-branch>..HEAD --oneline` has to list the commits you were given. If it comes back empty, stop and report that instead of returning an empty table, since an empty scope means the target is wrong rather than the code being clean.
>
> Then run `/code-review medium <base-branch>`, so the review covers the whole divergence rather than the working tree. `medium` rather than `low`, because this is the only pass in the skill hunting correctness bugs and `low` trades recall for confidence. Don't pass `--comment`, since there's no PR to comment on, and don't pass `--fix`, since the user decides what to act on.
>
> The review reports through the `ReportFindings` tool rather than as text, so transcribe what it reports into this table yourself, keeping its ordering, which is already most severe first:
>
> ```markdown
> | #   | Category            | Finding                               |
> | --- | ------------------- | ------------------------------------- |
> | 1   | <its category slug> | <finding, with a file:line reference> |
> ```
>
> Return the table and nothing else. Don't change any code.

This pass is not the Standards-and-Spec Review that ran inside Execute. That one checks conformance to the repo's standards and to the plan, and its hard findings were fixed in the loop. This one hunts correctness bugs, and what it finds goes to the user.

## Stop

Show the user the Execute summary and the findings table, then stop. develop is done. What to do about the findings is the user's call, and the numbered rows are there so they can point at the ones they want.

---

# Reference

The full definitions the steps above point to. Follow one completely whenever a step sends you here.

## Test-Driven Development

The red → green loop, plus what makes that loop produce tests worth keeping. Every part below applies on every cycle, so read it before the loop and consult it during, not after.

**What a good test is**: Tests verify behavior through public interfaces, not implementation details. Code can change entirely; tests shouldn't. A good test reads like a specification: "user can checkout with valid cart" tells you exactly what capability exists, and it survives refactors because it doesn't care about internal structure.

**Seams: where tests go**: A **seam** is the public boundary you test at, the interface where you observe behavior without reaching inside. Tests live at seams, never against internals.

The seams are already decided. They arrive with your spec in the `Test seams` column of the work units table, approved by the user along with the plan, so test at those and add none. You can't test everything, and fixing the seams before implementation starts is how the effort lands on critical paths and complex logic instead of spreading over every edge case. If implementation shows that a listed seam is wrong, or that a critical one is missing, report it in your final summary as a deviation from the spec rather than quietly testing somewhere else.

**Anti-patterns**:

- **Implementation-coupled**: mocks internal collaborators, tests private methods, or verifies through a side channel (querying the database instead of using the interface). The tell: the test breaks when you refactor but behavior hasn't changed.
- **Tautological**: the assertion recomputes the expected value the way the code does (`expect(add(a, b)).toBe(a + b)`, a snapshot derived by hand the same way, a constant asserted equal to itself), so it passes by construction and can never disagree with the code. Expected values must come from an independent source of truth: a known-good literal, a worked example, the spec.
- **Horizontal slicing**: writing all tests first, then all implementation. Bulk tests verify _imagined_ behavior: you test the _shape_ of things rather than user-facing behavior, the tests go insensitive to real changes, and you commit to test structure before understanding the implementation. Work in **vertical slices** instead: one test, one implementation, repeat, each test a **tracer bullet** that responds to what the last cycle taught you.

**Rules of the loop**:

- **Red before green.** Write the failing test first, then only enough code to pass it. Don't anticipate future tests or add speculative features.
- **One slice at a time.** One seam, one test, one minimal implementation per cycle.
- **Refactoring is not part of the loop.** Standards-and-Spec Review reports what needs restructuring, and a later round implements those fixes once tests pass. Neither belongs inside the red → green cycle.

## Standards-and-Spec Review

Two-axis review of the diff between `HEAD` and a fixed point:

- **Standards**: does the code conform to this repo's documented coding standards?
- **Spec**: does the code faithfully implement the plan it was given?

Both axes run as parallel subagents so they don't pollute each other's context, then you aggregate their findings.

**1. Pin the fixed point.** It's the base branch you were given, already resolved for you, so don't go looking for one and don't assume a trunk name. Confirm the ref resolves (`git rev-parse <fixed-point>`) and the diff is non-empty before going further, so a bad ref fails here instead of inside two parallel subagents.

Capture the diff command once: `git diff <fixed-point>...HEAD`, three-dot so the comparison runs against the merge-base. Note the commit list too, via `git log <fixed-point>..HEAD --oneline`.

**2. The spec is the approved plan** you were given as your spec. There's nothing to look up and nobody to ask. In a fix round it's still that original plan, never the remediation table, since the remediation table is derived from the plan and can't be the thing that judges it.

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

**4. Spawn both subagents in parallel.**

Standards subagent brief:

> <the diff command and the commit list>
>
> <the standards-source files found in step 3>
>
> Read the **smell baseline** under `## Standards-and-Spec Review` in the skill file at <the absolute path you were given>. It's a fixed set of code smells the Standards axis carries on top of whatever the repo documents.
>
> Report, per file or hunk where relevant, (a) every place the diff violates a documented standard, citing the standard by file and rule, and (b) any baseline smell you spot, naming it and quoting the hunk. Distinguish hard violations from judgement calls: a documented-standard breach can be hard, baseline smells are always judgement calls, and a documented repo standard overrides the baseline. Skip anything tooling enforces. Under 400 words.

Spec subagent brief:

> <the diff command and the commit list>
>
> <the approved plan>
>
> Report (a) requirements the spec asked for that are missing or partial, (b) behaviour in the diff that nothing asked for, which is scope creep, and (c) requirements that look implemented but where the implementation looks wrong. Quote the spec line for each finding. Under 400 words.

**5. Aggregate.** Assemble both reports under `## Standards` and `## Spec` headings, verbatim or lightly cleaned. Don't merge or rerank findings across the axes, for the reason below.

Label every finding **hard** or **judgement call**, since that label is what whoever ran this acts on. A breach of a documented repo standard is hard, and so is every Spec finding: a missing requirement, unasked-for behaviour, and a requirement implemented wrong all are. Every smell-baseline finding is a judgement call.

End with a one-line summary: total findings per axis, how many of those are hard, and the worst issue _within each axis_ if there is one. Don't pick a single winner across axes, since that's the reranking the separation exists to prevent.

**Why two axes**: A change can pass one axis and fail the other:

- Code that follows every standard but implements the wrong thing → **Standards pass, Spec fail.**
- Code that does exactly what the plan asked but breaks the project's conventions → **Spec pass, Standards fail.**

Reporting them separately stops one axis from masking the other.
