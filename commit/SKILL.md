---
name: commit
description: Turns file changes into commits. Use whenever the user asks to commit changes, create commits, or clean up the working tree. Not for amending a single existing commit.
---

# Commit

Turns a messy working tree into a readable git history: a handful of commits, each doing one coherent thing, each named consistently.

The temptation is to just `git add -A && git commit -m "updates"`. Resist it — that produces a commit nobody can review or bisect. The other failure mode is over-splitting: one commit per file is just as unreadable as one giant commit, because it destroys the signal of "these files changed together for a reason." The goal is commits that match the actual units of work.

## Commit message format

Always use this format, across every repo — don't infer or match a different convention from `git log`:

```
type(scope): short imperative message
```

- **type** — one of `feat`, `fix`, `chore`, `docs`, `test`, `refactor`, `perf`, `style`, `ci`, `build`, `revert`.
- **scope** — the affected package/app/area (e.g. `admin-web`, `services`, `api`, `deps`). Optional: include it when the change is localized to one package or area, omit it (`type: message`, no parens) when the change is repo-wide, cross-cutting, or scope would be redundant.
- **message** — lowercase, imperative mood ("add", not "added"/"adds"), no trailing period. Keep the subject short; add a body only when the subject + diff don't already explain the "why".

Examples:

```
feat(admin-web): add pagination to list endpoint
fix: handle null response from signer
chore(deps): bump drizzle-kit to 0.24
test: cover empty-cart edge case
```

## Workflow

### 1. Confirm you're on a feature branch

```
git branch --show-current
```

Never create commits directly on `main` or `dev` (or `master`, if that's the repo's trunk name). If the current branch is one of those, stop and tell the user — don't commit and don't create/switch branches yourself without asking, since branch naming (`feature/…`, `fix/…`, `chore/…`, etc.) and ticket linkage are the user's call. Only proceed with the rest of this workflow once you're on a proper `feature/`, `fix/`, `chore/`, or similarly-scoped branch.

### 2. Survey the damage

```
git status --short
git diff --stat
```

Get the full shape of what changed before deciding anything. Don't start staging files based on filenames alone — a file named `list.ts` could be a brand-new feature or a one-line typo fix, and you won't know which grouping it belongs to until you've looked.

### 3. Read the actual diffs, not just filenames

For everything in `git status`, read the diff (`git diff <path>` for unstaged, `git diff --cached <path>` if something's already staged). A filename tells you what area of the code changed; the diff tells you _what changed and why it belongs with other changes_. This is the step that makes correct grouping possible — skipping it and grouping by directory or filename alone produces plausible-looking but wrong groupings.

While reading, actively watch for:

- **Mechanical/incidental changes riding along with real work** — a formatter reflowing an unrelated file, a lockfile bump, a stray reformatting of a file nobody meant to touch. These usually deserve their own small commit rather than being folded into a feature commit, since they have a different "reason" than the feature does.
- **Local tooling/editor state that shouldn't be committed at all** — things like editor config directories, local settings files, scratch files. If something in `git status` looks like machine-local state rather than project work, leave it out of every commit (don't just lump it into whichever commit is convenient).

### 4. Group changes into logical commits, in dependency order

A group is a set of files that share a _reason_ for changing — usually a feature, a fix, or a mechanical maintenance task. Common patterns:

- A feature's implementation, its tests, its infra/config wiring, and its docs update usually belong in **one** commit, even though they span directories — a reviewer wants to see the whole feature at once, not "the code" in one commit and "the docs for that code" three commits later.
- A shared library/package change that several apps depend on is its own commit, separate from the apps that consume it.
- Pure mechanical changes (lockfile regeneration, a rename, a formatter pass on an unrelated file, a version bump) get their own small commit(s), separate from feature work, so a reviewer isn't stuck reading dependency-bump churn inside a feature diff.
- Unrelated fixes discovered along the way don't belong in the feature commit they were noticed during — give them their own commit.

There's no fixed "right number" of commits — a small change might legitimately be one commit, a big one might be six or seven. Let the actual seams in the diff decide, not a target count.

Once grouped, order them by dependency: when one group depends on another (an app consumes a shared package, a migration must run before the code that assumes it), commit the dependency first. This keeps each commit buildable/reviewable in isolation as much as possible. A reasonable default order: shared packages/libraries → apps/services that consume them → infra/config → docs → lockfile-only changes last (since the lockfile reflects the cumulative effect of everything before it).

### 5. Stage and commit each group explicitly

```
git add <specific file> <specific file> ...
git commit -m "$(cat <<'EOF'
type(scope): short message

Optional body line explaining why, if the subject line alone
doesn't carry enough context.
EOF
)"
```

Always name the specific files being staged — never `git add -A` or `git add .`. A blanket add silently pulls in whatever's sitting in the working tree at that moment, including things from a later group or things that shouldn't be committed at all (see step 3's tooling-state warning). Explicit staging is what makes the grouping from step 4 actually stick.

Use a heredoc so multi-line messages and quoting come through correctly, and follow the commit message format defined above.

### 6. Verify

```
git status --short
git log --oneline -<N>
```

Working tree should be clean (or show only genuinely-excluded files, e.g. local tooling state you deliberately left out — mention these to the user). The log should read as a coherent story of the work, each entry short and following the format above.

## Notes

- If the user has already staged some files in a way that doesn't match good grouping, it's fine to `git reset` (not `--hard`) to unstage everything and re-stage deliberately per group — `git reset` alone only unstages, it doesn't touch the working tree.
- If you're unsure whether two changes belong in one commit or two, err toward the grouping a reviewer would want: "can I understand and evaluate this commit without needing the next one loaded too?"
- Never use destructive git operations (`reset --hard`, `checkout --`, `clean -f`) as part of this workflow — everything here only needs `add`, `commit`, and non-destructive `reset` to unstage.
