# um-ref-merge

A [Claude Code](https://claude.com/claude-code) skill that manages a local
install of the [`um-ref`](https://github.com/UltraMessaging/um-ref) skill
against its upstream git repo. It handles three workflows:

- **Install** a fresh copy of `um-ref` into `~/.claude/skills/um-ref/`.
- **Update** an existing install with newer upstream content while preserving
  local customizations, via a 3-way merge anchored at the commit the user's
  clone was installed from.
- **Contribute** local `um-ref` improvements back upstream by squashing the
  merged result onto `main` and pushing. The upstream team uses trunk-based
  development; there is no PR review flow.

All three assume the user is running Claude inside a git clone of the
upstream `um-ref` repo.

## What it does

`um-ref` is a knowledge-heavy skill that users are encouraged to extend —
adding domain knowledge as they work with Claude. Between an install and
current upstream HEAD, both the maintainer and the user may have been
editing the skill in parallel with no shared git history. For Update and
Contribute, `um-ref-merge` walks Claude through a proper 3-way merge:

- **BASE** — the commit the user's `um-ref` clone is checked out at.
  Users are told to keep the clone intact at their install commit, so its
  current state is BASE. This skill trusts that state; it does not run
  `git checkout` on the user's behalf.
- **OURS** — current upstream `um-ref/` (from `origin/main`).
- **THEIRS** — the user's active `~/.claude/skills/um-ref/`.

It creates a working branch off BASE, overlays the active skill, commits,
then merges `origin/main`. Policy overrides pull auto-generated files
(`java_api.md`, `dotnet_api.md`, `config-data.xml`, `index-ume.m4`,
`index-dro.m4`) unconditionally from upstream. Claude guides conflict
resolution and lands the merged result in the user's active skill. For
Contribute, it also squashes the merged content onto `main` in the upstream
clone, ready to push.

## Prerequisites

- Claude Code installed and working.
- A git clone of <https://github.com/UltraMessaging/um-ref> that you run
  Claude Code in when invoking this skill, with a clean `git status`.
- For Update / Contribute: the `um-ref` skill installed at
  `~/.claude/skills/um-ref/` with your local improvements.

## Install this skill

Clone this repo somewhere convenient:

    git clone https://github.com/UltraMessaging/um-ref-merge.git

Then, in that directory, launch Claude Code and ask:

```
Please install the skill in this repo.
```

To pick up updates later, launch Claude Code in the clone and ask:

```
Please update um-ref-merge from this repo.
```

## Use

1. `cd` into your git clone of the upstream `um-ref` repo.
2. Confirm `git status` is clean.
3. Start Claude Code and invoke the skill by name:

        /um-ref-merge

   Or ask in natural language — the description triggers on prompts like
   "install the um-ref skill", "update my um-ref skill", or "contribute
   my um-ref skill improvements back."

4. Claude will pick the workflow based on state and intent:
   - No `~/.claude/skills/um-ref/` yet → **Install** (a straight copy from
     the repo clone).
   - Active skill exists, you asked to update → **Update** (3-way merge,
     write result back to the active skill).
   - You asked to contribute / merge back → **Contribute** (same merge,
     plus squash onto `main` in the repo clone, ready to push).

5. For Update / Contribute, Claude will:
   - Back up your active skill to `/tmp/$USER-um-ref.pre-merge/`
     (any prior backup at that path is removed first).
   - Trust your clone's current commit as BASE (verify with a quick
     `diff -rq` sanity check against the active skill).
   - Run the 3-way merge and apply policy overrides.
   - Ask you about any conflicts that need judgement.
   - Overlay the merged tree onto `~/.claude/skills/um-ref/`.

6. For Contribute, Claude will additionally stage the merged content on
   a working branch in the upstream clone and show you `git diff --stat`
   before committing. Review the diff — if you're happy, tell Claude to
   commit. Claude then squashes the working branch onto `main` for one
   clean commit and runs `git push`. `git push` only happens on your
   explicit go-ahead.

   The upstream team uses trunk-based development; contributors without
   write access on the upstream repo should open a GitHub issue
   describing the change instead.

## When *not* to use this

- **Maintainer UM release upgrades.** If you're the `um-ref` maintainer
  cutting a new release from an updated UM source tree, follow
  `um-ref/RELEASE_UPGRADE.md` in the upstream repo instead — this skill
  is for user-facing flows.
- **Architectural rewrites.** If you rewrote the skill from scratch,
  open a GitHub issue describing the intent first so the maintainer can
  weigh in before you invest in a mechanical merge.

## Contributing

This repo uses trunk-based development — the team pushes improvements
directly to `main` on <https://github.com/UltraMessaging/um-ref-merge>.
There is no PR review flow. Contributors without write access should
open a GitHub issue describing the change instead.

Note that changes to `um-ref-merge/SKILL.md` are what actually affect
behavior — the README is just install/usage documentation for humans.
