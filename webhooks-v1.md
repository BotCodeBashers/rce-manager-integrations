# Webhooks v1

A webhook is an address of yours that RCE Manager posts to whenever something
happens at an event: a match is called, a timer starts, a result is recorded.
Use it for Discord bots, scoreboards and stream automation. (Hardware timers
are better on the [live connection](live-v1.md): no server to run.)

You need the event's integration key, from the organizer. See
[keys-and-api-v1.md](keys-and-api-v1.md).

## Registering an address

```sh
curl -X POST https://www.rcemanager.com/api/v1/integration/webhooks \
  -H "Authorization: Bearer rcm_live_…" \
  -d url=https://scores.example.com/rcm \
  -d arena="Arena 1" \
  -d label="Arena 1 scoreboard"
```

- `url` (required): an `https://` address on the public internet.
- `arena` (optional): only that arena's messages. Leave it out for every arena.
- `label` (optional, up to 60 characters): what the organizer sees in their
  list.

A test message (`ping`) is sent straight away, so you can see it working.

**Register every time your tool starts.** Registering an address that's
already there (same `url` and `arena`) isn't an error: it answers `200` with
the existing webhook and turns it back on if it had been turned off. That's
how a tool that was offline over a long lunch, or overnight, picks up again.

| Request | What it does |
|---|---|
| `POST /api/v1/integration/webhooks` | Register (201), or re-register (200) |
| `GET /api/v1/integration/webhooks` | List this event's webhooks |
| `DELETE /api/v1/integration/webhooks/{id}` | Remove one (204) |

Up to 10 webhooks per event. Errors are `{ "error", "message" }`:
`invalid_url`, `unknown_arena` (the message lists the arenas),
`too_many_webhooks`, `invalid_token`.

## The messages

Every message is one JSON object, posted with `Content-Type: application/json`:

```json
{
  "v": 1,
  "type": "timer.started",
  "id": "msg_8a1d0c52-…",
  "seq": 1042,
  "event": 236,
  "arena": "Arena 1",
  "at": "2026-10-19T18:04:11Z",
  "data": {
    "match": { "id": 18234, "state": "in_progress", "arena": "Arena 1", "...": "the feeds' match fields" },
    "timer": { "match_id": 18234, "state": "running", "duration_s": 180, "remaining_s": 180,
               "ends_at": "2026-10-19T18:07:11Z", "server_time": "2026-10-19T18:04:11Z",
               "version": 17, "last_source": { "type": "arena_manager" },
               "authority": { "type": "rce_manager" } }
  }
}
```

| `type` | When | `data` |
|---|---|---|
| `match.called` | A match is called to an arena | `match`, `timer` |
| `match.uncalled` | A called match goes back to the queue | `match`, `timer` |
| `match.completed` | A result is recorded | `match`, `timer` |
| `match.reset` | A result is undone | `match`, `timer`, `previous_winner_id`, `previous_outcome` |
| `match.winner_changed` | A result is corrected | `match`, `timer`, `previous_winner_id`, `previous_outcome` |
| `timer.started` / `timer.paused` / `timer.reset` / `timer.adjusted` | The match timer changes, from any screen or a hardware timer | `match`, `timer` |
| `timer.finished` | A running timer reaches zero | `match`, `timer` |
| `queue.updated` | An arena's queue changed: its current match, up next, or progress; or who runs its clock (`authority`) | `arena`, as in the [state reading](keys-and-api-v1.md#state) |
| `ping` | A test message | `webhook_id`, `message` |

- Matches and timers have exactly the fields of the
  [read API](keys-and-api-v1.md#the-timer). Results pulled in from Challonge
  or TrueFinals arrive as `queue.updated`.
- More types and fields will be added. **Ignore anything you don't know.**

## Answering

- **Answer `2xx` within 5 seconds**, then do your work. Anything else (an
  error, a timeout, a redirect) is a failure, and the message is tried again.
- **Answer `410 Gone`** to stop. The address is turned off.

## Order, repeats and gaps

- Each address gets its messages **in order**, one at a time. `seq` counts up
  across the event; sort by it if you store messages.
- A message can arrive **twice** (a retry after your answer got lost). Keep
  the last few `id`s and ignore repeats.
- `seq` can **skip** numbers (a message for another arena, or a test message
  to another address).
- For timers, a higher `timer.version` is always the newer state.
- A retried message keeps its original `server_time`, so count down to
  `ends_at`, not from `remaining_s`.

## When your address doesn't answer

RCE Manager retries the oldest waiting message after 10 seconds, 30 seconds,
1 minute, 2 minutes, then every 5 minutes, keeping everything after it in
line. **After an hour without an answer the address is turned off** and the
waiting messages are dropped. To start again, register again (your tool does
this when it starts), or ask the organizer to click **Turn back on**. Then
read [`/state`](keys-and-api-v1.md#state) once to catch up.

## Checking a message came from RCE Manager (optional)

Each post carries:

```
RCM-Signature: t=1760897051,v1=5d1c…
RCM-Message-Id: msg_…
RCM-Message-Type: timer.started
RCM-Delivery-Attempt: 1
```

`v1` is an HMAC-SHA256 of `"<t>.<raw body>"`, keyed with the event's
integration key, in hex. To check it:

```python
import hashlib, hmac, time

def from_rce_manager(headers, raw_body: bytes, key: str) -> bool:
    parts = dict(p.split("=", 1) for p in headers["RCM-Signature"].split(","))
    expected = hmac.new(key.encode(), f"{parts['t']}.".encode() + raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, parts["v1"]) and abs(time.time() - int(parts["t"])) < 300
```

```js
const crypto = require("crypto");
function fromRceManager(sigHeader, rawBody, key) {
  const { t, v1 } = Object.fromEntries(sigHeader.split(",").map((p) => p.split("=")));
  const expected = crypto.createHmac("sha256", key).update(`${t}.${rawBody}`).digest("hex");
  return v1 && crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(v1)) && Math.abs(Date.now() / 1000 - t) < 300;
}
```

## What the organizer sees

Each webhook in their Integrations section, with **Working**, **Retrying** or
**Turned off**, and a log of the last 20 deliveries ("Timer started · Arena 1
· Delivered"). They can remove one or turn it back on. They don't need to set
anything up.
