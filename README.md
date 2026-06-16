# zcoms-skill

A [Claude Agent Skill](https://github.com/anthropics/skills) that lets Claude (Claude Code, or any
agent that supports the skills standard) **reach you on Telegram and WhatsApp** — send
notifications, ask a question and wait for your reply (Telegram), hold a back-and-forth
conversation, message a WhatsApp contact and read unread WhatsApp, and send/receive files —
using the [**zcoms**](https://github.com/Zouriel/zcoms) command-line tool.

> The CLI itself lives in a separate repo: **[Zouriel/zcoms](https://github.com/Zouriel/zcoms)**.
> This repo is just the skill that teaches an agent how to drive it.

## What it does

Once installed, the agent knows to message you on Telegram when it finishes a long task, hits a
blocker, or needs a decision it can't make on its own — and you can reply right from your phone:

```sh
# Telegram (primary — the only channel with a blocking ask/chat)
zc tg send @you "build passed ✅"                      # fire-and-forget notification
answer=$(zc tg ask @you "deploy to prod now or wait?")  # ask and block until you reply
reply=$(zc tg chat @you "ready?" --timeout 5m --json)   # scriptable back-and-forth
zc tg send-file @you ./report.pdf "the report"          # send a file
zc tg download @you                                     # pick & download media you sent

# WhatsApp (optional sidecar; recipient is a phone number / JID, not @username)
zc wa status                                            # is the sidecar paired & connected?
zc wa send +9607988692 "build passed ✅"                # fire-and-forget (no blocking ask)
zc wa unread --json                                     # read unread 1:1s (poll to await a reply)
```

## Prerequisites

1. Install and log in to **zcoms** — see <https://github.com/Zouriel/zcoms> (download a release,
   run `zc tg login`, verify with `zc tg auth`).
2. Know your own Telegram `@username` (the recipient the agent will message).
3. **WhatsApp is optional** — to use `zc wa`, enable it in `agent-settings.json` and run the
   Baileys sidecar, then pair with `zc wa login` (check with `zc wa status`). ⚠️ Baileys is an
   unofficial client and carries a ban risk; prefer Telegram for interactive use.

## Install

This is a standard Agent Skill — a folder with a `SKILL.md`. Install it wherever your agent loads
skills from.

**Claude Code (personal skills):**
```sh
git clone https://github.com/Zouriel/zcoms-skill
cp -r zcoms-skill/skills/zc ~/.claude/skills/zc
```
(or symlink it). For a single project, copy into that project's `.claude/skills/` instead.

**As a slash command (optional):** if you'd rather invoke it explicitly with `/zc`, copy the body
into `~/.claude/commands/zc.md` instead of `~/.claude/skills/`.

Then tell the agent your Telegram `@username` (and, if your credentials live in a project `.env`,
which folder to run `zc` from). The agent will use the skill automatically when it needs to reach
you, or you can nudge it ("let me know on Telegram when this is done").

## Layout

```
zcoms-skill/
├── README.md
├── LICENSE
└── skills/
    └── zc/
        └── SKILL.md        # the skill definition (frontmatter + instructions)
```

## License

[MIT](LICENSE) — same as zcoms.
