---
name: zc
description: Reach the user from the terminal using the `zc` CLI (zcoms), over Telegram (`zc tg`) and WhatsApp (`zc wa`). Send notifications, ask a question and block until they reply (Telegram), hold a scriptable back-and-forth, post updates to a project's group, create and read group chats, edit, pin and delete messages, and send or download files. Use this whenever you need to tell the user a task finished or is blocked, get a decision you cannot proceed without while they are away from the keyboard, or have an ongoing conversation with them. Trigger it even when the user just says things like "let me know on Telegram", "text me when done", "message me on WhatsApp", "ask me on my phone", or "ping me if you get stuck".
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

- **Found:** the folder is bound to a project group. Post there, running from inside that folder.
  Keep the main chat for milestones (done, blocked, a decision needed); put screenshots, files
  and step-by-step progress in a thread named after the piece of work (Telegram):
  ```sh
  zc project say "build is green, PR is up"                          # milestone: main chat
  zc project say --thread "login form" "step 2: validation wired"    # progress: thread
  zc project say --thread "login form" --file /tmp/login.png "new login form"
  ```
  **Finished UI work:** take a screenshot of the running app and post it to the work's thread,
  e.g. `npx playwright screenshot --full-page http://localhost:5173/login /tmp/login.png`, then
  `zc project say --thread "<work>" --file /tmp/login.png "<what changed>"`. Only screenshot pages
  that need no login unless the repo says how to sign in.
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
zc tg send "Acme build" --thread "login bug" "on it"            # post in a group thread
zc tg send-file "Acme build" ./shot.png --thread 12 "screenshot" # thread by id or name
```

Files are capped at 20 MB; images go as photos, other files as documents. `--thread` takes a
topic id or name (see `zc tg group topics`) and needs the daemon.

**Edit, pin and delete** (Telegram). Name a message by the `message_id` that `send` printed (or from
`chat --read N --json`), or `last` for the newest message you sent in that chat:

```sh
zc tg edit @user last "build finished (all 212 tests green)"   # fix what you just sent
zc tg edit "Acme build" 4821 "deploy moved to 6pm"             # by message id
zc tg pin "Acme build" last [--silent]                         # --silent: no notification
zc tg pin @user 4821 --for-me                                  # private chat, only your side
zc tg unpin "Acme build" last | zc tg unpin "Acme build" --all
zc tg delete "Acme build" last                                 # delete for everyone
zc tg delete "Acme build" 4821 4822                            # several by id
```

`zc tg send <chat> --keep "<text>"` sends a message that never disappears; `--pin [--silent]`
also pins it (implies `--keep`, takes `--thread`, needs the daemon).

Edit works on text messages you sent, within Telegram's edit window. Pinning in a group needs the
pin right there; deleting someone else's message needs the delete right. Deletes cannot be undone:
only delete what the user asked you to. Prefer editing a status message you posted over sending a new one each step.

`chat` flags: `-w/--wait` (default on), `-t/--timeout <dur>` (`0` = no limit), `-r/--read <N>`,
`--json` (one line per message: `message_id`, `sender`, `kind`, `text`, `file`, `disappear_in`), `--download`.
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
zc tg group topic "Acme build" "login bug"     # start a thread (forum topic); prints its id
zc tg group topics "Acme build"                # list threads (id, name)
zc tg group topic-rename "Acme build" "login bug" "auth" # rename a thread (owner only)
zc tg group create "Acme build" Ibrahim --topics  # create with threads on from the start
zc tg group create "Acme build" Ibrahim --topics --disappear 7d  # disappearing messages on
zc tg group disappear "Acme build" [1d|7d|2w|off] # whole group: show or set the timer
zc tg group disappear "Acme build" 12h --thread "standup"  # one thread only
```

Threads are Telegram forum topics. `group topic` turns topics on by itself only when this account
created the group; otherwise an admin has to enable them in the group settings. Basic groups are
never upgraded. WhatsApp groups have no threads: `zc wa group topic` says so, and a `--thread`
send on WhatsApp goes to the group itself.

**Disappearing messages** (Telegram): Telegram's own timer covers the whole group (whole days,
every thread). For one thread, use `--thread`: Telegram has no per-thread timer, so the comms
daemon deletes that thread's messages once they are older than the timer (checked every minute;
minutes and hours work too). Either way only messages sent after the timer is set are affected,
and the bare `zc tg group disappear "<group>"` lists every timer. Pinning does not save a message from it, so post
anything that must stay with `zc tg send "<group>" --pin "<text>"` (sent outside the timer).
Only change a group's timer when the user asks.

Telegram groups are created as supergroups. Set the owner once with `zc config owner <contact>`
(no argument shows it); it must be a contact in `zc contacts`.

**Agent in groups:** every group is off until turned on. `zc group on "<group>"` lets anyone there
call the agent (an @mention or a reply to it); it answers briefly, web search only, in the thread
it was called from. Callers can ask it to "start a thread for X": it opens one thread in that group
and answers there. In project groups it can also send screenshots of the running app or repo files
back to the caller (never secrets, at most 5 files of 20 MB).
The owner and write-role members can also ask it to pin its answer, pin or unpin the message
they replied to, correct or delete its last message; only the owner can have it delete someone
else's message or turn disappearing messages on or off (asked inside a thread it applies to that
thread only).
`zc group off "<group>"` stops it. Add `--wa` for a WhatsApp group.

## Projects (`zc project`)

Binds a repo folder to a group; the agent answers calls there from the code (read only). Run from
inside the folder (any subdirectory works):

```sh
zc project init [--group <name>] [--follow on|off] [--wa]  # prompts for anything missing
zc project say "<text>"              # post an update to the project's group (milestones)
zc project say [--thread <name|id>] [--file <path>]... "<text>"
zc project say --pin [--thread <name|id>] "<text>"  # pinned status that never disappears
                                     # files (photos as photos), caption on the first; --thread
                                     # reuses that thread or creates it
zc project status | sync | remove [--purge]
zc project members | alias add|rm <member> <alias>
zc project tasks [done|drop <id> | assign <id> <member>]
zc project follow on|off             # store messages + morning standup
zc project standup now
```

If the group cannot take a thread (WhatsApp, or topics off in a group this account did not create),
text goes to the main chat and `--file` sends go to the owner's private chat with a note. On
WhatsApp, post milestones only and send screenshots only when asked.

`init` writes the `.zcoms` file, and `init`/`sync` write the routing rules into the repo's
AGENTS.md/CLAUDE.md. Binding and turning groups on are owner actions: only run
`project init`, `group on` or `config owner` when the user asks.

## Rules and gotchas

- **One Telegram session at a time** when the daemon is down: a blocked `zc tg ask`/`chat` holds
  the TDLib lock, so do not start a second `zc tg` command meanwhile. With the daemon up, calls
  go through it.
- `zc tg ask` never times out; use it only when you truly need the answer.
- Batch context into one good question rather than many small ones.
- No em or en dashes in messages to the owner; use commas, colons or periods.
