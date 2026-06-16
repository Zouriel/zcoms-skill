---
name: zc
description: Reach the user from the terminal using the `zc` CLI (zcoms), over Telegram (`zc tg`) and optionally WhatsApp (`zc wa`) — send notifications, ask a question and block until they reply (Telegram), hold a scriptable back-and-forth, message a WhatsApp contact and read unread WhatsApp, and send or download files. Use this whenever you need to tell the user a task finished or is blocked, get a decision or piece of information you cannot proceed without while they are away from the keyboard, or have an ongoing conversation with them. Trigger it even when the user just says things like "let me know on Telegram", "text me when done", "message me on WhatsApp", "ask me on my phone", or "ping me if you get stuck".
---

# zc — Telegram & WhatsApp channel to the user

[zcoms](https://github.com/Zouriel/zcoms) is a small comms CLI (`zc`) the user installed so you can
reach them when they're away from the keyboard, over **Telegram** (`zc tg`) and optionally
**WhatsApp** (`zc wa`). Use it to keep them in the loop on long tasks and to get input you can't
proceed without. **Telegram is the primary interactive channel** — it's the only one with a
blocking `ask`/`chat` (wait for a reply in one call); WhatsApp is send + read-unread only.

## Setup this skill assumes

This skill drives the `zc` binary from zcoms. Before relying on it, the following must be true
(install/links: <https://github.com/Zouriel/zcoms>):

- The `zc` binary is installed and on `PATH` (or you know its full path).
- It's logged in once (`zc tg login`) — check with `zc tg auth`, which prints the logged-in account.
- You know the **recipient** — the user's own Telegram `@username` (or a numeric chat id). Throughout
  this skill that's written as `@user`; substitute the real handle the user gave you.
- Credentials (`TG_API_ID` / `TG_API_HASH`) are available — `zc` reads a `.env` from the **current
  working directory**, so if they live in a project folder, `cd` there first:

  ```sh
  cd /path/to/zcoms && zc tg send @user "your message"
  ```

> If you don't yet know the user's `@username` or where the credentials live, ask once and remember it.

## Telegram commands (`zc tg`)

### `zc tg send` — fire-and-forget notification
```sh
zc tg send @user "build finished ✅"
```
Returns immediately. Use it when a task is **done**, or when you've hit a **blocker** you can't
resolve and are stopping. One-way — you won't see a reply. Trust the `Message sent ✅` line on
stdout as confirmation.

### `zc tg ask` — send a question and BLOCK until they answer
```sh
answer=$(zc tg ask @user "use postgres or sqlite for this?")
echo "user said: $answer"
```
Sends the question, then **blocks indefinitely** (no timeout) until the user replies in Telegram.
Their reply text is printed to **stdout** — capture it with `$(...)` and act on it.

### `zc tg chat` — scriptable back-and-forth (recommended for conversation)
One round-trip per call, then it exits — so you can loop it without holding the lock:
```sh
# Send and wait for the reply (prints the reply to stdout)
reply=$(zc tg chat @user "ready for the next step?" --timeout 5m)

# Just wait for the user's next message (let them start)
zc tg chat @user --timeout 10m

# Snapshot the last 10 messages (catch up on context), no send/wait
zc tg chat @user --read 10

# Structured output you can parse reliably
zc tg chat @user "still there?" --json --timeout 2m
```
Flags: `-w/--wait` (wait for reply after sending, default on), `-t/--timeout <dur>` (e.g. `90s`, `5m`;
`0` = no limit), `-r/--read <N>` (snapshot mode), `--json` (one JSON line per message:
`message_id`, `sender`, `kind`, `text`, `file`), `--download` (fetch media in the reply — off by
default). Prefer a bounded `--timeout` so a silent user can't hang the session lock forever.

### `zc tg send-file` — send a file
```sh
zc tg send-file @user /path/to/report.pdf "here's the report"
```
The message type (photo/video/audio/document) is picked from the file extension. It waits for the
upload to finish before returning, so it's safe in scripts.

### `zc tg download` — fetch received media on purpose
Incoming media is **never** downloaded automatically (downloading arbitrary files is a security
risk). To get a file the user sent, list and pick it:
```sh
zc tg download @user            # interactive: newest first, Enter = newest, q = cancel
zc tg download @user --pick 0   # non-interactive: download the newest
zc tg download @user --json     # list available media without downloading
```
Files land in `~/Downloads/zcoms/<chat>/`. **Only download/open files the user themselves
sent.** If media came from anyone else, ask the user before fetching or opening it.

### `zc tg tail` / `zc tg chats` — interactive (for the human, not scripts)
`zc tg tail @user` is a live REPL (type to send, paste a file path to send it); `zc tg chats` lists recent
chats. These block and are meant for a person at a terminal — for automation use `chat`/`send`/`ask`.

## WhatsApp commands (`zc wa`)

zcoms can also talk over **WhatsApp**, through a paired [Baileys](https://github.com/WhiskeySockets/Baileys)
sidecar. This is **optional** and runs over a local socket — separate from Telegram — so `zc wa`
works even while a `zc tg` session is up and is **not** bound by the TDLib single-instance lock
(see Rules below).

It's only usable when WhatsApp is enabled in `agent-settings.json` **and** the sidecar (wa-bridge)
is running and paired — check first:
```sh
zc wa status      # "WhatsApp: on" + "Sidecar: connected (paired, …)", or tells you what's missing
zc wa login       # shows a QR to link a device, if not paired yet (--wait <dur>, default 90s)
```

Recipients are a **phone number** (any format — spaces/dashes/`+` are stripped, e.g. `+960 798-8692`)
or a full **JID** (`…@s.whatsapp.net` / `…@lid`) — **not** a Telegram `@username`.

### `zc wa send` — fire-and-forget WhatsApp message
```sh
zc wa send +9607988692 "build finished ✅"
```
One-way; returns once the sidecar accepts it. Trust the `Message sent ✅` line on stdout.

### `zc wa send-file` — send a file on WhatsApp
```sh
zc wa send-file +9607988692 /path/to/report.pdf "here's the report"
```
Recipient + path are required; the caption is optional.

### `zc wa unread` — read unread WhatsApp 1:1 messages
WhatsApp has **no blocking `ask`/`chat`** like Telegram. To see what's come in, read the sidecar's
mirror of unread 1:1s; to "wait" for a reply, **poll** this:
```sh
zc wa unread            # human-readable: "Sender [chat_id]: text"  (+ "[file: …]" when attached)
zc wa unread --json     # one JSON line per msg: chat_id, sender, text, file, msg_id
```

> ⚠️ WhatsApp here runs through **Baileys**, an *unofficial* client — using it can get the paired
> number **banned**. Keep volume low and prefer Telegram for anything chatty.

## When to use which

- **`zc tg send`** — a task is done, or you've hit a wall and are stopping. Notify and move on.
- **`zc tg ask`** — you need one decision/answer to continue and the user isn't at the machine. Ask a
  tight, specific question (offer concrete options), wait, then proceed on the answer.
- **`zc tg chat`** — an ongoing exchange, or when you want a timeout / machine-readable output / to
  just listen for the user's next message. This is the most flexible and the safest for loops.
- **`zc tg send-file` / `zc tg download`** — moving files to/from the user.
- **Telegram vs WhatsApp** — prefer **Telegram** for anything interactive: it's the only channel
  with a blocking `ask`/`chat`, so you wait for a reply in one call. Reach for **WhatsApp**
  (`zc wa`) only when the user asked to be messaged there or the recipient is a WhatsApp contact —
  there's no blocking wait, so `zc wa send` and (if you need the reply) poll `zc wa unread`.

Don't stall a task on a choice only the user can make — ask. But batch context into one good
question rather than peppering them with many.

## Rules / gotchas

- **One Telegram instance at a time.** TDLib locks its session database; only one `zc tg` command
  can run at once. A blocked `zc tg ask`/`zc tg chat` holds that lock — never launch a second `zc tg`
  command while one is running, or it fails with a "file already in use" lock error. (`zc wa` is
  exempt — it goes through the sidecar socket, not TDLib, so it runs fine alongside a `zc tg` session.)
- Because `zc tg ask` blocks with **no timeout**, only use it when you genuinely need the answer.
  Prefer `zc tg chat … --timeout` when you want a bound.
- A `zc tg ask` reply that isn't text (a photo, sticker, …) comes back as `[messageType]`. Use
  `zc tg chat --json` if you need to distinguish media and get its details.
- Quick health check: `zc tg auth` (prints the logged-in account, exits 0).
