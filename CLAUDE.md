# Yankees 2026 Highlights — maintenance guide

Live site: **https://tgon21.github.io/yankees-2026-highlights/**
Repo: `Tgon21/yankees-2026-highlights` (GitHub account **Tgon21**) · GitHub Pages serves `main` branch root.
Owner: Tyler. This is a single-file static site — everything lives in `index.html`.

## What the page is

Every 2026 New York Yankees game as a card: click-to-load YouTube highlight embed
(official MLB channel only), date tile, matchup, final score (W/L chip), a one-line
MLB.com recap link, and a Gameday box-score link. Below the games: AL East standings
and Yankees batting/pitching tables. Paginated one month per page; searchable by
opponent; sortable newest/oldest; "Hide scores" toggle hides result chips AND recap
lines for spoiler-free viewing.

**Rules that must survive any edit:**
- 2026 season only. Tyler explicitly does not want prior seasons on the page.
- Videos must be from MLB's official YouTube channel (embedded via youtube-nocookie.com).
- Every game must reconcile against MLB's official schedule before publishing —
  never ship a game list that hasn't been checked for missing dates/doubleheaders.
- Scores/standings/stats/recaps all come from MLB's Stats API (statsapi.mlb.com,
  public, no auth). Do not scrape Baseball Reference (no API, scraping disallowed).
- Keep the "Hide scores" spoiler behavior intact when touching cards or recaps.

## Where the data lives

All data is baked into `index.html` as JS consts (no runtime fetching):

- `const GAMES=[...]` — one object per game:
  `{id, date, opp, homeGame, post, dh, ys, os, w, pk, rh, rs}`
  - `id` YouTube video ID · `date` ISO · `opp` short team name ("Red Sox")
  - `homeGame` true = Yankee Stadium · `post` postseason label or null
  - `dh` doubleheader game number (1/2) or null
  - `ys`/`os` Yankees/opponent final score · `w` win flag
  - `pk` MLB gamePk → box score at `mlb.com/gameday/{pk}`
  - `rh` recap headline · `rs` recap slug → `mlb.com/news/{rs}`
- `const STAND=[...]` — AL East rows `{team, w, l, pct, gb, streak, l10}`
- `const PLAYERS={hit:[...], pit:[...]}` — batting/pitching tables

## How to refresh (new games / updated standings & stats)

1. **Schedule (source of truth).** Completed games, scores, gamePks:
   `https://statsapi.mlb.com/api/v1/schedule?sportId=1&teamId=147&startDate=2026-03-20&endDate=<today>&gameType=R,F,D,L,W`
   Count completed games (`status.detailedState == "Final"`); this is the target
   number GAMES must match. `gameNumber` disambiguates doubleheaders.

2. **Find highlight videos.** yt-dlp is bot-blocked on this server; use curl with a
   browser User-Agent and parse `ytInitialData` JSON out of the HTML:
   - Primary: playlist `PLDN7hDtp_5qdCVzt4AXqUwiIdCdWFykBK`
     ("2026 New York Yankees Full Game Highlights") — lags ~2 weeks behind.
   - Gap fill: YouTube search sorted by date
     (`/results?search_query=YANKEES+official+full+game+highlights+2026&sp=CAISAhAB`),
     keep only videos whose channel/owner is **MLB**.
   - Playlist pages return 100 items; page the rest via POST to
     `youtube.com/youtubei/v1/browse?key=<INNERTUBE_API_KEY from page HTML>` with
     `{"context":{"client":{"clientName":"WEB","clientVersion":"2.20250730.01.00"}},"continuation":"<token>"}`.
   - Title formats seen (all Away-vs-Home): `Away vs. Home Game Highlights (M/D/26)`,
     `AWAY vs. HOME Full Game Highlights (M/D/26)`,
     `AWAY vs. HOME[ GAME n]: Official Full Game Highlights (Month D) | 2026 MLB Season`.

3. **Recaps.** Per game: `https://statsapi.mlb.com/api/v1/game/{gamePk}/content`
   → `editorial.recap.mlb` → take `headline` (rh) and `slug` (rs).

4. **Standings.** `https://statsapi.mlb.com/api/v1/standings?leagueId=103&season=2026&standingsTypes=regularSeason`
   → division id **201** = AL East. Shorten team names to nicknames for display.

5. **Player stats.** `https://statsapi.mlb.com/api/v1/teams/147/roster/fullSeason?season=2026&hydrate=person(stats(group=[hitting,pitching],type=[season],season=2026))`
   — curl needs `-g` (brackets trigger URL globbing otherwise). **Pitfalls:** traded
   players appear once per team stint plus a combined row — dedupe by name keeping
   the row with max PA (hitters) / max IP (pitchers); drop position players from the
   pitching list (`pos != "P"`); apply minimums (≈40 PA, ≈15 IP); sort desc by PA/IP.

6. **Verify before publishing.** Every GAMES date must exist in the schedule and
   vice versa; per-date video count must match per-date game count (doubleheaders);
   home/away flag must agree with the schedule. Also update the two hardcoded
   "Through <date>" labels in the standings/stats section headers. The masthead
   stat strip (games, first/latest, record) computes itself from GAMES.

7. **QA + deploy.** Check HTML tag balance and `node --check` the inline script,
   then `git add -A && git commit && git push` (git identity is configured locally
   in this repo; commits as Tgon21). Pages auto-rebuilds in ~1 min; confirm with
   `gh api repos/Tgon21/yankees-2026-highlights/pages/builds/latest --jq .status`
   until it reports `built`.

## If the season ends / postseason starts

October games get `post` labels like `"AL Wild Card Game 1"` / `"ALDS Game 2"`
(parsed from MLB's title). The page already styles October specially (red accents,
"Postseason & the stretch" header) — just populate `post` and it lights up.
