# tgcli-skill

A [Claude Agent Skill](https://github.com/anthropics/skills) that lets Claude (Claude Code, or any
agent that supports the skills standard) **reach you on Telegram** — send notifications, ask a
question and wait for your reply, hold a back-and-forth conversation, and send/receive files —
using the [**TgCli**](https://github.com/Zouriel/TgCli) command-line tool.

> The CLI itself lives in a separate repo: **[Zouriel/TgCli](https://github.com/Zouriel/TgCli)**.
> This repo is just the skill that teaches an agent how to drive it.

## What it does

Once installed, the agent knows to message you on Telegram when it finishes a long task, hits a
blocker, or needs a decision it can't make on its own — and you can reply right from your phone:

```sh
tg send @you "build passed ✅"                      # fire-and-forget notification
answer=$(tg ask @you "deploy to prod now or wait?")  # ask and block until you reply
reply=$(tg chat @you "ready?" --timeout 5m --json)   # scriptable back-and-forth
tg send-file @you ./report.pdf "the report"          # send a file
tg download @you                                     # pick & download media you sent
```

## Prerequisites

1. Install and log in to **TgCli** — see <https://github.com/Zouriel/TgCli> (download a release,
   run `tg login`, verify with `tg auth`).
2. Know your own Telegram `@username` (the recipient the agent will message).

## Install

This is a standard Agent Skill — a folder with a `SKILL.md`. Install it wherever your agent loads
skills from.

**Claude Code (personal skills):**
```sh
git clone https://github.com/Zouriel/tgcli-skill
cp -r tgcli-skill/skills/tg ~/.claude/skills/tg
```
(or symlink it). For a single project, copy into that project's `.claude/skills/` instead.

**As a slash command (optional):** if you'd rather invoke it explicitly with `/tg`, copy the body
into `~/.claude/commands/tg.md` instead of `~/.claude/skills/`.

Then tell the agent your Telegram `@username` (and, if your credentials live in a project `.env`,
which folder to run `tg` from). The agent will use the skill automatically when it needs to reach
you, or you can nudge it ("let me know on Telegram when this is done").

## Layout

```
tgcli-skill/
├── README.md
├── LICENSE
└── skills/
    └── tg/
        └── SKILL.md        # the skill definition (frontmatter + instructions)
```

## License

[MIT](LICENSE) — same as TgCli.
