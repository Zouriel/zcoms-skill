---
name: zc
description: Reach the user from the terminal using the `zc` CLI (zcoms), over Telegram (`zc tg`) and WhatsApp (`zc wa`). Send notifications, ask a question and block until they reply (Telegram), hold a scriptable back-and-forth, post updates to a project's group, create and read group chats, and send or download files. Use this whenever you need to tell the user a task finished or is blocked, get a decision you cannot proceed without while they are away from the keyboard, or have an ongoing conversation with them. Trigger it even when the user just says things like "let me know on Telegram", "text me when done", "message me on WhatsApp", "ask me on my phone", or "ping me if you get stuck".
---

# zc: Telegram and WhatsApp channel to the user

[zcoms](https://github.com/Zouriel/zcoms) is a comms CLI (`zc`) the user installed so you can reach
them when they are away from the keyboard. **Telegram** (`zc tg`) is the interactive channel (the
only one with a blocking `ask`/`chat`); **WhatsApp** (`zc wa`) is send, read and groups.

## Routing rule: where updates to the owner go (check this first)

Before sending the owner an update (done, blocked, progress), look for a **`.zcoms`** file in the
current directory or any parent:

```sh
d=$PWD; while [ "$d" != / ] && [ ! -f "$d/.zcoms" ]; do d=$(dirname "$d"); done; [ -f "$d/.zcoms" ] && echo bound
```

- **Found:** the folder is bound to a project group. Post there, running from inside that folder:
  ```sh
  zc project say "build is green, PR is up"
  ```
- **Not found:** message the owner directly:
  ```sh
  zc tg send @ZourielCorbet "build is green, PR is up"
  ```

Questions you need answered (`ask`/`chat`) still go to the owner directly on Telegram.

## Setup this skill assumes

- `zc` is on `PATH`. Most commands route through the **comms daemon** (`zcoms-daemon.service`),
  which owns the Telegram session and runs WhatsApp in process. `zc wa …`, `zc tg group …`,
  `zc group` and `zc project` need the daemon (and the agent, for `group`/`project`) running.
- Telegram credentials (`TG_API_ID` / `TG_API_HASH`) are only needed when the daemon is down and
  `zc tg` opens its own session. `zc` loads a `.env` from the **current working directory**
  (overriding the environment), else uses the environment, else credentials baked into the
  build. If you hit "missing TG_API_ID or TG_API_HASH", `cd` to the folder holding the `.env` first.
- Recipients for `zc tg`: `@username`, a contact name (`zc contacts list`), a group name, or a
  chat id. The owner is `@ZourielCorbet`.

## Telegram (`zc tg`)

```sh
zc tg send @user "build finished"                 # fire and forget; trust "Message sent ✅"
answer=$(zc tg ask @user "postgres or sqlite?")    # blocks with NO timeout; reply on stdout
reply=$(zc tg chat @user "next step?" --timeout 5m)  # one round trip, then exits
zc tg chat @user --timeout 10m                    # just wait for their next message
zc tg chat @user --read 10                        # snapshot the last 10 messages
zc tg send-file @user ./report.pdf "the report"   # type picked from extension; waits for upload
```

`chat` flags: `-w/--wait` (default on), `-t/--timeout <dur>` (`0` = no limit), `-r/--read <N>`,
`--json` (one line per message: `message_id`, `sender`, `kind`, `text`, `file`), `--download`.
Prefer `chat --timeout` over `ask` when you want a bound. A non-text `ask` reply comes back as
`[messageType]`; use `chat --json` to see details.

**Daemon-only-off commands:** `zc tg auth`, `login`, `logout`, `chats`, `tail` and `download` need
their own TDLib session, so they refuse while the daemon runs (stop it with
`systemctl --user stop zcoms-daemon`; never do that from inside a bridge session).
`zc tg download @user [--pick 0] [--json] [-n 30]` saves to `~/Downloads/zcoms/<chat>/`. Only
download files the user themselves sent; ask before fetching anyone else's media.

## WhatsApp (`zc wa`)

Runs in process (whatsmeow) inside the daemon. Recipients: a phone number (any format), a JID, a
contact name, or a group name; not a Telegram `@username`.

```sh
zc wa status                                   # connection state; pair via console Connectors page
zc wa send +9607988692 "build finished"
zc wa send-file +9607988692 ./report.pdf "the report"
zc wa read +9607988692 [count]                 # recent history (--count, default 10)
zc wa unread [--json]                          # unread 1:1s: chat_id, sender, text, file, msg_id
```

No blocking wait on WhatsApp: to get a reply, poll `zc wa unread`. It is an unofficial client, so
keep volume low and prefer Telegram for anything chatty.

## Groups (`zc tg group`, `zc wa group`)

The same subcommands exist under both. A group is named by its title (as `group list` shows it),
an alias, or its native id.

```sh
zc tg group create "Acme build" Ibrahim Aisha  # members are contacts; the OWNER IS ALWAYS ADDED
zc tg group add "Acme build" Hassan            # add members
zc tg group list                               # groups this account is in ([on], [follow] flags)
zc tg group members "Acme build"
zc tg group view "Acme build" --limit 10       # latest N (default 50)
zc tg group view "Acme build" --around 1234    # N messages either side of a message id
zc tg group view "Acme build" --follow         # stay attached: print new, send typed lines
zc tg send "Acme build" "deploy is green"      # plain send works by group name too
```

Telegram groups are created as supergroups. Set the owner once with `zc config owner <contact>`
(no argument shows it); it must be a contact in `zc contacts`.

**Agent in groups:** every group is off until turned on. `zc group on "<group>"` lets anyone there
call the agent (an @mention or a reply to it); it answers briefly, web search only.
`zc group off "<group>"` stops it. Add `--wa` for a WhatsApp group.

## Projects (`zc project`)

Binds a repo folder to a group; the agent answers calls there from the code (read only). Run from
inside the folder (any subdirectory works):

```sh
zc project init [--group <name>] [--follow on|off] [--wa]  # prompts for anything missing
zc project say "<text>"              # post an update to the project's group
zc project status | sync | remove [--purge]
zc project members | alias add|rm <member> <alias>
zc project tasks [done|drop <id> | assign <id> <member>]
zc project follow on|off             # store messages + morning standup
zc project standup now
```

`init` writes the `.zcoms` file. Binding and turning groups on are owner actions: only run
`project init`, `group on` or `config owner` when the user asks.

## Rules and gotchas

- **One Telegram session at a time** when the daemon is down: a blocked `zc tg ask`/`chat` holds
  the TDLib lock, so do not start a second `zc tg` command meanwhile. With the daemon up, calls
  go through it.
- `zc tg ask` never times out; use it only when you truly need the answer.
- Batch context into one good question rather than many small ones.
- No em or en dashes in messages to the owner; use commas, colons or periods.
