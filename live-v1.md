# Live connection v1

Keep one WebSocket open and RCE Manager tells you about every change as it
happens: matches called and decided, timers started and paused, queues
changing. It's the same information as [webhooks](webhooks-v1.md), without
running a server. Use it for hardware timers, stream PCs and scoreboards.

You need the event's integration key, from the organizer. See
[keys-and-api-v1.md](keys-and-api-v1.md).

To control the timer from your device, add a `device` name when you subscribe
and send commands: see [timer-control-v1.md](timer-control-v1.md).

## Connecting

```
wss://www.rcemanager.com/cable?key=rcm_live_…
```

or connect to `wss://www.rcemanager.com/cable` and send the key as a header on
the upgrade request:

```
Authorization: Bearer rcm_live_…
```

The header keeps the key out of web server logs; use it if your WebSocket
library allows it. Send `Sec-WebSocket-Protocol: actioncable-v1-json` if you
can (recommended, not required). Always `wss://`.

Every frame both ways is one JSON object in a text frame.

## The conversation

**1. RCE Manager says hello:**

```json
{"type":"welcome"}
```

**2. You subscribe**, naming the arenas you want (leave `arenas` out for all
of them, including arenas added later):

```json
{"command":"subscribe","identifier":"{\"channel\":\"IntegrationChannel\",\"arenas\":[\"Arena 1\"]}"}
```

`identifier` is a JSON object **written as a string**. Keep that exact string:
every later frame for this subscription carries it back. Up to 20 arenas,
each named exactly as the organizer named it.

**3. You get a snapshot** of the current state, then a confirmation:

```json
{"identifier":"{\"channel\":\"IntegrationChannel\",\"arenas\":[\"Arena 1\"]}",
 "message":{"v":1,"type":"snapshot","id":"msg_…","seq":1041,"event":236,"arena":null,
            "at":"2026-10-19T18:04:11Z",
            "data":{"event":{"id":236,"name":"Robot Rumble 2026","date":"2026-10-19","rce_event_id":6692,"test":false},
                    "progress":{"completed":20,"total":64},
                    "arenas":[{"name":"Arena 1","progress":{"completed":12,"total":40},
                               "current":{"…":"the match in the arena"},"up_next":["…"],
                               "timer":{"match_id":18234,"state":"running","duration_s":180,"remaining_s":143,
                                        "ends_at":"2026-10-19T18:06:34Z","server_time":"2026-10-19T18:04:11Z",
                                        "version":17,"last_source":{"type":"arena_manager"},
                                        "authority":{"type":"rce_manager"}},
                               "authority":{"type":"rce_manager"}}]}}}

{"identifier":"…","type":"confirm_subscription"}
```

Each arena is exactly what the [state reading](keys-and-api-v1.md#state)
gives. The snapshot's `seq` is the last change it includes.

**4. Then every change**, as the messages listed in
[webhooks-v1.md](webhooks-v1.md#the-messages) (`match.called`,
`timer.started`, `queue.updated`, …), each wrapped like this:

```json
{"identifier":"…","message":{"v":1,"type":"timer.paused","id":"msg_…","seq":1042,"event":236,
 "arena":"Arena 1","at":"…","data":{"match":{"…":"…"},"timer":{"…":"…"}}}}
```

**Every 3 seconds RCE Manager also sends:**

```json
{"type":"ping","message":1760897051}
```

You don't need to answer it.

**To ask for a fresh snapshot** at any time (for example after your own code
dropped a frame):

```json
{"command":"message","identifier":"{\"channel\":\"IntegrationChannel\",\"arenas\":[\"Arena 1\"]}","data":"{\"action\":\"resync\"}"}
```

## The rules for applying messages

- **A snapshot replaces everything you have.** A new one can arrive at any
  time: RCE Manager sends one if it notices a message went missing, so you
  never have to work out gaps yourself.
- **Then apply each message.** Ignore one whose `seq` is the same as or lower
  than one you already applied. `seq` skips numbers for arenas you didn't ask
  for; that's normal.
- **Every message carries the complete state** of its match and timer, so
  applying one twice does no harm. For timers, a higher `version` is newer.
- **Count a running timer down to `ends_at`**, corrected by the difference
  between your clock and `server_time`.

## Staying connected

- **Treat 10 seconds without any frame as a dead connection:** close it and
  reconnect.
- **Reconnect with back-off:** wait 1, 2, 4, 8, 16, then 30 seconds between
  tries, plus a little randomness. Go back to 1 second once you've had a
  snapshot.
- **After every reconnect, subscribe again.** You get a new snapshot, so
  nothing is lost.
- Connections drop when RCE Manager restarts (deploys, daily restarts). That's
  normal; the rules above handle it.
- **`{"type":"disconnect","reason":"unauthorized","reconnect":false}`** means
  the key is wrong. Don't retry; show "key rejected".
- **`{"identifier":"…","type":"reject_subscription"}`** means the subscribe
  frame was wrong (for example `arenas` isn't a list of names).

## Limits and sizes

- About **20 connection attempts a minute per key**. More gets an HTTP 429 at
  the handshake; wait a minute.
- A one-arena snapshot is about 5–6 KB; all arenas can be 25 KB or more.
  **Small devices should subscribe to one arena:** many ESP32 WebSocket
  libraries buffer about 15 KB.

## Examples

**Try it from a terminal** with [websocat](https://github.com/vi/websocat),
then paste the subscribe frame:

```sh
websocat -H 'Authorization: Bearer rcm_live_…' wss://www.rcemanager.com/cable
```

**Python** (`pip install websockets`):

```python
import asyncio, json, websockets

KEY = "rcm_live_…"
SUB = json.dumps({"channel": "IntegrationChannel", "arenas": ["Arena 1"]})

async def main():
    delay = 1
    while True:
        try:
            async with websockets.connect("wss://www.rcemanager.com/cable",
                                          additional_headers={"Authorization": f"Bearer {KEY}"}) as ws:
                await ws.send(json.dumps({"command": "subscribe", "identifier": SUB}))
                last = None
                while True:
                    frame = json.loads(await asyncio.wait_for(ws.recv(), timeout=10))
                    msg = frame.get("message")
                    if not isinstance(msg, dict):
                        continue  # welcome, ping, confirm_subscription
                    if msg["type"] == "snapshot":
                        last, delay = msg["seq"], 1
                        print("snapshot", [a["name"] for a in msg["data"]["arenas"]])
                    elif last is not None and msg["seq"] > last:
                        last = msg["seq"]
                        print(msg["seq"], msg["type"], msg["arena"])
        except Exception as e:
            print("reconnecting:", e)
            await asyncio.sleep(delay)
            delay = min(delay * 2, 30)

asyncio.run(main())
```

**ESP32 / small devices**, in outline: open the WebSocket with the key in the
URL or header; on `welcome`, send the subscribe frame for your one arena;
keep the latest snapshot's arena and timer; apply messages with a higher
`seq`; drive your display from `timer.ends_at`; if nothing arrives for 10
seconds, close and reconnect with back-off.
