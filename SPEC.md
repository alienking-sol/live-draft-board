# Specification

## Goal

Display a live Yahoo fantasy draft on a TV, pick by pick, for the twelve people in one
private league. Legible from across a room. Zero interaction during the draft — it starts
itself, updates itself, and stops itself.

## Non-goals

- Making picks, setting lineups, or any write operation
- Supporting leagues other than the one configured in `.env`
- Public hosting, user accounts, or multi-tenancy
- Beating the Yahoo draft room on latency

## Architecture

```
Yahoo Fantasy API  ──poll──>  Node poller  ──SSE──>  browser (TV)
                                   │
                              in-memory state
                              (current draft only)
```

Single process. No database. State lives in memory for the duration of the draft and is
discarded when the process exits. A flat JSON file is written at the end purely so the
board can be reopened after the fact.

### Poller

Reads `league/{key}` and branches on `draft_status`:

| `draft_status` | Behavior |
| --- | --- |
| `predraft` | Poll `league/{key}` every 60s waiting for the flip. Nothing else. |
| `drafting` | Poll `draftresults` every 5s. Refresh `teams` and `settings` once at transition. |
| `postdraft` | Final fetch, write the board to disk, stop polling. |

Static data — league settings, team names, the player pool — is fetched once on
transition into `drafting` and cached. Only `draftresults` is polled on the hot path.

### Polling interval

Yahoo's responses carry a `refresh_rate="60"` attribute, which is the cadence Yahoo
suggests. 5s during an active draft is a deliberate overshoot: a draft lasts about two
hours, so roughly 1,400 requests across the window, and only on one night a year.

If a request returns 429 or 5xx, back off exponentially (5s → 10s → 20s → 40s, capped at
60s) and surface the staleness in the header rather than freezing the display. The header
always shows how long ago the last successful sync was, so nobody is looking at stale
data believing it is live.

### Diffing

`draftresults` returns the full pick list every time, not a delta. Compare incoming pick
count to the previous poll:

- unchanged → no event
- grew → emit one `pick` event per new pick, in order
- shrank or changed → full board resync (Yahoo corrected something)

The last new pick in a batch gets the amber highlight until the next one lands.

### Transport

Server-sent events from poller to browser. One-directional, auto-reconnecting, and enough
for this — a websocket would be more machinery for no benefit.

## Auth

OAuth 2.0 authorization code flow. `npm run auth` runs it once and stores the refresh
token locally in `.tokens.json` (gitignored).

Access tokens expire after roughly an hour. The client refreshes proactively at the
45-minute mark rather than reactively on a 401, because a token expiring mid-draft and
taking a retry cycle to recover is the most likely way this breaks in front of an
audience.

Redirect URI is `https://localhost:8080/callback` and must match the value registered
with the Yahoo app exactly. Yahoo requires https on redirect URIs; plain
`http://localhost` is rejected.

## Data model

```
Draft {
  leagueKey, leagueName, numTeams, numRounds, isSnake
  teams:   [{ teamKey, name, manager, logoUrl, draftPosition }]
  picks:   [{ pickNumber, round, teamKey, playerKey, playerName, position, proTeam }]
  status:  predraft | drafting | postdraft
  lastSync: timestamp
}
```

Derived in the browser, not stored:

- **Board grid** — picks mapped to (round, team) cells, snake order applied on even rounds
- **Best available** — player pool minus drafted keys, sorted by ADP
- **Position runs** — position counts over a trailing window of picks
- **Roster holes** — `settings.roster_positions` minus what each team has drafted

## Views

Two routes, both 1600×900, designed for a TV at typical living-room distance.

- `/` — the board
- `/war-room` — best available, position runs, roster holes

## Testing

The approach that matters: `draftresults` persists after a draft ends, so last season's
league key provides a complete, real draft to develop against. `npm run replay -- <key>`
fetches a completed draft and feeds it to the poller pick by pick on a timer, which
exercises the full diff-and-render path without waiting for draft night.

Draft night should be the second time the app runs end to end, not the first.

## Deliberate constraints

- No write scope requested. Read is sufficient and narrower is easier to justify.
- No player data persisted beyond the current draft.
- Runs on localhost. Not deployed, not exposed, not shared outside the league.
