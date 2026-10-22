# RCE Manager integration standard

Read these guides at <https://www.rcemanager.com/developers> or on GitHub at
<https://github.com/BotCodeBashers/rce-manager-integrations>; they are the same.

How outside tools — stream overlays, arena timers, scoreboards, anything the
robot combat community builds — connect to RCE Manager. The goal is that an
integration user can get a working integration from these docs and examples alone.

## The guides

| Guide | For | Version |
|---|---|---|
| [Data feeds](feeds-v1.md) | The public schedule and public brackets as JSON, read-only | v1 |
| [Key and API](keys-and-api-v1.md) | The event's integration key, and the state of every arena and its timer | v1 |
| [Live connection](live-v1.md) | Hardware timers, stream PCs and scoreboards that follow every change over one WebSocket | v1 |
| [Timer control](timer-control-v1.md) | Starting, pausing, resetting and adjusting an arena's clock from a device, and running it | v1 |
| [Webhooks](webhooks-v1.md) | Bots, scoreboards and stream automation that get a message on every change | v1 |

Everything except the data feeds needs the event's **integration key**. The
organizer finds it on the QR Codes page, on the Stream tab, and can email it
to you with every address filled in. Try things on a test event first.

## Principles

**Organizers stay in charge of access.** A feed only shows what a page the
organizer already shares shows. The schedule feed uses the event's public
schedule link token, so whoever an organizer gives that link to can read the
feed, and nobody else. Brackets are already public, so their feed is too.

**Read-only means read-only.** The v1 feeds answer `GET` only. Nothing a feed
does can change event data.

**Versions only grow.** Each response carries `"version": 1`. Within a version:

- Fields may be **added**. Your tool must ignore fields it does not know.
- New values may appear in a field that lists its values (such as `state`).
  Treat an unknown value as "something else" rather than failing.
- Fields are never renamed, removed, or given a different meaning. A change
  like that is a new version at a new address (`/api/v2/...`), and the old
  version keeps working.

**One way to say each thing.** A match looks the same in every feed it
appears in, so code that reads one feed reads the other.

**No extra servers.** Feeds are served by the same app as the pages, with
caching that makes repeat requests nearly free.

## Rules every feed follows

- **Format:** JSON (`application/json`), UTF-8. Times are ISO-8601 strings in
  UTC, such as `"2026-10-19T18:04:11Z"`.
- **Errors:** a status code plus `{ "error": "<code>", "message": "<sentence>" }`.
  Branch on `error`; show `message` to a person.

  | Status | `error` | Meaning |
  |---|---|---|
  | 401 | `invalid_token` | Missing or wrong token |
  | 404 | `not_found` | Nothing at this address |
  | 429 | `rate_limited` | Too many requests; wait `Retry-After` seconds |

- **Caching:** every response has an `ETag`. Send it back as `If-None-Match`
  and, if nothing changed, you get `304 Not Modified` with no body. Browsers
  do this for you.
- **Rate limit:** 60 requests a minute per IP address, across all feeds. A
  tool that pulls when someone presses a button, or polls every 10–15
  seconds, never comes close.
- **Browser tools:** any web page or OBS browser dock may read a feed
  (`Access-Control-Allow-Origin: *`). Pass the token as `?t=` from a browser;
  that keeps the request simple.
- **Addresses:** use `https://www.rcemanager.com`. Don't cache match `id`s
  forever: they change if an organizer unlocks and relocks a bracket.
