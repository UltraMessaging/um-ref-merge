---
name: um-ref-merge
description: Manage a Claude Code install of the `um-ref` skill against its upstream git repo (https://github.com/UltraMessaging/um-ref) — install a fresh copy, update an existing install while preserving local customizations, or contribute local changes back by pushing to `main` (the upstream team uses trunk-based development, no PR review). Triggers on prompts like "install the um-ref skill", "update my um-ref skill", "please update the /um-ref skill from this repo", "merge my um-ref changes back", "contribute my um-ref skill improvements". Requires the user to be running Claude inside a git clone of the upstream `um-ref` repo. For Update and Contribute, the clone must be checked out at the commit corresponding to the user's install (their BASE) — that one-time manual setup replaces all the BASE-guessing machinery earlier versions of this skill needed. Do NOT trigger on unrelated skills or on general git-merge tasks — this skill knows about `um-ref`'s specific file layout and would give wrong answers for other skills.
---

# um-ref-merge — install, update, and contribute for the `um-ref` skill

## Scope

Three workflows against the upstream `um-ref` repo
(https://github.com/UltraMessaging/um-ref):

- **Install** — copy the skill from a fresh repo clone into
  `~/.claude/skills/um-ref/`. No active skill exists yet.
- **Update** — pull newer skill content into an existing active
  skill, preserving whatever local edits the user (or Claude on their
  behalf) has made.
- **Contribute** — same merge as Update, plus squash the result onto
  `main` and push. The upstream team uses trunk-based development;
  there is no PR review flow.

Update and Contribute both rely on the user's repo clone being
**already checked out at the commit corresponding to their install**
— see [User-provided precondition](#user-provided-precondition)
below. That one-time manual setup is what lets this skill stay simple.

If the user is the **um-ref maintainer** doing a UM version upgrade of
the skill, this is the wrong tool — direct them to
`um-ref/RELEASE_UPGRADE.md`.

**Applicability.** If the current working directory is not a clone of
the upstream `um-ref` repo, treat the user's prompt as if this skill
does not exist — defer to any `CLAUDE.md` in the current repo or to
other matching skills.

## Choosing the workflow

- `~/.claude/skills/um-ref/` **does not exist** → **Install**.
- `~/.claude/skills/um-ref/` exists, user asks to update / sync / pick
  up the latest → **Update**.
- User asks to contribute / merge changes back → **Contribute**.

If intent is ambiguous, ask.

## Install workflow

Preconditions:

- Working directory is a clone of the upstream repo.
- `~/.claude/skills/um-ref/` does **not** exist. If it does, the user
  is Updating, not Installing.

Steps:

    mkdir -p ~/.claude/skills
    cp -a "$REPO_ROOT/um-ref" ~/.claude/skills/

Tell the user to **keep the clone intact** at its current commit.
That commit is their BASE for any future Update or Contribute — the
clone itself is the record of where their install came from.

## User-provided precondition

For **Update** and **Contribute**, the user's repo clone must be
checked out at BASE — the commit their active skill was installed
from. The expected default is that they never moved off it: install
copies `um-ref/` out of the clone, and users are told to leave the
clone alone until they Update or Contribute. If they did move off
(e.g. to browse upstream), they check out BASE again before invoking
this skill.

This skill trusts that state. It does not try to identify BASE for
the user, and it does not run `git checkout`.

**Verify BASE.** Before doing anything else, sanity-check that the
checked-out `um-ref/` matches the active skill closely:

    diff -rq ~/.claude/skills/um-ref/ um-ref/ 2>&1 | head -30

Expect only files the user has intentionally touched, plus any new
files they have added.

If nearly every file differs, or the diff shows content the user
does not recognize as their own edits, BASE is wrong. **Stop and
report the diff to the user.** Do not try to recover, do not guess
at a different BASE, do not run `git checkout` on their behalf — a
wrong BASE produces a subtly incorrect merge. Ask the user to
re-verify their checkout and re-invoke the skill.

## Update workflow

### Step 1 — Backup and set variables

Always back up before overwriting anything in the active skill. The
backup lives in `/tmp` (username-scoped) so it doesn't pollute the
active skill set. If a backup from an earlier run is already there,
delete it first — the current active skill is what matters, not last
run's snapshot:

    rm -rf "/tmp/${USER}-um-ref.pre-merge"
    cp -a ~/.claude/skills/um-ref "/tmp/${USER}-um-ref.pre-merge"

Tell the user about the backup and how to restore from it.

Fetch upstream:

    cd "$REPO_ROOT"
    git fetch origin

### Step 2 — Create a working branch off BASE

    git checkout -b um-ref-merge-work

Working directory is at BASE (the user already checked out BASE per
the precondition). The new branch will hold their improvements and
then the merge.

### Step 3 — Overlay the active skill onto the working tree

Replace `um-ref/` in the clone with the contents of the active
skill, excluding the active skill's own `.git` if it has one:

    find um-ref -mindepth 1 -maxdepth 1 -exec rm -rf {} +
    (cd ~/.claude/skills/um-ref && \
     find . -mindepth 1 -maxdepth 1 ! -name '.git' -print0 | \
     xargs -0 -I{} cp -a {} "$REPO_ROOT/um-ref/")

Sanity-check with `git status` and `git diff --stat`. The diff should
match what the user has actually changed — nothing more. If it looks
wildly wrong, BASE is wrong; restore from backup and re-verify.

### Step 4 — Commit the user's improvements

    git add -A
    git commit -q -m 'user improvements'

### Step 5 — Merge upstream into the working branch

    git merge origin/main

Most of the tree is now merged automatically. Git leaves conflict
markers only where the two sides changed the same lines.

### Step 6 — Policy overrides

Certain files are **not hand-authored** — they're generated from UM
source by `build.sh` or bundled from UM's doc-source tree. They must
always match the upstream release, never a user's version.

| File               | Origin                                       |
| ------------------ | -------------------------------------------- |
| `java_api.md`      | Generated by `gen_java_api.py`               |
| `dotnet_api.md`    | Generated by `gen_dotnet_api.py`             |
| `config-data.xml`  | Bundled from UM `doc/Config/reference/`      |
| `index-ume.m4`     | Bundled from UM `doc/UME/`                   |
| `index-dro.m4`     | Bundled from UM `doc/DRO/`                   |

Take upstream's version of each unconditionally, regardless of merge
state:

    cd um-ref
    for f in java_api.md dotnet_api.md config-data.xml \
             index-ume.m4 index-dro.m4; do
      if git show origin/main:um-ref/"$f" >/dev/null 2>&1; then
        git show origin/main:um-ref/"$f" > "$f"
        git add "$f"
      fi
    done
    cd ..

If any of these show up as user-modified in the pre-merge diff, flag
it — that's almost certainly a mistake (they regenerated locally
with a different UM source version, or hand-edited a bundled
artifact). Do not silently accept those edits.

### Step 7 — Resolve remaining conflicts

Run `git status`. For each unmerged file, read it, understand both
sides' intent, and apply a **conservative semantic merge**. Git has
already auto-resolved everything mechanically resolvable; what
remains is edits on overlapping lines that need judgment. Classify
each conflict block and act accordingly:

- **Additive, independent** — both sides added related but
  non-conflicting content in the same region (e.g. both appended a
  row to a table, both added a bullet to a list). Keep both sides
  in an order that reads well. Silent auto-resolve is fine here.
- **Additive, potentially redundant** — both sides added content
  covering similar ground from different angles. Attempt a merged
  paragraph that captures both intents; if you can't do so without
  loss, present the options to the user and ask.
- **Contradictory** — both sides changed the same fact, rule,
  default value, code example, or claim, and both can't be true.
  **Stop and ask** which version should win, and why. Never
  silently pick one; never fabricate a compromise.
- **Structural rewrite vs. small edit** — maintainer rewrote a
  section that the user made minor edits to. Ask whether the user's
  edit still applies to the rewritten version.

Never silently drop either side's changes. Never fabricate content
to fill a conflict. When in doubt, ask.

Also handle these cross-tree cases (visible via `git status`):

- **Deleted by upstream, still in user's version**: confirm with the
  user — was the removed content moved elsewhere? Do they still
  need it? Do not silently restore.
- **Renamed**: git's rename detection usually handles this
  correctly; verify by inspecting the merged file.

When everything is resolved:

    git add -A
    git commit -q -m 'merged'

### Step 8 — Apply the merged result to the active skill

Overlay the merged `um-ref/` onto the active skill. Preserve the
active skill's `.git` if it has one — some users track their local
customizations there and expect to see the merge as pending changes:

    find ~/.claude/skills/um-ref -mindepth 1 -maxdepth 1 \
        ! -name '.git' -exec rm -rf {} +
    (cd "$REPO_ROOT/um-ref" && \
     find . -mindepth 1 -maxdepth 1 -print0 | \
     xargs -0 -I{} cp -a {} ~/.claude/skills/um-ref/)

Tell the user:
- Their active skill is now updated.
- The pre-merge backup remains at `/tmp/${USER}-um-ref.pre-merge`
  if they need to roll back. It will be overwritten on the next
  Update or Contribute run and cleared on reboot.
- Their new BASE for future Update or Contribute runs is the tip of
  the `um-ref-merge-work` branch. If they want a stable reference,
  they can tag it or note the SHA.

**For Update, stop here.**

## Contribute workflow

The upstream `um-ref` team uses **trunk-based development**:
contributions land directly on `main`. There is no PR review flow and
no `gh` command. Every push is trusted to have been tested locally.

This skill assumes the contributor has write access on the upstream
repo. Contributors without write access should open a GitHub issue
describing the change and let a maintainer land it.

Steps 1–7 are identical to Update. The `um-ref-merge-work` branch in
the repo clone now holds the merged result and is exactly what will
land on `main`.

Do not run Step 8's active-skill overlay yet — apply it only after
the user has approved the contribution content, so what lands on
their live skill is the same content they've reviewed for the push.

### Step 8-C — Filter the contribution and show the user

Some files in the merged result may be **personal-workflow**
artifacts rather than UM knowledge — they belong in the user's
active skill but not in the upstream repo. Check for:

- `.gitignore` with personal scratchpad patterns (e.g. `x`, `x.*`,
  `*.x`) that reflect the user's own file-naming habits.
- Files named `notes.md`, `todo.md`, `scratch.md`, or similar.
- Any new file whose content is clearly project-specific to the
  user's own work rather than general UM reference material.

For each suspect file, ask the user: include in the contribution or
exclude? Exclude by removing from the working tree and re-running
`git add -A`.

Then show the user the summary:

    git status
    git diff --stat origin/main

**Stop before committing.** The user reviews and either instructs
you to commit and push, or asks for adjustments.

### Step 9-C — Apply the merged result to the active skill

Same as Update Step 8. Do this after the user has approved the
contribution content.

### Step 10-C — Push to main (only on explicit approval)

Squash the merged branch into `main` and push:

    git checkout main
    git pull                                     # fast-forward local main to origin/main
    git merge --squash um-ref-merge-work
    git commit -m 'contribute local um-ref improvements'
    git push
    git branch -D um-ref-merge-work              # force-delete: squash, not a real merge

The user's clone is now on `main` at the newly pushed commit. That
commit is their new BASE for any future Update or Contribute — the
clone continues to serve as the record of where their install is.

If `git pull` reports someone else advanced `main` while you were
working, re-do the merge in the working branch against the new
`origin/main` before the final push, and re-test.

Do **not** `git push` without explicit permission. This is a public
repo, and the push is where the change becomes visible to others.

## When the full merge is overkill

If the user's improvements are trivial and independent of anything
the maintainer changed — e.g., a single new deep-dive file with no
`SKILL.md` edits — skip the branch-and-merge machinery for
Contribute. Copy the new file(s) into `um-ref/`, verify upstream
didn't add a colliding path, and stage. The merge machinery is
overhead when there's nothing to reconcile.

Conversely, if the user rewrote the skill from scratch or made
architecturally sweeping changes, a mechanical merge won't capture
the intent — suggest they open a GitHub issue describing the
changes first so the team can weigh in on approach before a big
push lands.

## Guardrails

- **Backup first.** Before overwriting the user's active skill, run
  `rm -rf "/tmp/${USER}-um-ref.pre-merge"` and then
  `cp -a ~/.claude/skills/um-ref "/tmp/${USER}-um-ref.pre-merge"`.
  The `/tmp` location keeps the backup out of the active skill set.
- **BASE checkout is the user's responsibility.** Don't guess. If
  they can't identify their install, stop.
- **Preserve executable bits.** Use `cp -a` (archive mode), never
  `cp -r` — the skill's `.py` and `.sh` files are executable.
- **Preserve the active skill's `.git` on overlay.** Some users
  track local customizations there.
- **Generated files are always upstream's.** Never merge user
  changes into them.
- **No pushes, force-pushes, or branch deletions** without explicit
  user permission.
- **No `git clean -fdx` or `git reset --hard` in the upstream repo
  clone** — users may have local work you don't know about.

## Worked example

User installed from commit `f97258b`. Their clone is still at
`f97258b` (they were told to keep it intact). Upstream has since
released 1.0 and added new content. The user has added a new deep
dive `wc_details.md` on wildcard receivers and inserted a
router-table entry for it in `SKILL.md`. Meanwhile, between their
install and 1.0, the maintainer added
`configuration_best_practices.md` and rewrote §3 of `SKILL.md`.

Verify BASE:

    diff -rq ~/.claude/skills/um-ref/ um-ref/ | head
    # Shows only SKILL.md diff and wc_details.md as new — plausible.

Backup, branch, overlay, commit:

    rm -rf "/tmp/${USER}-um-ref.pre-merge"
    cp -a ~/.claude/skills/um-ref "/tmp/${USER}-um-ref.pre-merge"
    git checkout -b um-ref-merge-work
    find um-ref -mindepth 1 -maxdepth 1 -exec rm -rf {} +
    (cd ~/.claude/skills/um-ref && \
     find . -mindepth 1 -maxdepth 1 ! -name '.git' -print0 | \
     xargs -0 -I{} cp -a {} "$REPO_ROOT/um-ref/")
    git add -A && git commit -m 'user improvements'

Merge:

    git merge origin/main

`git status` shows `SKILL.md` in conflict (both sides edited near
§3). `wc_details.md` and `configuration_best_practices.md` auto-add.

Resolve `SKILL.md` conflict semantically: the user's router-table
row still applies to the rewritten §3; keep it and adapt the wording
to match the new section. Ask the user only if the intent is
unclear. Commit.

Contribute path: show the user the diff. They approve. Apply merged
result to active skill. Commit on the branch. User decides whether
to squash onto `main` and `git push`.
