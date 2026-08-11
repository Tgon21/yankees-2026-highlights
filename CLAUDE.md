# Yankees 2026 Highlights — maintenance guide

Live site: **https://tgon21.github.io/yankees-2026-highlights/**
Repo: `Tgon21/yankees-2026-highlights` (GitHub account **Tgon21**) · GitHub Pages serves `main` branch root.
Owner: Tyler. This is a single-file static site — everything lives in `index.html`.

## What the page is

Every 2026 New York Yankees game as a card: click-to-load YouTube highlight embed
(official MLB channel only), date tile, matchup, final score (W/L chip), a one-line
MLB.com recap link, and a Gameday box-score link. Below the games: a Talkin' Yanks
section (every 2026 episode of Jomboy Media's Yankees podcast, latest 6 shown with a
show-all toggle), then the remaining schedule, AL East standings and Yankees
batting/pitching tables. Games are paginated one month per page; searchable by
opponent; sortable newest/oldest; "Hide scores" toggle hides result chips, recap
lines AND Talkin' Yanks episode titles (titles give away results) for spoiler-free
viewing.

**Rules that must survive any edit:**
- 2026 season only. Tyler explicitly does not want prior seasons on the page.
- Game videos must be from MLB's official YouTube channel; podcast episodes from the
  official Talkin' Yanks channel (`@TalkinYanks`). All embedded via youtube-nocookie.com.
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
- `const TY=[...]` — Talkin' Yanks episodes `{id, ep, date, title}`, newest first
  (sorted by `ep` desc). `id` YouTube video ID · `ep` episode number · `date` ISO
  air date · `title` without the trailing `| NNNN`. The episode-count label renders
  itself from TY.length.
- `const STAND=[...]` — AL East rows `{team, w, l, pct, gb, streak, l10}`
- `const PLAYERS={hit:[...], pit:[...]}` — batting/pitching tables
- `const SCHED=[...]` — remaining (unplayed) games `{date, opp, home, t, dh}`:
  `t` is first pitch in **Eastern time** (convert from the schedule API's UTC
  `gameDate` via America/New_York), `dh` doubleheader game number or null.
  Rebuild from the same schedule call: keep games whose `detailedState` is not
  "Final" and not "Postponed" (postponed originals are dropped — their makeup
  dates appear as separate Scheduled entries). Played games must be REMOVED
  from SCHED when they're added to GAMES. The "games left" count label renders
  itself from SCHED.length.

## How to refresh (new games / updated standings & stats)

1. **Schedule (source of truth).** Completed games, scores, gamePks:
   `https://statsapi.mlb.com/api/v1/schedule?sportId=1&teamId=147&startDate=2026-03-20&endDate=<today>&gameType=R,F,D,L,W`
   Count completed games (`status.detailedState == "Final"`); this is the target
   number GAMES must match. `gameNumber` disambiguates doubleheaders.

2. **Find highlight videos.** Try `yt-dlp --flat-playlist` first if it's available
   and not bot-blocked (it was blocked on the original build server). Fallback that
   always works: curl with a browser User-Agent and parse the `ytInitialData` JSON
   out of the page HTML:
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
   **Exception — new faces:** anyone who debuted or was called up recently and is
   appearing in the lineup gets included even if under the minimums (e.g. George
   Lombard Jr., added after his Aug 4 debut). Compare the roster response against
   the current PLAYERS tables each refresh; if someone new has played, add them
   rather than letting the minimums silently drop them.

6. **Talkin' Yanks episodes.** Source: the official channel `youtube.com/@TalkinYanks`
   (Jomboy Media). **Check BOTH tabs — most episodes are live streams:**
   - `/videos` tab: the weekly studio uploads (Boone pressers, previews).
   - `/streams` tab: the post-game reaction episodes — this is where most of the
     season lives; the Videos tab alone leaves big gaps in the episode numbers.
   Same scraping technique as MLB videos (browser-UA curl + `ytInitialData`; the
   channel tabs use the newer `lockupViewModel` structure, not `videoRenderer`;
   page with the same `youtubei/v1/browse` continuation POST). A fetch sometimes
   returns 0 bytes — just retry. Tab listings only give relative dates ("2 weeks
   ago"); get the exact air date from each watch page:
   `"dateText":{"simpleText":"[Streamed live on ]Mon D, YYYY"}`.
   - Include: numbered episodes (`title | NNNN`), 2026 season = ep 1335 (Mar 22
     season preview) onward. Occasionally a real episode's title omits the number
     (e.g. 1395 "Yankees Get a HUGE Update on Aaron Judge") — do a gap check on
     the number sequence and slot unnumbered episode-shaped streams into the gaps.
   - Exclude: "Watchin'..." game watch-along streams, pre-game shows, and
     compilation videos without episode numbers.
   - Cadence: roughly one episode after every series/notable game, so expect a few
     new ones per refresh. Verify the number sequence has no holes before shipping.

7. **Verify before publishing.** Every GAMES date must exist in the schedule and
   vice versa; per-date video count must match per-date game count (doubleheaders);
   home/away flag must agree with the schedule. Also update the two hardcoded
   "Through <date>" labels in the standings/stats section headers. The masthead
   stat strip (games, first/latest, record) computes itself from GAMES.

8. **QA + deploy.** Check HTML tag balance and `node --check` the inline script,
   then `git add -A && git commit && git push`. Pages auto-rebuilds in ~1 min;
   confirm with
   `gh api repos/Tgon21/yankees-2026-highlights/pages/builds/latest --jq .status`
   until it reports `built`.

## If the season ends / postseason starts

October games get `post` labels like `"AL Wild Card Game 1"` / `"ALDS Game 2"`
(parsed from MLB's title). The page already styles October specially (red accents,
"Postseason & the stretch" header) — just populate `post` and it lights up.
