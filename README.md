# um-ref-merge

A [Claude Code](https://claude.com/claude-code) skill that helps a contributor
merge their locally-improved `um-ref` skill back into the upstream
[UltraMessaging/um-ref](https://github.com/UltraMessaging/um-ref) repo via a
proper 3-way merge.

The companion skill it operates on is `um-ref`, which lives at
<https://github.com/UltraMessaging/um-ref> and — once installed — at
`~/.claude/skills/um-ref/`.

## What it does

Between a contributor's install and current upstream HEAD, both the maintainer
and the contributor have been editing the skill in parallel with no shared git
history. `um-ref-merge` walks Claude through a proper 3-way merge:

- **BASE** — the release the contributor originally installed from (identified
  from the local git clone's HEAD, reflog, or `um-ref.tgz` history).
- **OURS** — current upstream HEAD.
- **THEIRS** — the contributor's improved `~/.claude/skills/um-ref/`.

It stages BASE/OURS/THEIRS in a scratch git repo, runs `git merge`, applies
policy overrides for auto-generated files (`java_api.md`, `dotnet_api.md`,
`config-data.xml`, `index-ume.m4`, `index-dro.m4`), guides conflict resolution,
and lands the merged result both in the contributor's active skill and on a
branch in the upstream clone ready for a PR.

## Prerequisites

- Claude Code installed and working.
- The `um-ref` skill installed at `~/.claude/skills/um-ref/` with your local
  improvements.
- A clean git clone of <https://github.com/UltraMessaging/um-ref> that you run
  Claude Code in when invoking this skill. `git status` should be clean; do
  **not** `git pull` immediately before invoking the skill — the current
  HEAD vs. `origin/main` gap is used as the primary signal for identifying
  BASE.

## Install

Skills are loaded from `~/.claude/skills/<skill-name>/`. Install this one by
copying the `um-ref-merge/` directory from this repo into that location:

    git clone https://github.com/UltraMessaging/um-ref-merge.git
    mkdir -p ~/.claude/skills
    cp -a um-ref-merge/um-ref-merge ~/.claude/skills/

To pick up updates later, `git pull` in the clone and re-copy:

    cd um-ref-merge
    git pull
    cp -a um-ref-merge/. ~/.claude/skills/um-ref-merge/

If you'd rather not clone the repo, you can fetch `SKILL.md` directly. This
works today because the skill is a single file; if it later grows helper
files, this shortcut will silently miss them.

    mkdir -p ~/.claude/skills/um-ref-merge
    cd ~/.claude/skills/um-ref-merge
    wget https://github.com/UltraMessaging/um-ref-merge/raw/main/um-ref-merge/SKILL.md

(Use `curl -O <url>` if `wget` isn't installed.)

Verify the install by starting Claude Code and running `/help` — the skill's
description should appear in the available-skills list.

## Use

1. `cd` into your git clone of the upstream `um-ref` repo.
2. Confirm `git status` is clean and you haven't just `git pull`ed.
3. Start Claude Code and invoke the skill by name:

        /um-ref-merge

   Or ask in natural language — the description triggers on prompts like
   "merge my um-ref skill changes into this repo" or "contribute my um-ref
   skill improvements back."

4. Claude will:
   - Back up your active skill to `~/.claude/skills/um-ref.pre-merge/`.
   - Identify BASE, set up the 3-way merge, and apply policy overrides.
   - Ask you about any conflicts that need judgement.
   - Update `~/.claude/skills/um-ref/` in place.
   - Stage the merged content on a branch in the upstream clone and show
     you `git diff --stat` before committing.
5. Review the diff. If you're happy, tell Claude to commit; then run
   `git push` and `gh pr create` yourself when ready.

## When *not* to use this

- **Maintainer release upgrades.** If you're the `um-ref` maintainer cutting a
  new release from an updated UM source tree, follow `um-ref/RELEASE_UPGRADE.md`
  in the upstream repo instead — this skill is for contributor merges.
- **Trivial contributions.** If you only added a single new deep-dive file
  and made no `SKILL.md` edits, you don't need 3-way merge machinery — just
  copy the file into `um-ref/` on a branch and open a PR.
- **Architectural rewrites.** If you rewrote the skill from scratch, open a
  GitHub issue describing the intent first so the maintainer can weigh in
  before you invest in a mechanical merge.

## Contributing

Improvements to this skill are welcome via PR against
<https://github.com/UltraMessaging/um-ref-merge>. Note that changes to
`um-ref-merge/SKILL.md` are what actually affect behavior — the README is
just install/usage documentation for humans.
