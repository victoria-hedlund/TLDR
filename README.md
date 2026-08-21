# /tldr

Cut through Claude's verbal Diarrhea with TLDR - A Claude skill that condenses recent session work into a fixed scaffold —
**TL;DR / Did / Found / Next** — so you can save your scanning time for what you actual want!

## What you get

Ask for a recap and you get this shape, every time:

```
**TL;DR** — the bottom line, in a sentence.

**Did**      what actually changed
**Found**    what's now known that wasn't before
**Next**     what's still open, and what to do about it
```

Bottom line at the top, next action at the bottom, always in the same place — so you can
skim it in five seconds instead of reading another three paragraphs.

## Install

Pick the row that matches you. It's the same skill either way — only the name you type differs.

| You use | Do this | Then type |
|---|---|---|
| **Claude app** (claude.ai, desktop) | Download [`SKILL.md`](.claude/skills/tldr/SKILL.md) and add it in your skills settings | just ask for a tldr |
| **Claude Code**, want it in one step | the two commands below | `/tldr-plugin:tldr` |
| **Claude Code**, want the clean name | copy one folder, below | `/tldr` |

### Claude Code — plugin (quickest)

Nothing to download. Run these two commands inside Claude Code:

```
/plugin marketplace add victoria-hedlund/TLDR
/plugin install tldr-plugin@tldr
```

Then use it with `/tldr-plugin:tldr`. Plugins namespace whatever they carry, which is where
that prefix comes from. If it doesn't show up, run `/reload-plugins`.

### Claude Code — plain `/tldr`

If you'd rather type `/tldr` with no prefix, install the skill directly instead of as a plugin:

```bash
git clone https://github.com/victoria-hedlund/TLDR /tmp/tldr && \
  cp -r /tmp/tldr/.claude/skills/tldr ~/.claude/skills/tldr && \
  rm -rf /tmp/tldr
```

Restart Claude Code and `/tldr` is there.

No git? Hit **Code → Download ZIP** on this repo, unzip it, and copy the folder
`.claude/skills/tldr` into `~/.claude/skills/`.

### Claude app

Marketplaces and slash commands are Claude Code features and don't apply here. Instead, save
[`SKILL.md`](.claude/skills/tldr/SKILL.md) through the app's skills settings. After that you
don't type a command at all — asking for a recap, a gist, or "get to the point" triggers it.

## You don't have to type anything

Worth knowing on any of the three: the skill fires on intent, not just the command. "Summarise
that", "catch me up", "where are we at" all trigger it. The slash command is just the explicit way.

## Repo layout

- `.claude/skills/tldr/SKILL.md` — the skill itself. Everything else is packaging.
- `plugins/tldr-plugin/` + `.claude-plugin/` — the marketplace wrapper.

## Editing the skill

`SKILL.md` exists twice: the working copy at `.claude/skills/tldr/`, and the plugin copy under
`plugins/`. Plugins are copied on install and can't reference files outside their own directory,
so a symlink isn't an option. After editing, sync them:

```bash
cp .claude/skills/tldr/SKILL.md plugins/tldr-plugin/skills/tldr/SKILL.md
```

then raise `version` in both `.claude-plugin/marketplace.json` and
`plugins/tldr-plugin/.claude-plugin/plugin.json` — users only receive updates when it changes.
