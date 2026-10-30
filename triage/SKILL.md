---
name: triage
description: Walks through a list one item at a time and has you decide an action for each, without carrying any of them out, then prints a summary of every decision. Use when the user asks to go through, triage, or decide on a list of items (review findings, issues, TODOs, emails, open questions) one by one. Not for executing the decided actions.
---

`triage` in this document means you, the top-level agent running this skill. No subagent can reach the user, so every question for the user happens in one of your own turns.

Go through a list one item at a time, get an action decided for each one, and record it. Execute nothing. The skill ends with a summary, and acting on it is a separate request.

## Find the list

Take the list from what the user gave you: an argument, a pasted list, a file, a link, or a list that's already in the conversation, such as review findings or a table from an earlier step. When it's ambiguous which list is meant, ask once. Never merge two lists on your own.

Number the items in their original order and keep that numbering throughout. Say how many items there are before you start.

## Decide, one item at a time

For each item, in order:

1. Collect what's needed to recommend an action. Read the referenced file, issue, or line if that's what the item points at. That's read-only work: no edits, no commands with side effects, no messages sent.
2. Ask with `AskUserQuestion`, one item per call:
   - `header`: the item number, e.g. `#3 of 10`.
   - `question`: the item in one line, then a blank line, then the context that matters for deciding, in plain prose.
   - `options`: 2 to 4 concrete actions, your recommendation first and marked `(Recommended)`. Make each option an action someone could carry out later ("Fix in WidgetDetailView: pass additionalFilter", "Skip: out of scope", "Open a follow-up issue"), not a vague verdict. Include a skip or defer option whenever one is plausible.
3. Record the answer as given. When the user picks "Other", record their words, not a paraphrase. If the answer leaves the action unclear, ask one follow-up about that same item before moving on.

Keep going until every item has a decision. Don't batch several items into one question, don't skip ahead, and don't revisit an item unless the user asks to.

## Don't execute

Deciding is the whole job. Even when an action is a one-line fix, or the user's answer reads like "do it", record it as the decision and move on to the next item. Executing comes after the summary, and only when the user asks for it in a new turn.

## Keep the decisions on disk

Append one line per decision to `triage.md` in the session's scratchpad directory, or in `/tmp` if there isn't one, as soon as it's made:

```
#<n> | <item, one line> | <decided action>
```

A long list can outlast your context, and that file is how a compacted session picks up where it left off. If it already exists for this list, read it first and continue from the first undecided item.

## Summary

Once every item is decided, print the summary as a table in the original order:

```markdown
| #   | Item           | Action           |
| --- | -------------- | ---------------- |
| 1   | <item, short>  | <decided action> |
```

Below it, group the counts by kind of action (e.g. 4 fix, 3 skip, 1 follow-up issue), and give the path of the decisions file. Then stop. Don't start on any action, and don't offer to unless the user asks what comes next.
