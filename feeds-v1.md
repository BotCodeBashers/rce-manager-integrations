# Data feeds v1: schedule and brackets

Two read-only JSON feeds for your own tools, such as a stream overlay that
shows what's fighting now and what's up next. Read the
[integration standard](README.md) first; it covers versioning, errors,
caching and rate limits for every feed.

- **[Schedule feed](#schedule-feed):** one event's fights in the order they
  will be run, across all arenas or one.
- **[Bracket feed](#bracket-feed):** every match of one competition's bracket.

## Finding your address

You get both from the event organizer. Each is the address of a page you can
already open, **with `.json` added**:

| You have | Feed address |
|---|---|
| Public schedule link: `https://www.rcemanager.com/schedule?t=Ab3dEf...` | `https://www.rcemanager.com/schedule.json?t=Ab3dEf...` |
| Public bracket page: `https://www.rcemanager.com/6692/7384/bracket` | `https://www.rcemanager.com/6692/7384/bracket.json` |

The organizer's QR code page shows the public schedule link; the bracket page
is linked from the event on RobotCombatEvents. The same feeds also live at
`/api/v1/schedule?t=...` and `/api/v1/brackets/6692/7384`, which is what links
inside the feeds use.

## Schedule feed

```
GET https://www.rcemanager.com/schedule.json?t=<token>
GET https://www.rcemanager.com/schedule.json?t=<token>&arena=Arena%201
```

The token is the `t=` value from the public schedule link. You can send it as
an `Authorization: Bearer <token>` header instead of in the address. Add
`arena=` to get one arena only (use the arena's name exactly as listed in
`arenas`; matches with no arena are listed under `Unassigned`).

### Example

```json
{
  "feed": "schedule",
  "version": 1,
  "generated_at": "2026-10-19T18:04:11Z",
  "event": { "name": "Robot Rumble 2026", "date": "2026-10-19", "rce_event_id": 6692 },
  "arena": null,
  "arenas": [
    { "name": "Arena 1", "progress": { "completed": 12, "total": 40 } },
    { "name": "Arena 2", "progress": { "completed": 8, "total": 24 } }
  ],
  "progress": { "completed": 20, "total": 64 },
  "queue": [
    {
      "id": 18234,
      "queue_index": 1,
      "state": "in_progress",
      "arena": "Arena 1",
      "competition": {
        "id": 311, "rce_competition_id": 7384, "name": "Antweight", "weight_class": "Antweight",
        "bracket_type": "double_elimination", "stage": null,
        "bracket_url": "https://www.rcemanager.com/6692/7384/bracket",
        "bracket_feed_url": "https://www.rcemanager.com/api/v1/brackets/6692/7384"
      },
      "section": "winners",
      "round": 2,
      "round_label": "Winners Round 2",
      "number": 9,
      "robot1": { "id": 501, "name": "Alpha Bot", "display_name": "Alpha Bot", "team": "Team Alpha",
                  "image_url": "https://example.com/alpha.jpg", "seed": 1 },
      "robot2": { "id": 508, "name": "Hotel Bot", "display_name": "Hotel Bot", "team": "Team Hotel",
                  "image_url": null, "seed": 8 },
      "robot1_from": null, "robot2_from": null,
      "robot1_source": null, "robot2_source": null,
      "winner_id": null, "outcome": null, "bye": false, "rematch": false
    },
    {
      "id": 18250,
      "queue_index": 2,
      "state": "waiting",
      "arena": "Arena 1",
      "competition": { "id": 311, "...": "same as above" },
      "section": "winners",
      "round": 3,
      "round_label": "Winners Round 3",
      "number": 13,
      "robot1": { "id": 501, "...": "..." },
      "robot2": null,
      "robot1_from": null,
      "robot2_from": "Winner of M#10",
      "robot1_source": null,
      "robot2_source": { "match_id": 18235, "takes": "winner" },
      "winner_id": null, "outcome": null, "bye": false, "rematch": false
    }
  ],
  "completed": [
    { "id": 18200, "queue_index": null, "state": "complete", "winner_id": 501, "outcome": "ko", "...": "..." }
  ]
}
```

### Fields

| Field | Meaning |
|---|---|
| `event` | The event's `name`, `date` (`YYYY-MM-DD`) and RobotCombatEvents id |
| `arena` | The arena you asked for with `arena=`, or `null` for all |
| `arenas` | Every arena with its `progress` |
| `progress` | `completed` and `total` fights for the whole event (the page's progress bar) |
| `queue` | Fights still to come, **in the order they will be run**: the one in the arena, then what's next. With `arena=`, that arena's order |
| `completed` | Finished fights, in bracket order (competition, then round). RCE Manager does not record when a fight finished, so this is not the order they happened |

Each item in `queue` and `completed` is a [match](#match).

## Bracket feed

```
GET https://www.rcemanager.com/6692/7384/bracket.json
```

No token: bracket pages are public. Until the organizer locks the bracket the
feed says `"published": false` and lists no matches; the same address fills in
once it is locked.

### Example

```json
{
  "feed": "bracket",
  "version": 1,
  "generated_at": "2026-10-19T18:04:11Z",
  "event": { "name": "Robot Rumble 2026", "date": "2026-10-19", "rce_event_id": 6692 },
  "competition": {
    "id": 311, "rce_competition_id": 7384, "name": "Antweight", "weight_class": "Antweight",
    "bracket_type": "double_elimination", "stage": null,
    "bracket_url": "https://www.rcemanager.com/6692/7384/bracket",
    "bracket_feed_url": "https://www.rcemanager.com/api/v1/brackets/6692/7384",
    "arena": "Arena 1", "published": true, "finished": false,
    "external_url": "https://challonge.com/ants_2026"
  },
  "rounds": [
    { "section": "winners", "round": 1, "label": "Winners Round 1" },
    { "section": "losers", "round": -1, "label": "Losers Round 1" },
    { "section": "grand_final", "round": 0, "label": "Grand Finals" }
  ],
  "matches": [
    {
      "id": 18230, "state": "waiting", "arena": "Arena 1", "competition": { "...": "..." },
      "section": "losers", "round": -1, "round_label": "Losers Round 1", "number": 17,
      "robot1": null, "robot2": null,
      "robot1_from": "Loser of M#1", "robot2_from": "Loser of M#2",
      "robot1_source": { "match_id": 18200, "takes": "loser" },
      "robot2_source": { "match_id": 18201, "takes": "loser" },
      "winner_id": null, "outcome": null, "bye": false, "rematch": false
    }
  ],
  "standings": null,
  "finals": null
}
```

### Fields

| Field | Meaning |
|---|---|
| `competition` | The competition (see [match](#match)'s `competition`), plus its `arena`, whether it is `published` (locked), whether it is `finished` (every fight played), and its Challonge or TrueFinals page as `external_url` if it has one |
| `rounds` | Each round once, in the order the bracket page draws them: winners rounds, the 3rd place match, losers rounds, then the grand final |
| `matches` | Every match, in that same order, then by `number` |
| `standings` | Swiss only, otherwise `null`: each robot's `rank`, `robot`, `wins`, `losses`, `played`, `byes` and `buchholz` (the sum of its opponents' wins), best first. No two robots share a rank |
| `finals` | Two-stage tournaments only (Swiss, then an elimination finals), otherwise `null`: whether the finals are drawn yet (`published`), and their `bracket_url` and `feed_url`. The finals are a bracket of their own; read their feed for its matches |

Matches here have no `queue_index` (that is the schedule's order).

## Match

| Field | Meaning |
|---|---|
| `id` | The match's id. Stable while the bracket stays locked; changes if the organizer unlocks and relocks it |
| `queue_index` | Schedule feed only: 1 for the first fight in `queue`, 2 for the next, and so on. `null` in `completed` |
| `state` | `in_progress` (called to the arena), `upcoming` (both robots known, not called yet), `waiting` (a robot isn't known yet, or a Swiss round isn't drawn yet), or `complete` |
| `arena` | The arena it is fought in |
| `competition` | `id`, `rce_competition_id`, `name` (e.g. `"Antweight"`, or `"Antweight Finals"`), `weight_class`, `bracket_type`, `stage` (`"finals"` for a two-stage finals bracket, else `null`), `bracket_url` (the public page), `bracket_feed_url` |
| `section` | `winners`, `losers`, `grand_final` or `third_place`. `null` for round robin and Swiss, which have no sections |
| `round` | The round number: positive for winners rounds, negative for losers rounds, 0 for the grand final, -1 for a single elimination's 3rd place match |
| `round_label` | The round as people say it: `"Winners Round 2"`, `"Losers Round 1"`, `"Grand Finals"`, `"3rd Place Match"`, `"Round 3"` |
| `number` | The match number printed on the bracket ("M#9"). A label, not an id |
| `robot1`, `robot2` | The two [robots](#robot), or `null` for a slot not decided yet. A Swiss bye has `robot2: null` |
| `robot1_from`, `robot2_from` | For an empty slot, what fills it: `"Winner of M#3"`, `"Loser of M#1"`. `null` when the robot is known, and in round robin and Swiss |
| `robot1_source`, `robot2_source` | The same as data: `{ "match_id": 18200, "takes": "winner" }` (or `"loser"`). Use these to draw bracket lines |
| `winner_id` | The winning robot's `id`, once decided |
| `outcome` | `"ko"`, `"decision"` or `"forfeit"`, once decided; may be `null` |
| `bye` | `true` for a Swiss bye: one robot, no fight, counted as a win |
| `rematch` | Swiss only: `true` when the two robots already met and no fresh opponent was possible |

## Robot

| Field | Meaning |
|---|---|
| `id` | The robot's id in this event |
| `name` | The robot's name |
| `display_name` | The name to show: the same as `name`, with the team added — `"Bot 1 (Team X)"` — only when two robots in the competition share a name |
| `team` | The team name |
| `image_url` | A picture of the robot, or `null` |
| `seed` | Its seed in this competition, or `null` |

## What the feeds leave out

- Contact details, payment, pit tables, check-in and safety status.
- Judges' scores.
- The match timer, robot readiness and rest timers. A pulled timer is out of
  date immediately; live timer data is planned separately.
- Finals fights a two-stage queue expects but has not created yet.

## Examples

### curl, re-pulling only when something changed

```sh
URL='https://www.rcemanager.com/schedule.json?t=YOUR_TOKEN'
curl -s -D headers.txt "$URL" -o schedule.json
ETAG=$(grep -i '^etag:' headers.txt | cut -d' ' -f2- | tr -d '\r')

# Later: 304 and no body if nothing changed, 200 and a new schedule.json if it did.
curl -s -o schedule.json -w '%{http_code}\n' -H "If-None-Match: $ETAG" "$URL"
```

### Browser or OBS dock

```js
const url = "https://www.rcemanager.com/schedule.json?t=YOUR_TOKEN&arena=Arena%201";

async function nowAndNext() {
  const res = await fetch(url);          // the browser handles ETags for you
  if (!res.ok) throw new Error((await res.json()).message);
  const feed = await res.json();
  const name = (r, from) => (r ? r.display_name : from || "TBD");
  return feed.queue.slice(0, 3).map(m =>
    `${m.state === "in_progress" ? "NOW" : "NEXT"}: ${name(m.robot1, m.robot1_from)} vs ${name(m.robot2, m.robot2_from)}`
  );
}
```

### Python: what's fighting in each arena

```python
import json, urllib.request

URL = "https://www.rcemanager.com/schedule.json?t=YOUR_TOKEN"

def name(robot, waiting_on):
    return robot["display_name"] if robot else (waiting_on or "TBD")

with urllib.request.urlopen(URL) as res:
    feed = json.load(res)

for arena in feed["arenas"]:
    fights = [m for m in feed["queue"] if m["arena"] == arena["name"]]
    print(f'{arena["name"]}  ({arena["progress"]["completed"]}/{arena["progress"]["total"]} done)')
    for m in fights[:3]:
        label = "NOW " if m["state"] == "in_progress" else "NEXT"
        print(f'  {label} {name(m["robot1"], m["robot1_from"])} vs {name(m["robot2"], m["robot2_from"])}'
              f'  — {m["competition"]["name"]}, {m["round_label"]}')
```

## Questions and requests

Missing something your tool needs? Ask the event organizer to pass it on, or
contact the RCE Manager team. New fields can be added to v1 without breaking
existing tools.
