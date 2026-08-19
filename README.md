# /tldr

Cut through Claude's verbal Diarrhea with TLDR - A Claude skill that condenses recent session work into a fixed scaffold —
**TL;DR / Did / Found / Next** — so you can save your scanning time for what you actual want!

## Layout

- `.claude/skills/tldr/SKILL.md` — the skill. Anyone working in this repo gets `/tldr`.
- `plugins/tldr-plugin/` + `.claude-plugin/` — marketplace wrapper, for installing it elsewhere.

## Install elsewhere (Claude Code)

```
/plugin marketplace add victoria-hedlund/TLDR
/plugin install tldr-plugin@tldr
```

The repo is private, so this only works for people with access to it.

In the **Claude app**, marketplaces don't apply — install the skill through the app's
skills settings instead.

## Editing the skill

`SKILL.md` exists twice: the working copy at `.claude/skills/tldr/`, and the plugin copy
under `plugins/`. Plugins are copied on install and can't reference files outside their own
directory, so a symlink isn't an option. After editing, sync them and bump the version:

```bash
cp .claude/skills/tldr/SKILL.md plugins/tldr-plugin/skills/tldr/SKILL.md
```

then raise `version` in both `.claude-plugin/marketplace.json` and
`plugins/tldr-plugin/.claude-plugin/plugin.json` — users only receive updates when it changes.
