# um-ref-merge

A [Claude Code](https://claude.com/claude-code) skill that manages a local
install of the [`um-ref`](https://github.com/UltraMessaging/um-ref) skill
against its upstream git repo. It handles three workflows:

- **Install** a fresh copy of `um-ref` into `~/.claude/skills/um-ref/`.
- **Update** an existing install with newer upstream content while preserving
  local customizations, via a 3-way merge anchored at the installed VERSION.
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

- **BASE** — the release the user originally installed, identified from the
  `VERSION` file in the active skill and looked up as a `release-<VERSION>`
  tag in the upstream repo. (A pre-1.0 fallback uses `um-ref.tgz` history.)
- **OURS** — current upstream `um-ref/` (from `origin/main`).
- **THEIRS** — the user's active `~/.claude/skills/um-ref/`.

It stages BASE/OURS/THEIRS in a scratch git repo, runs `git merge`, applies
policy overrides for auto-generated files (`VERSION`, `java_api.md`,
`dotnet_api.md`, `config-data.xml`, `index-ume.m4`, `index-dro.m4`), guides
conflict resolution, and lands the merged result in the user's active
skill. For Contribute, it also squashes the merged content onto `main`
in the upstream clone, ready to push.

## Prerequisites

- Claude Code installed and working.
- A git clone of <https://github.com/UltraMessaging/um-ref> that you run
  Claude Code in when invoking this skill, with a clean `git status`.
- For Update / Contribute: the `um-ref` skill installed at
  `~/.claude/skills/um-ref/` with your local improvements.

## Install this skill

Skills are loaded from `~/.claude/skills/<skill-name>/`. Install this one
by copying the `um-ref-merge/` directory from this repo into that location:

    git clone https://github.com/UltraMessaging/um-ref-merge.git
    mkdir -p ~/.claude/skills
    cp -a um-ref-merge/um-ref-merge ~/.claude/skills/

To pick up updates later, `git pull` in the clone and re-copy:

    cd um-ref-merge
    git pull
    cp -a um-ref-merge/. ~/.claude/skills/um-ref-merge/

Verify the install by starting Claude Code and running `/help` — the
skill's description should appear in the available-skills list.

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
   - Back up your active skill to `~/.claude/skills/um-ref.pre-merge/`.
   - Read `VERSION` from the active skill, resolve `release-<VERSION>` in
     the repo, and extract BASE.
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
