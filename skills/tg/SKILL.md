---
name: tg
description: Reach the user on Telegram from the terminal using the `tg` CLI (TgCli) — send notifications, ask a question and block until they reply, hold a scriptable back-and-forth conversation, and send or download files. Use this whenever you need to tell the user a task finished or is blocked, get a decision or piece of information you cannot proceed without while they are away from the keyboard, or have an ongoing conversation with them. Trigger it even when the user just says things like "let me know on Telegram", "text me when done", "ask me on my phone", or "ping me if you get stuck".
---

# tg — Telegram channel to the user

[TgCli](https://github.com/Zouriel/TgCli) is a small Telegram CLI (`tg`) the user installed so you can
reach them when they're away from the keyboard. Use it to keep them in the loop on long
tasks and to get input you can't proceed without.

## Setup this skill assumes

This skill drives the `tg` binary from TgCli. Before relying on it, the following must be true
(install/links: <https://github.com/Zouriel/TgCli>):

- The `tg` binary is installed and on `PATH` (or you know its full path).
- It's logged in once (`tg login`) — check with `tg auth`, which prints the logged-in account.
- You know the **recipient** — the user's own Telegram `@username` (or a numeric chat id). Throughout
  this skill that's written as `@user`; substitute the real handle the user gave you.
- Credentials (`TG_API_ID` / `TG_API_HASH`) are available — `tg` reads a `.env` from the **current
  working directory**, so if they live in a project folder, `cd` there first:

  ```sh
  cd /path/to/TgCli && tg send @user "your message"
  ```

> If you don't yet know the user's `@username` or where the credentials live, ask once and remember it.

## Commands

### `tg send` — fire-and-forget notification
```sh
tg send @user "build finished ✅"
```
Returns immediately. Use it when a task is **done**, or when you've hit a **blocker** you can't
resolve and are stopping. One-way — you won't see a reply. Trust the `Message sent ✅` line on
stdout as confirmation.

### `tg ask` — send a question and BLOCK until they answer
```sh
answer=$(tg ask @user "use postgres or sqlite for this?")
echo "user said: $answer"
```
Sends the question, then **blocks indefinitely** (no timeout) until the user replies in Telegram.
Their reply text is printed to **stdout** — capture it with `$(...)` and act on it.

### `tg chat` — scriptable back-and-forth (recommended for conversation)
One round-trip per call, then it exits — so you can loop it without holding the lock:
```sh
# Send and wait for the reply (prints the reply to stdout)
reply=$(tg chat @user "ready for the next step?" --timeout 5m)

# Just wait for the user's next message (let them start)
tg chat @user --timeout 10m

# Snapshot the last 10 messages (catch up on context), no send/wait
tg chat @user --read 10

# Structured output you can parse reliably
tg chat @user "still there?" --json --timeout 2m
```
Flags: `-w/--wait` (wait for reply after sending, default on), `-t/--timeout <dur>` (e.g. `90s`, `5m`;
`0` = no limit), `-r/--read <N>` (snapshot mode), `--json` (one JSON line per message:
`message_id`, `sender`, `kind`, `text`, `file`), `--download` (fetch media in the reply — off by
default). Prefer a bounded `--timeout` so a silent user can't hang the session lock forever.

### `tg send-file` — send a file
```sh
tg send-file @user /path/to/report.pdf "here's the report"
```
The message type (photo/video/audio/document) is picked from the file extension. It waits for the
upload to finish before returning, so it's safe in scripts.

### `tg download` — fetch received media on purpose
Incoming media is **never** downloaded automatically (downloading arbitrary files is a security
risk). To get a file the user sent, list and pick it:
```sh
tg download @user            # interactive: newest first, Enter = newest, q = cancel
tg download @user --pick 0   # non-interactive: download the newest
tg download @user --json     # list available media without downloading
```
Files land in `~/Downloads/telegramcli/<chat>/`. **Only download/open files the user themselves
sent.** If media came from anyone else, ask the user before fetching or opening it.

### `tg tail` / `tg chats` — interactive (for the human, not scripts)
`tg tail @user` is a live REPL (type to send, paste a file path to send it); `tg chats` lists recent
chats. These block and are meant for a person at a terminal — for automation use `chat`/`send`/`ask`.

## When to use which

- **`tg send`** — a task is done, or you've hit a wall and are stopping. Notify and move on.
- **`tg ask`** — you need one decision/answer to continue and the user isn't at the machine. Ask a
  tight, specific question (offer concrete options), wait, then proceed on the answer.
- **`tg chat`** — an ongoing exchange, or when you want a timeout / machine-readable output / to
  just listen for the user's next message. This is the most flexible and the safest for loops.
- **`tg send-file` / `tg download`** — moving files to/from the user.

Don't stall a task on a choice only the user can make — ask. But batch context into one good
question rather than peppering them with many.

## Rules / gotchas

- **One instance at a time.** TDLib locks its session database; only one `tg` command can run at
  once. A blocked `tg ask`/`tg chat` holds that lock — never launch a second `tg` command while one
  is running, or it fails with a "file already in use" lock error.
- Because `tg ask` blocks with **no timeout**, only use it when you genuinely need the answer.
  Prefer `tg chat … --timeout` when you want a bound.
- A `tg ask` reply that isn't text (a photo, sticker, …) comes back as `[messageType]`. Use
  `tg chat --json` if you need to distinguish media and get its details.
- Quick health check: `tg auth` (prints the logged-in account, exits 0).
