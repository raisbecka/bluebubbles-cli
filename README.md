# bub — BlueBubbles Server CLI

Stdlib-only Python CLI for the BlueBubbles Server REST API. No pip
dependencies — runs anywhere Python 3.8+ does, including a bare docker
container. Designed to be paired with a SKILL.md so an agent can read and
send iMessages and look up contacts.

## Install

```sh
cp bub /usr/local/bin/bub && chmod +x /usr/local/bin/bub
# or just: ./bub ...
```

## Config (first wins)

| Source | Keys |
|---|---|
| CLI flags | `--host --port --scheme --password` |
| Env vars | `BLUEBUBBLES_HOST` `BLUEBUBBLES_PORT` `BLUEBUBBLES_SCHEME` `BLUEBUBBLES_PASSWORD` |
| Config file | `$BUB_CONFIG` or `~/.config/bub/config.json` or `~/.bubrc` |

Config file format:

```json
{"host": "127.0.0.1", "port": "1234", "scheme": "http", "password": "your-bb-password"}
```

The BB password is the one shown in the BlueBubbles Server app
(Server → Settings → "Password"). In docker, pass
`BLUEBUBBLES_HOST=<mac-ip> BLUEBUBBLES_PORT=1234 BLUEBUBBLES_PASSWORD=...`
as env vars.

Optional: `BUB_TIMEOUT` (seconds, default 300) — AppleScript sends can
take a while; the server may keep working after the client gives up.

## Output & exit codes

- JSON on stdout (pretty by default, `--compact` for single-line).
- Exit 0 = ok, 1 = API error, 2 = usage/config/network error.
- Errors go to stderr as `bub: <message>`.

## Commands

```
bub ping                        health check
bub info                        server metadata (version, iMessage account)
bub stats                       entity totals
bub logs                        server logs (text blob)
bub alerts                      server alerts
bub update-check                check for server updates

bub chats list [--limit N] [--offset N]
bub chats get <guid>
bub chats messages <guid>       all messages in a chat
bub chats search <text>         search chats by display name
bub chats new <addr>... [--message TEXT]
bub chats rename <guid> <name>
bub chats add-participant <guid> <addr>
bub chats remove-participant <guid> <addr>
bub chats delete <guid>

bub messages send <chat> [text] [--attach f1 f2] [--effect ID] [--reply-to GUID] [--dry-run]
bub messages list [--limit N] [--chat <addr-or-guid>]
bub messages search <text>
bub messages get <guid>
bub messages react <chat> <message-guid> <emoji>
bub messages edit <guid> <text>        (needs private API)
bub messages unsend <guid>             (needs private API)

bub attach download <guid> [-o path]
bub attach send <chat> <file>

bub contacts list [--search TEXT] [--limit N]
bub contacts get <address>             iCloud contact card
bub contacts query <addr>...

bub handles get <address>
bub handles availability <address> [--kind imessage|facetime]  (needs private API)
bub handles query [--address A]

bub schedule list
bub schedule create <chat> <text> --at "YYYY-MM-DD HH:MM"
bub schedule update <id> --at ...
bub schedule delete <id>

bub findmy devices
bub findmy friends

bub raw <METHOD> <path> [--param k=v]... [--body JSON]   # escape hatch
```

`<chat>` / `<addr>` accept a phone number (`+15551234567`), an email
(`foo@icloud.com`), or an existing chat GUID (`iMessage;+;123...`).
Numbers/emails are converted to 1:1 chat GUIDs automatically.

## Examples

```sh
bub messages list --limit 10 --chat +15551234567
bub messages search "invoice"
bub messages send +15551234567 "On my way"
bub messages send user@example.com "photo" --attach ~/Pictures/cat.jpg
bub contacts list --search dentist
bub attach download <attachment-guid> -o /tmp/clip.mov
bub schedule create +15551234567 "Good morning!" --at "2026-10-13 07:00"
```

## Gotchas (learned the hard way)

- `tempGuid` is **required** for AppleScript sends (the CLI generates one).
- Sending to a chat the server has never seen works (it creates the chat),
  but *querying* messages by address needs the real DB chat GUID — the CLI
  resolves this via `/handle/query` automatically.
- `private-api` is **off** on this server, so: no message edit/unsend, no
  reactions, no availability checks, no reply-to. Enable it in the BB app
  if you need those.
- AppleScript sends depend on macOS granting BlueBubbles Server
  **Automation → Messages** permission (System Settings → Privacy &
  Security → Automation). Without it, sends hang ~5 min then fail with
  `AppleEvent timed out (-1712)`. Read paths are unaffected.
- The server restarts Messages.app on AppleScript failure; the first send
  after that is slow.
- `bub contacts list` returns macOS Contacts as seen by the server (it
  needs Contacts access, which this server has).
