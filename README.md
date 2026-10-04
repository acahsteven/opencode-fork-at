# opencode-fork-at (`oc-fork`)

Fork an [OpenCode](https://opencode.ai) session at **any** message point — including
messages that are so old the TUI no longer shows them.

## The problem

OpenCode's TUI only keeps the **newest 100 messages per session** in its in-memory
client store:

- on load it fetches `session.messages({ sessionID, limit: 100 })` and keeps `slice(-100)`;
- on every `message.updated` event it `shift()`s the oldest entry once the array exceeds 100.

The **Fork from message** dialog (and the Timeline) read that same store, so anything
older than the last 100 messages simply isn't there — scrolling or searching won't help.

The data itself is not lost: it's all still on the server. `oc-fork` reads the full
transcript and calls the fork API directly, so the 100-message window doesn't apply.

## Requirements

- `opencode` CLI on `PATH` (override with `OPENCODE_BIN`)
- `jq`
- `curl`
- `bash` (works with the ancient 3.2 shipped on macOS)

## Install

```sh
install -m755 oc-fork ~/.local/bin/oc-fork   # or anywhere on your PATH
```

## Usage

```sh
oc-fork <sessionID> --list                    # list forkable user messages (index / time / preview)
oc-fork <sessionID> --search "text"           # most recent match; --first for earliest
oc-fork <sessionID> --index 12                # 12th user message (1 = oldest)
oc-fork <sessionID> --msg-id msg_xxx          # pick directly
oc-fork <sessionID> --index 12 --include      # also include the selected message
oc-fork <sessionID> --index 12 --dry-run      # resolve + show the request, don't fork
```

Get a session id with `opencode session list`.

### Options

| Flag | Meaning |
| --- | --- |
| `--list` | print forkable user messages with a 1-based index |
| `--index N` | select the N-th forkable user message (1 = oldest) |
| `--search TEXT` | select a user message containing `TEXT` (case-insensitive) |
| `--first` / `--last` | earliest / latest match for `--search` (default `last`) |
| `--msg-id msg_xxx` | select a message directly |
| `--include` | include the selected message in the new session (see below) |
| `-d, --directory DIR` | project directory to pass to the server |
| `-u, --url URL` | reuse an already-running server instead of spawning one |
| `-p, --port PORT` | port for the spawned server (0 = random, default) |
| `--dry-run` | resolve the message and print the request without forking |

Human-readable progress goes to **stderr**; the selected/created id goes to **stdout**,
so it composes in scripts.

## Fork semantics (important)

The server copies the transcript **strictly before** the `messageID` you pass — the
target message itself is **not** included. This matches the TUI's *Fork* action, which
then re-sends that message's prompt into the new session.

- **default**: the new session ends just before the selected message.
- **`--include`**: the selected message is included too (internally it targets the next
  message as the cut point). Handy when you want to replay from an old prompt.

## How it works

```
opencode export <sid>   →   resolve target msgID   →   opencode serve + POST /session/{id}/fork
```

1. `opencode export` dumps the complete `{ info, messages[] }` (every `msg_xxx` with its
   parts) — no 100-message cap.
2. The target is resolved with the same filter the TUI dialog uses: `role == "user"`
   with a non-synthetic text part.
3. A short-lived headless `opencode serve` is started; then:

   ```http
   POST /session/{sessionID}/fork
   { "messageID": "msg_xxx" }
   ```

   The server (`Session.fork`) loads **all** messages from storage, takes
   `messages.slice(0, cut)` where `cut` is the target's index (exclusive), creates a new
   session, and re-inserts each message/part with fresh ids — remapping assistant
   `parentID` and compaction `tail_start_id` so references stay consistent.

The spawned server is killed on exit (`trap ... EXIT INT TERM`), and the temporary
transcript file (created `0600` thanks to `umask 077`) is deleted afterwards.

## Notes

- A local OpenCode server is unsecured unless `OPENCODE_SERVER_PASSWORD` is set.
- `rm` unlinks temp files; if the script is `kill -9`'d a temp file may linger in
  `$TMPDIR` (clean it manually).
- The forked session is persisted in OpenCode's own database — that's the product, and
  it's separate from the temporary export.

## License

MIT
