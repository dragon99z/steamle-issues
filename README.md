# Steamle

**A daily Wordle-style game: read an anonymized Steam review and guess the game in 5 tries.**

🎮 **Play:** https://steamle.dragon99z.de  
🐞 **Report a problem or suggest something:** [open an issue](../../issues/new/choose)

The source code lives in a private repository. This repo is the public issue tracker, and also describes the project: how it plays, how it is built, and how to run it.

---

## For players

### How to play

1. You see a real Steam review with the game's name, characters and places blacked out as `[redacted]`.
2. Guess the game. You have **5 tries**.
3. A wrong guess **or a skip** costs one try and reveals another review of the same game.
4. Win by guessing the game before you run out of tries. The answer is revealed when the round ends.

Things to know:

- **One puzzle per day**, identical for everyone. It changes at **00:00 UTC**.
- **Archive:** every past puzzle can be played later at `/archive` (and `/day/<date>`).
- **Review history:** click any filled try slot or guess chip to look at the review that was on screen at that point.
- **Winning early:** the remaining unseen reviews are unlocked on the result screen so you can read them.
- **Autocomplete** suggests games from Steam's catalog as you type. It's only a typing aid, so any title can be guessed.
- **Reviews** are shown as a Steam-style card (recommended or not, hours played, Early Access badge, date, helpful/funny counts). The reviewer's name and avatar are never shown.

### Your data and privacy

- Your progress (guesses, streaks, win/loss) is stored **in your own browser** (`localStorage`). There is no account, so progress is per device and browser.
- **Anonymous stats:** when a round ends, one row `{ date, tries, won }` can be submitted to build the "X% solved it in 3 tries" chart. No IP address, user agent, cookie or ID is stored with it.
- **Visit counter:** a page view is counted once per visitor per UTC day using a one-way hash that is discarded at midnight. Only date, path and timestamp are stored. No cookies are used for this.
- Review text is written by Steam users. Steamle is an independent fan project and is **not affiliated with Valve Corporation**. Steam and the Steam logo are trademarks of Valve.

### Reporting issues

Use [Issues](../../issues/new/choose):

- **Bug report:** something is broken. Include the page URL and your browser/device.
- **Puzzle / review problem:** a review is offensive, spoils the answer, or a game name is wrong. Include the puzzle date or URL.
- **Feature idea:** anything you'd like to see.

Search existing issues first, and please don't post personal data.

---

## For developers / portfolio

### Tech stack

| Area | Choice |
|---|---|
| Framework | Next.js 16 (App Router), React 19 |
| Language | TypeScript, plus Node ESM scripts (`.mjs`) |
| Styling | Tailwind CSS 3 |
| Runtime | Node.js **22.5+** |
| Storage | JSON files for game content, built-in `node:sqlite` for anonymous stats |
| Deployment | Docker (Debian-slim image, `output: "standalone"`), `docker compose` |
| Tests | Plain Node test scripts, no framework (`npm test`) |

### Features

- Daily puzzle with a pre-built, **append-only schedule** so adding games never changes past answers
- **Answer kept server-side:** the browser only gets the first review; guesses and skips are checked by the API
- Real review data pulled from Steam's Store API, with automatic **redaction** of the game's name (fuzzy matching), spoiler names and profanity, and filtering of spam, URLs, gibberish and non-English text
- Steam-style review card, with browsable try history
- Full archive with per-date status
- Anonymous guess-distribution stats (SQLite) and a cookieless, privacy-friendly visit counter
- **Admin panel** (password + signed session cookie): edit or delete games, reassign any day's puzzle, manage the redaction word list, and run background jobs (extend schedule, pull new Steam games)
- SEO and ads groundwork: sitemap, structured data, social images, editorial pages, ad-script allowlist
- Security headers and a per-request CSP script nonce, login throttling, cross-site request rejection
- Mobile-verified layout (iPhone viewport checked with Playwright, tap targets ≥ ~40px)

### Architecture

```
app/          routes: puzzle, archive, day/[date], editorial pages, admin, API route handlers
components/   GameClient (game state machine), ReviewCard, GuessInput, ArchiveGrid, ...
lib/          puzzle lookup, runtime JSON data store, stats DB, admin auth, search, date helpers
scripts/      Steam fetch / name generation / schedule builder (+ offline tests in __tests__/)
data/         games.json, schedule.json, config.json, gameNames.json, steamle.db (stats)
```

Key design decisions:

- **Runtime data loading.** `games.json`, `schedule.json` and `config.json` are read from disk per request (mtime-cached), not imported at build time, so admin edits show up immediately without a rebuild. This is why `/` and `/archive` are `force-dynamic`: otherwise Next.js would prerender today's puzzle once and serve it forever.
- **Deterministic daily puzzle.** `dayIndex = days since launchDate (UTC)`, looked up in an append-only `{ dayIndex, gameId }` list. Every server gives the same answer without shared state.
- **`node:sqlite` instead of `better-sqlite3`.** Built into Node, so no native compilation or musl/glibc problems. Only simple `INSERT` / `GROUP BY` queries are needed.
- **No Edge middleware for auth.** Admin checks run in the Node runtime (`crypto.timingSafeEqual` over an HMAC-signed, 12-hour cookie), keeping the design small for a single-admin tool.
- **Single Node process assumption** for the data cache, visit dedupe and background jobs. Scaling out would need a shared cache or lock.

### Run it locally

Requires **Node 22.5 or newer** (for `node:sqlite`).

```bash
npm install
npm run dev          # http://localhost:3000
```

Production build:

```bash
npm run build
npm run start
```

No database setup is needed: the stats DB is created at `data/steamle.db` on first use.

Useful scripts:

| Command | What it does |
|---|---|
| `npm run lint` | ESLint |
| `npm test` | Run all offline tests in `scripts/__tests__/` |
| `npm run data:refresh` | Fetch games and reviews from Steam, regenerate names, extend the schedule |
| `npm run fetch:steam` | Fetch reviews (seed list, specific App IDs, `--random=N`, `--count=N`, `--name="..."`) |
| `npm run gen:names` | Rebuild `data/gameNames.json` |
| `npm run build:schedule` | Extend `data/schedule.json` (default horizon: today + 60 days) |
| `npm run fetch:applist` | Cache Steam's full app catalog (needs `STEAM_API_KEY`) |

Environment variables (see `.env.example`):

| Variable | Needed for |
|---|---|
| `ADMIN_PASSWORD` | Enables `/admin` (returns 503 if unset) |
| `ADMIN_SESSION_SECRET` | Signing the admin session cookie |
| `STEAM_API_KEY` | Full-catalog fetch and autocomplete (optional) |
| `GITHUB_TOKEN` | Read-only token for the `/changelog` page (optional) |
| `SITE_URL` | Override the canonical domain for local/staging builds (optional) |

### Pulling real Steam data

`npm run data:refresh` calls Steam's public Store API (`appdetails`, `appreviews`), pages through reviews with Steam's cursor and keeps only games with at least 500 usable reviews. Reviews are cleaned (BBCode stripped, URLs, gibberish and non-English dropped, clues capped at 220 characters) and spoilers are redacted. Requests run one at a time with a delay to respect Steam's API terms. The filter is a heuristic, not moderation, so spot-check `data/games.json` afterwards.

A game entry in `data/games.json` looks like:

```json
{
  "id": "unique-slug",
  "name": "Exact Game Title",
  "appid": 123456,
  "reviews": [
    {
      "text": "clue text, 30–220 characters",
      "recommended": true,
      "hoursOnRecord": 187.4,
      "hoursAtReviewTime": 152.1,
      "earlyAccess": false,
      "datePosted": "2025-06-15",
      "helpfulCount": 340,
      "funnyCount": 12
    }
  ]
}
```

### Deploy with Docker

```bash
cp .env.example .env              # set ADMIN_PASSWORD at minimum
sudo chown -R 1001:1001 ./data    # match the container's non-root user, once
docker compose up -d
```

`./data` is bind-mounted so admin edits and stats survive restarts and rebuilds. The app needs a **writable, persistent filesystem**, so it fits a single container or VM, not stateless or read-only serverless platforms. Keep the Automation job enabled in the admin panel so the schedule never falls behind today (otherwise the game 404s for unscheduled dates).

---

*Steamle is an independent fan project, not affiliated with, endorsed by, or sponsored by Valve Corporation.*
