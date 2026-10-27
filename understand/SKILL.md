---
name: understand
description: Interviews you about a feature or change until it is clear, verifies the facts the input asserts, and returns a checkpoint of what was settled and what is out of scope. Use before planning or building, when a request is too vague to plan against, or when another skill asks for it. Not for writing the plan and not for writing code.
---

`understand` in this document means you, the top-level agent running this skill. Everything it spawns is a subagent, and no subagent can reach the user, so every question for the user happens in one of your own turns.

Interview the user about an idea until it's clear enough to plan against, then hand back a checkpoint summary of what was settled. This skill decides nothing about implementation, proposes no design, and writes no code. It ends the moment the understanding is agreed.

Every spawn below uses the same shape: a sentence telling you what to spawn, then the brief in a quote block. Everything inside the quote block goes into the subagent's prompt. Everything outside it is yours.

**Models.** Run this skill on `opus`, and spawn every `Explore` agent below on `sonnet`. Grilling is the highest-judgement work there is in building a feature: a requirement settled wrong here reaches the plan and gets built, and nothing downstream is looking for it. If you're on something weaker, say so in your first turn and offer the user `/model opus` before the interview starts. Name the model on every spawn, because an unnamed model is whatever the session happens to default to.

## Get context

Figure out what the user wants from whatever you were given: a written prompt, a GitHub issue (number or link), a linked file, or the arguments a calling skill passed on. If nothing was given, detect a GitHub issue from the branch name, recent commits, or session history. If that finds nothing, ask directly what they want to do.

When the input is a GitHub issue, read it with `gh issue view <number> --comments` and carry its body into the Checkpoint `## Context`. Read the comments too, since that's usually where an issue gets amended.

## Research

Look up only what the issue or idea names explicitly: a file, module, or function. This bounds how far you go into the codebase, not how much of the input you read. Leave open questions and possibilities for grilling.

Spawn an `Explore` subagent for the lookup instead of reading files yourself, so the raw file contents stay out of your context and you keep only the answer.

Verify every number the input asserts. A row count, a threshold, a "13 rows" in a table, an expected filter result: check it against the real data before it reaches the checkpoint. Specs go stale between being written and being built, and a control number that no longer holds is the cheapest possible place to catch a changed requirement. Report what you found either way, and flag any figure that moved.

## Grill

Interview the user about the context and what Research found until you reach a shared understanding. Map decisions as a **design tree**: each decision branches into the decisions that depend on it. Treat scope boundaries as a normal branch, since "what's out of scope" is a question like any other.

Work the tree one question at a time. The **frontier** is every decision whose prerequisites are already settled: the questions you can ask _now_ without guessing at answers you haven't heard yet. Pick the single most important open frontier question and ask it.

Ask through `AskUserQuestion`, one question per call, recommended answer first. Shape it like so:

- `header`: the question's title, two or three words, 12 characters at most. This is the chip the UI draws above the question, and it's the only styled text in the call.
- `question`: the question sentence alone on the first line, then a blank line, then the context that justifies it, as many plain-prose paragraphs as it takes. Name code in plain words rather than in backticks.
- `options`: the recommended answer first, the rest after.

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

- **Yes**: the understanding is settled, so finish below.
- **No**: ask what's missing or wrong, add it to the tree as a new branch, and work the frontier again.

## Finish

Return the summary. It's the whole product of this skill, and whatever comes next, a plan, a build, or nothing, works from it.

When another skill invoked you, stop there. Its pipeline carries the summary onward and it has its own stops for the user, so don't add one it didn't ask for.

Otherwise offer to save the summary, once. It's in the conversation, and the conversation ends. If the user says yes, write it to `.develop/<feature>/checkpoint.md` and say the absolute path. `<feature>` is the branch name with the type prefix stripped and any remaining slashes flattened to dashes: `feat/42-oauth-refresh` gives `42-oauth-refresh`. On a trunk, or a branch with no recognizable shape, slugify the topic instead. Check `git check-ignore -q .develop` and say so in the same turn if the folder isn't ignored, but don't edit `.gitignore`: that's the user's call, made once per repo.
