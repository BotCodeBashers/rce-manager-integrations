# Integration key and API v1

The integration key lets your tool read an event's live state, arena by
arena: what's fighting, what's up next, and the match timer. It follows the
[integration standard](README.md) rules for versioning, errors and caching.

## Getting the key

The event organizer gives it to you. It's on their **QR Codes** page, on the **Stream** tab, under
**Integrations**, and they can email it to you with every address already
filled in. One key covers the whole event; there's nothing to configure.

- A real event's key starts `rcm_live_`. A test event's key starts
  `rcm_test_` and works only on that test event. A test event keeps its key
  when the organizer resets it, so you can keep testing with it.
- A key works for one event only. Each new event has its own key.

## Sending the key

Either way works:

```
Authorization: Bearer rcm_live_…
https://www.rcemanager.com/api/v1/integration/state?key=rcm_live_…
```

From a browser page or an OBS dock, use `?key=`: a custom header makes the
browser send an extra preflight request first.

## The readings

| Address | What it gives you |
|---|---|
| `GET /api/v1/integration/whoami` | The event the key belongs to, and the addresses to use |
| `GET /api/v1/integration/state` | Every arena, each labeled: progress, the match in it, the next 3 up, and the timer |
| `GET /api/v1/integration/arenas/{arena}` | The same for one arena. Arena names are URL-encoded (`Arena%201`) |
| `GET /schedule.json?key=…` | The [schedule feed](feeds-v1.md), opened with the key instead of the schedule link's token |
| `POST /api/v1/integration/webhooks` | Have RCE Manager post to you as things happen: see [webhooks-v1.md](webhooks-v1.md) |
| `wss://www.rcemanager.com/cable?key=…` | Keep a connection open and hear about every change: see [live-v1.md](live-v1.md) |
| `POST /api/v1/integration/arenas/{arena}/timer` | Start, pause, reset or adjust an arena's timer: see [timer-control-v1.md](timer-control-v1.md) |

**Running several arenas from one system?** Read each arena by its own
address with the same key. Each reply names its arena, so nothing mixes.

### `state`

```json
{
  "feed": "state",
  "version": 1,
  "generated_at": "2026-10-19T18:04:11Z",
  "event": { "id": 236, "name": "Robot Rumble 2026", "date": "2026-10-19", "rce_event_id": 6692, "test": false },
  "progress": { "completed": 20, "total": 64 },
  "arenas": [
    {
      "name": "Arena 1",
      "progress": { "completed": 12, "total": 40 },
      "current": { "id": 18234, "queue_index": 1, "state": "in_progress", "...": "a match, as in the feeds" },
      "up_next": [ { "id": 18250, "queue_index": 2, "state": "upcoming", "...": "a match" } ],
      "timer": {
        "match_id": 18234,
        "state": "running",
        "duration_s": 180,
        "remaining_s": 143,
        "ends_at": "2026-10-19T18:06:34Z",
        "server_time": "2026-10-19T18:04:11Z",
        "version": 17,
        "last_source": { "type": "arena_manager" },
        "authority": { "type": "rce_manager" }
      },
      "authority": { "type": "rce_manager" }
    }
  ]
}
```

- `current` is the match called to that arena, or `null`.
- `up_next` is the next three in that arena's run order, numbered by
  `queue_index` as on the arena's screen.
- Matches and robots have exactly the [feeds' match fields](feeds-v1.md#match).
- `timer` is `null` when nothing is called.
- `authority` is who runs the arena's clock, even when nothing is called:
  `{ "type": "rce_manager" }`, or `{ "type": "device", "device": "ArenaTimer A1" }`
  when the organizer chose a device. See
  [timer-control-v1.md](timer-control-v1.md#who-runs-the-clock).

`arenas/{arena}` returns `{ "feed": "arena", "version", "generated_at",
"event", "arena": { …one arena, as above… } }`. An arena nothing uses yet is
an empty reply (`current: null`, `up_next: []`, `timer: null`, and its
`authority`), not an error.

### The timer

| Field | Meaning |
|---|---|
| `state` | `idle` (not started), `running`, `paused`, or `finished` (reached zero). More values may be added; treat unknown ones as "something else" |
| `duration_s` | The match length for that competition |
| `remaining_s` | Seconds left as of `server_time` |
| `ends_at` | When it reaches zero, while `running`; otherwise `null` |
| `server_time` | RCE Manager's clock when the reply was made |
| `version` | Goes up by one on every change to this match's timer |
| `last_source` | Who last changed it: `{ "type": "event_manager" | "arena_manager" | "timer_bridge" | "system" | "integration" }`, plus `"device"` (its name) for `integration`; or `null`. More types will be added |
| `authority` | Who runs this arena's clock: `{ "type": "rce_manager" }`, or `{ "type": "device", "device": "…" }`, whose time wins ([timer-control-v1.md](timer-control-v1.md#who-runs-the-clock)). More types may be added |

**Counting down smoothly:** don't poll every second. Read once, then count
down to `ends_at` yourself, correcting for the difference between your clock
and `server_time`. Re-read every few seconds, or when `version` might have
changed, to catch pauses and resets.

## Caching, errors and limits

- Every reply has an `ETag`. Send it back as `If-None-Match` and you get
  `304 Not Modified` when nothing changed, including the moment a timer
  reaches zero.
- Errors are `{ "error", "message" }`: `401 invalid_token` for a missing,
  wrong or other-event key; `429 rate_limited`.
- **60 requests a minute per key**, across this API and the feeds. Devices
  sharing one venue Wi-Fi address don't share that limit; each key has its
  own.

## Examples

```sh
KEY=rcm_live_…
curl -s -H "Authorization: Bearer $KEY" https://www.rcemanager.com/api/v1/integration/arenas/Arena%201
```

```js
const url = "https://www.rcemanager.com/api/v1/integration/arenas/Arena%201?key=rcm_live_…";
const { arena } = await (await fetch(url)).json();
const t = arena.timer;
console.log(arena.current ? `${arena.current.robot1.display_name} vs ${arena.current.robot2.display_name}` : "Nothing called",
            t ? `${t.state} ${t.remaining_s}s` : "");
```
