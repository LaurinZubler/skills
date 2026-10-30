---
name: pull-request
description: Writes the description for a pull request. Use whenever the user asks to open a PR, write or rewrite a PR description, or fill in a PR body. Not for reviewing the code in a PR.
---

`pull-request` in this document means you, the agent running this skill. Write the description for a pull request.

A PR description has one reader, the person who has to review the change without your last three hours of context. Give them the problem, the shape of the solution, and a way to check it. Then stop.

Two failure modes. The first is the empty template, every heading present, each filled with a sentence that restates its own title ("**Why?** To fix the bug."). The second is the diff in prose, a paragraph per changed file, which the reviewer can read faster in the diff itself. What the diff can't tell them is why, and which decisions were judgement calls.

Length follows the change. A one-line fix gets three lines. A 40-file refactor gets 300 words. Never pad a section to fill it out.

This skill only reads. It runs `git` and `gh` read commands and nothing else. Don't run the test suite, the linter, the typechecker, or a build in order to write a description. CI runs all of that on every PR and posts the verdict on the PR itself, where the reviewer already sees it. A local run adds nothing to the description and costs minutes.

## Title

Same format as a commit subject:

```
type(scope): short imperative message
```

`type` is one of `feat`, `fix`, `chore`, `docs`, `test`, `refactor`, `perf`, `style`, `ci`, `build`, `revert`. `scope` is the affected package or area, omitted when the change is repo-wide. The message is lowercase, imperative, no trailing period.

The title lands in the merge commit and in every list of PRs anyone ever scans, so it has to say what the change does. `fix(admin-web): expire idle sessions after 30 minutes`, not `session fixes` or `#412`.

## The description

Three core sections, always. Then optional blocks, each with a trigger. Include a block when its trigger is true, delete it entirely when it isn't.

### Core

**`## Context`.** The problem in one or two sentences, as it stood before this branch. What was broken, missing, or slow, and what it cost. Link the issue here (`Closes #412`) rather than in a section of its own. Be concrete. "Sessions were kept until the cookie expired 14 days later, so an unattended browser stayed authenticated" beats "improves session security".

**`## Summary`.** An overview of what changed, as bullets. One line each, six or so at most. Say what the change does at the level someone skimming needs, and name the one symbol or path that anchors it (`TTL enforced in SessionStore.get()`).

Resist explaining the mechanism. How the TTL sweep works, which flex property collapses, why the regex is shaped that way, all of it belongs in the code or a comment beside it, not here. If a bullet needs a second sentence to make sense, the detail is too deep for this section. Cut it and trust the diff.

**`## Test plan`.** A markdown task list, one line per test. No commands, no paths to run. `- [x]` for anything already done, `- [ ]` for anything still open.

Automated tests are ticked, because CI ran them. Read the verdict off the PR with `gh pr checks <number>` rather than running anything yourself; if a check is red, the test plan is not the place to report it, fix the code. Each ticked line names what the test pins down, not the file it lives in.

Manual checks stay unticked, so the reviewer can see at a glance what is left. One line each, concrete enough to follow: which route, which window size, what should happen.

Leave e2e out entirely, ticked or not. It runs against a deployed environment on its own schedule, so its state says nothing about the commit under review. E2E specs the branch added or changed are part of the diff, not a line in the test plan.

```
## Test plan

- [x] Idle sessions expire at 30 minutes, and 29:59 still authenticates
- [x] A missing `lastSeen` stamp is treated as expired, not as fresh
- [ ] Log in on staging, idle 31 minutes, refresh: back to the login screen
```

### Optional blocks

| Block | Include when |
| --- | --- |
| `## Risk & rollback` | The change can break something in a deployed environment. Two lines at most: what breaks, and how to back out. |
| `## Migration` | Schema, data, or config changed. Migration files, backfill steps, ordering against the deploy. |
| `## Feature flags` | Anything ships gated. Flag name, default state, ramp plan. |
| `## Security & privacy` | Auth, roles, permissions, secrets, PII, or an external boundary is touched. |
| `## Performance` | The change is about performance, or plausibly regresses it. Numbers, not adjectives. |
| `## Docs` | Docs or rules were updated, or should have been and weren't. Say which. |
| `## Follow-ups` | A ticket was created during the work, or the user explicitly named something as a follow-up. Link it. Nothing else belongs here: don't invent suggestions, don't list debt you happened to notice, and drop the block entirely when neither applies. |
| `## Related PRs` | This is one of a stack, or depends on another PR merging first. |

Nothing else, and no `## Screenshots` block: images never get attached here, so a heading asking for them is a heading nobody fills. No checklist of tickboxes. They get rubber-stamped, and a rubber-stamped tickbox is worse than none at all because it looks like a signal. No "Type of change" radio list either; the title's `type` already carries it.

## What not to write

- **"This PR...".** The reader knows. Start with the substance.
- **Headings left empty**, or filled with `n/a`, or holding a `<!-- delete if not applicable -->` comment nobody deleted. Absent means not applicable.
- **A file-by-file walkthrough.** That's the diff's job.
- **Follow-up work you thought of yourself.** A PR description records the change, it doesn't pitch the next one. Unless there's a ticket or the author asked for it, leave it out.
- **Adjectives doing a number's work.** "Significantly faster", "much cleaner", "robust". Measure it or drop it.
- **Pass or fail claims.** "All tests green", "214 passing". CI already said so on the PR, and if it didn't, you're guessing.
- **Apologies and hedges.** "Sorry this is big", "hope this makes sense". Say what's true and stop.

## 1. Establish the range

```
git branch --show-current
gh pr status --json number,baseRefName,title,body
```

If a PR already exists for the branch, you're rewriting its body. Read the current one first and keep whatever in it is still true. If not, find the base branch. It's whatever the branch actually forked from, which in a GitFlow repo is usually `dev`, and for a stacked PR is the parent feature branch. Don't assume the repository default branch.

## 2. Read the diff and the commits

```
git log --oneline <base>..HEAD
git diff --stat <base>...HEAD
git diff <base>...HEAD
```

Read the diff itself, not just the stat. The commit subjects tell you how the work was grouped, the diff tells you what the change does and which parts matter. You need both. A description written from commit subjects alone reads like a changelog and misses every decision that never made it into a subject line.

Where a claim in the description can be checked, check it. If you're going to write that a spec guards the boundary, open the spec. If you're going to write that nothing else calls the function, grep for it. A confidently wrong PR description costs a reviewer more than no description at all.

## 3. Collect the links

Find the issue this closes, from the branch name, the commit bodies, or by asking. If an issue drove the work, read it (`gh issue view <n> --comments`). The comments are usually where the requirement got amended, and the amended version is what the reviewer judges against.

## 4. Pick the blocks

Walk the optional-block table and check each trigger against what you read in step 2. Decide once, before writing, instead of working it out while typing. Then write only the blocks that survived.

## 5. Write it

Core sections first, optional blocks after, in table order. Prose in `## Context`, bullets in `## Summary`. Markdown-link every issue, CI run, and PR you reference so the reviewer can click through.

Show the draft to the user before it goes anywhere. They know things about the change that aren't in the diff.

## 6. Deliver it

Write the body to a file and pass the file. Piping markdown through shell quoting mangles backticks and newlines.

```
gh pr create --base <base> --title "<title>" --body-file <path>
gh pr edit <number> --body-file <path>
```

Run these only when the user asked for it in that turn. Writing a description is not permission to open or update a PR, and never permission to push.

## Report

Show the user the title and the body. If you created or updated the PR, give its link.
