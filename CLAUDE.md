# um-ref-merge repo — Claude Code instructions

This repo hosts the `um-ref-merge` skill. That skill's workflows
(Install / Update / Contribute in `um-ref-merge/SKILL.md`) manage the
separate `um-ref` skill; they do not apply to `um-ref-merge` itself.

## Installing or updating this skill from this repo

For prompts like "install the skill in this repo", "pull the skill in
this repo and install it", or "update um-ref-merge from this repo":

    git pull
    mkdir -p ~/.claude/skills/um-ref-merge
    cp -a um-ref-merge/. ~/.claude/skills/um-ref-merge/

Then tell the user which commit is now installed
(`git rev-parse --short HEAD`).
