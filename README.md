
# Live Draft Board

A personal draft-night display for **one private Yahoo Fantasy league**. It runs on a
laptop, gets thrown up on a TV, and shows the draft board filling in pick by pick while
the league drafts.

It is read-only, single-league, and not publicly hosted. Nothing it fetches is stored,
republished, or shared outside the twelve managers already in the league.

<img width="1600" height="900" alt="draft-board" src="https://github.com/user-attachments/assets/3279a753-214f-4430-8751-b06a0b690f7d" />

## What it shows

**Board view** — every pick in the draft, rounds down the left, teams across the top,
snake order handled. The team on the clock is highlighted with its pick timer. The most
recent pick flashes amber so the room can see what just went without looking at a phone.

**War room view** — a second screen for the people who want more than the board.

[image]

- **Best available** — preseason ADP ranking with everyone already drafted removed
- **Position runs** — what the room has been taking recently, so a run is visible while
  it is happening rather than afterward
- **Roster holes** — starting slots each team still has to fill

## How it works

Yahoo's Fantasy Sports API has no websocket and no webhooks, so the app polls. The
league's `draft_status` field drives the whole lifecycle: the poller idles while the
league is `predraft`, polls actively while `drafting`, and stops once the league flips to
`postdraft`. No manual start or stop on draft night.

Because it polls, the board trails the Yahoo draft room slightly — a few seconds under
normal conditions. It is a companion display, not a replacement for the draft room, and
it never tries to be faster than the source.

See [SPEC.md](SPEC.md) for the data model, endpoints, polling strategy, and failure
handling.

## Yahoo API access

This app needs approved access to the Yahoo Fantasy Sports API
([application form](https://sports.yahoo.com/developer/access/)). Access is read-only,
which is all this app requires — it displays picks, it does not make them.

Endpoints used, all read-only:

| Endpoint | Why |
| --- | --- |
| `league/{key}` | `draft_status`, team count, league name |
| `league/{key}/settings` | roster positions, scoring, draft type |
| `league/{key}/teams` | team names, managers, logos |
| `league/{key}/draftresults` | the picks themselves |
| `league/{key}/players` | eligible player pool for best-available |

## Running it

```bash
cp .env.example .env     # fill in your Yahoo client ID/secret and league key
npm install
npm run auth             # one-time OAuth handshake, stores a refresh token
npm start                # open http://localhost:8080 and cast it to the TV
```

## License

MIT — see [LICENSE](LICENSE).
