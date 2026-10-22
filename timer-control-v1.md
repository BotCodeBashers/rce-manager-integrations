# Timer control v1

A hardware timer, or any tool, can **start, pause, reset and adjust** an
arena's match timer: exactly what the Arena Manager's buttons do. The screens,
the stream overlay, webhooks and the live connection all see the change, and
the screens show who made it ("Paused by ArenaTimer A1").

You need the event's integration key ([keys-and-api-v1.md](keys-and-api-v1.md)).

## Name your device

Every command says which device sent it, in 1–40 characters (for example
`ArenaTimer A1`). Give each device its own name; it appears on the timer
screens.

- **Live connection:** add `device` when you subscribe (see
  [live-v1.md](live-v1.md)):
  ```json
  {"command":"subscribe","identifier":"{\"channel\":\"IntegrationChannel\",\"arenas\":[\"Arena 1\"],\"device\":\"ArenaTimer A1\"}"}
  ```
- **REST:** send `device` with each command.

## The commands

| `command` | What it does | Refused when |
|---|---|---|
| `start` | Starts (or resumes) the clock | it's already running (`already_running`) |
| `pause` | Stops the clock where it is | it isn't running (`not_running`) |
| `reset` | Back to the full match time, stopped | never, except as stale |
| `adjust` | Sets the time while stopped; needs `remaining_s` (0–5999). The device [running the clock](#who-runs-the-clock) may also adjust while running | it's running (`timer_running`), or the match is complete (`match_complete`) |

A command acts on the match **called to that arena** at this event. If
nothing is called there, it's refused with `no_match`.

**Fields** (the same for both ways of sending):

| Field | | |
|---|---|---|
| `command` | required | `start`, `pause`, `reset` or `adjust` |
| `arena` | required | the arena's name, exactly as the organizer named it |
| `id` | recommended | your own id for this command, up to 64 characters, unique per command |
| `version` | recommended | the timer `version` you last saw |
| `remaining_s` | `adjust`; optional on `start` and `pause` | the new time in seconds, 0–5999. On `start` and `pause`, the time your device shows: used only when your device [runs the clock](#who-runs-the-clock), otherwise ignored |

### Over the live connection

```json
{"command":"message","identifier":"…your subscription's identifier…",
 "data":"{\"action\":\"timer\",\"arena\":\"Arena 1\",\"command\":\"pause\",\"id\":\"a1-0042\",\"version\":17}"}
```

The answer comes back to your connection only, outside the `seq` count:

```json
{"identifier":"…","message":{"v":1,"type":"ack","id":"msg_…","seq":null,"event":236,"arena":"Arena 1","at":"…",
 "data":{"command_id":"a1-0042","command":"pause","duplicate":false,
         "timer":{"match_id":18234,"state":"paused","remaining_s":83,"version":18,
                  "last_source":{"type":"integration","device":"ArenaTimer A1"},"…":"…"}}}}

{"identifier":"…","message":{"v":1,"type":"error","id":"msg_…","seq":null,"event":236,"arena":"Arena 1","at":"…",
 "data":{"command_id":"a1-0042","command":"start","error":"stale_version",
         "message":"The timer changed since your version. …","timer":{"…":"the current timer"}}}}
```

The change itself also arrives as a normal message (`timer.paused`, with its
`seq`), like any other change.

### Over REST

```sh
curl -X POST https://www.rcemanager.com/api/v1/integration/arenas/Arena%201/timer \
  -H "Authorization: Bearer rcm_live_…" \
  -d command=pause -d device="ArenaTimer A1" -d id=a1-0042 -d version=17
```

`200` with `{ "feed": "timer_command", …, "duplicate": false, "timer": {…} }`,
or an error `{ "error", "message", "timer" }`. Form fields or JSON both work;
from a browser, use form fields.

## Who runs the clock

Normally RCE Manager keeps time, and a device's commands work like the
screens' buttons. The organizer can instead choose a device to run an arena's
clock, on the QR page. They choose it by the `device` name, from the devices
that have connected to the event, so connect at least once before asking
them. One device may run several arenas; each arena has at most one.

While your device runs an arena's clock, **its time wins**:

- Send the time you show as `remaining_s` with `start` and `pause`.
  Everyone takes it, instead of RCE Manager's own count.
- If your clock and RCE Manager's drift apart **by 2 seconds or more**, send
  `adjust` with your time, even while running: the clock keeps running from
  your time. Don't send it more often than that; each one redraws every
  timer screen.
- Pausing at `remaining_s: 0` ends the time: `timer.finished` goes out at
  once.
- The screens' buttons still work. When the Arena Manager presses Pause on
  the tablet, you get `timer.paused` like any change: **follow it**.
- Stay connected (live connection, with your `device` name). The timer
  screens show "ArenaTimer A1 runs this clock" while you're connected and
  "ArenaTimer A1 isn't connected" when you're not, so the Arena Manager
  knows to use the tablet. A device that loses power shows as not connected
  within about 3 minutes; one that closes its connection, at once.

Every timer, and every arena in the state reading, snapshot and
`queue.updated`, says who runs it:

```json
"authority": {"type": "device", "device": "ArenaTimer A1"}
"authority": {"type": "rce_manager"}
```

When the organizer changes it, the arena's `queue.updated` goes out with the
new `authority`. Check it rather than assuming: if your device isn't the one
named, your `remaining_s` is ignored and you should follow RCE Manager's
time.

## Repeats and stale commands

- **`id`:** if a command with the same `id` was already applied to this match
  (for example you resent it after a dropped connection), it isn't applied
  again; the answer says `"duplicate": true`. Use a fresh id for every
  command. If you use a counter, start it from something random at power-up.
- **`version`:** every change to the timer increases its `version`, and every
  message tells you the current one. Send the version you last saw: if the
  timer changed since (say the arena manager pressed Pause on the tablet), your
  command was decided on an out-of-date view and is refused as
  `stale_version`. The error includes the current timer: show it, and send
  again if the operator still wants to. Without `version` there's no check.

## Errors

| `error` | REST | Meaning |
|---|---|---|
| `invalid_token` | 401 | missing or wrong key |
| `invalid_command` | 422 | a field is missing or invalid (the message says which) |
| `device_required` | 422 | no device name (live: subscribe with `device`) |
| `unknown_arena` | 422 | not an arena at this event (the message lists them) |
| `no_match` | 409 | nothing is called in that arena |
| `stale_version` | 409 | the timer changed since your `version` |
| `already_running` / `not_running` / `timer_running` / `match_complete` | 409 | the timer's state doesn't allow it |
| `rate_limited` | 429 | too many requests (REST: 60 a minute per key) |

Every 409 includes the current `timer`, so you can always catch up.

## A device's loop, in outline

1. Connect to the live connection with the key, subscribed to your one arena
   with your `device` name.
2. Keep the latest snapshot's arena and timer; apply messages with a higher
   `seq` (see [live-v1.md](live-v1.md)).
3. When the operator presses a button, send the command with a fresh `id` and
   the `version` you have; if you run the clock, add your `remaining_s`.
4. On an `ack`, or on any error with a `timer`, show that timer.
5. Drive the display from the timer: count down to `ends_at` while running.
   If you run the clock, count on your own and send `adjust` only when the
   two differ by 2 seconds or more.
