# Project Context — World Cup 2026 Draft & Leaderboard

Read this file before making any changes. It covers architecture, data, APIs, and the patterns already established so you don't break things that are working.

---

## What this project is

A single-page web app for a 12-person World Cup 2026 draft pool. 12 friends each hold 4 teams (one per pot). The leaderboard tracks live match scores and awards points throughout the tournament. There is also a full schedule view with a "Next Up" section that highlights games where two players' teams are facing each other.

**Live URL:** https://ballesty-liam.github.io/worldcup2026/

---

## Architecture

**One file only: `index.html`**

No framework. No build tools. No npm. No dependencies beyond flagcdn.com for flag images and fonts from the system stack. Edit `index.html` directly. To deploy, commit and push to `master` — GitHub Pages auto-deploys within ~1 minute.

```
worldcup2026/
  index.html   ← the entire app (HTML + CSS + JS, ~1100 lines)
  Milan.jpg    ← joke image used on the "Buy me a coffee" tab
  README.md
  CLAUDE.md    ← this file
```

The file is structured as:
1. `<style>` — all CSS (CSS variables, per-view styles, mobile @media block at the bottom)
2. HTML — five view divs (`#view-home`, `#view-draft`, `#view-leaderboard`, `#view-schedule`, `#view-coffee`) plus a sticky `<header>`
3. `<script>` — all JS (data constants, draft logic, confetti, leaderboard, schedule)

---

## Views / tabs

| Tab button | View div | Notes |
|---|---|---|
| 🏠 Home | `#view-home` | Info cards, scoring rules, CTA buttons |
| 🎲 Draft | `#view-draft` | Shows locked draft results (`LOCKED_DRAFT`) |
| 🏆 Leaderboard | `#view-leaderboard` | Live scores table, expandable per-team breakdown |
| 📅 Schedule | `#view-schedule` | Next Up section + full 104-match schedule |
| 🙏 Buy me a coffee! | `#view-coffee` | Just a photo of Milan |
| ⚠️ Report Issue | *(button, no view)* | Opens a pre-filled GitHub issue |

`showView(v)` handles all view switching, tab active states, and triggers data loads.

---

## Key data constants (do not modify these)

### `PARTICIPANTS`
Array of 12 player names in draft order. Used for display only.

### `POTS`
Four arrays of 12 teams each (Pot 1–4). Each team has `{n: 'Name', c: 'iso-code'}`. The `c` field is the flagcdn.com ISO country code (e.g. `gb-eng` for England, `gb-sct` for Scotland).

### `TEAM_CODE`
Auto-built lookup: `{ 'Spain': 'es', 'Argentina': 'ar', ... }`. Use this whenever you need a flag image from a team name.

### `LOCKED_PARTICIPANTS` ← **never change this**
The permanent draft results. Array of `{name, teams: [pot1team, pot2team, pot3team, pot4team]}`. This is the source of truth for who owns which team. Used by both the leaderboard and the schedule.

### `LOCKED_DRAFT` ← **never change this**
The full draft ceremony state, used to replay the reveal ceremony and show the draft results page.

### `FIXTURES`
All 104 World Cup matches as a hardcoded array. Each entry: `{d: 'UTC datetime', g: 'A'–'L', h: 'Home team', a: 'Away team'}` for group stage, or `{d: 'UTC datetime', st: 'R32'|'R16'|'QF'|'SF'|'3rd Place'|'Final'}` for knockout rounds (teams TBD). Team names in FIXTURES use our internal names (matching `LOCKED_PARTICIPANTS`), not ESPN's display names.

---

## Scoring system

Defined in the `S` constant:
```js
{ groupWin:3, groupDraw:1, knockoutWin:3,
  bonusR32:1, bonusR16:1, bonusQF:2, bonusSF:2, bonusFinal:3 }
```

`scoreTeam(teamName)` iterates `allFixtures` (the live data from ESPN), finds finished matches for a given team, and returns `{mp, bonus, gd, total, results, played}`.

The leaderboard sums all four teams per player. Tiebreaker order: total → match points → goal differential.

The leaderboard table has **6 columns**: `#`, Participant, Match, Bonus, GD, Total. The expand row uses `colspan="6"`. Don't change this to 5.

---

## Live scores API

**ESPN public scoreboard — no API key required.**

```
GET https://site.api.espn.com/apis/site/v2/sports/soccer/fifa.world/scoreboard?dates=20260611-20260719
```

Returns all ~100 tournament events. Called in `refreshData()` which runs on leaderboard/schedule tab open and every hour via `setInterval`.

### ESPN → internal name normalization

ESPN uses different team names than our internal data. The `ESPN_NAME_MAP` object handles this:

```js
const ESPN_NAME_MAP = {
  'Czechia':        'Czech Republic',
  'Türkiye':        'Turkey',
  'United States':  'USA',
  'Cape Verde':     'Cabo Verde',
  'Congo DR':       'DR Congo',
};
```

`espnEventToFixture(event)` converts a raw ESPN event into the shape `scoreTeam()` and `getMatchScore()` expect:
```js
{ event_home_team, event_away_team, event_final_result, event_status, stage_name, league_round, event_date }
```

`event_status` values used: `'Finished'`, `'inprogress'`, `'Scheduled'`.

If you ever need to handle a new API or additional team name mismatches, add to `ESPN_NAME_MAP` and update `espnEventToFixture`.

---

## Schedule tab

`renderScheduleView()` is the entry point — it calls `renderNextUp()` and `renderFullSchedule()`. It's a no-op if `#view-schedule` is hidden.

**Next Up section** (`renderNextUp`):
- Finds today's fixtures in the user's local timezone using `localDateKey()`.
- Falls back to the next upcoming date if no games today.
- Shows "⚔️ PLAYER MATCHUP" badge when two different players' teams face each other.
- Shows "🔀 same player" badge when one player has both teams in the same group.
- Uses `getOwnerMap()` (built lazily from `LOCKED_PARTICIPANTS`) to look up team → player.

**Full schedule** (`renderFullSchedule`):
- Groups all 104 `FIXTURES` by local date.
- Gold left border on matchup rows, accent border on single-owned rows.
- Knockout rounds (no `h`/`a`) display as "TBD v TBD".
- `getMatchScore(home, away)` looks up the score from `allFixtures` using the fuzzy `teamMatch()` function.

---

## CSS conventions

CSS variables are defined in `:root`. Key ones:
```css
--bg, --surface, --surface2, --border   /* backgrounds */
--accent   /* gold #f59e0b — primary highlight */
--green, --red, --purple                /* status colours */
--text, --muted                         /* typography */
```

**Mobile breakpoint:** `@media(max-width:640px)` block at the bottom of `<style>`.

Mobile header pattern: `.h-nav` gets `order:3; width:100%` so it wraps to a second row below the title+status. Do NOT use `flex-direction:column` on `header` — it breaks `overflow-x:auto` on iOS Safari.

The leaderboard table wrapper uses `overflow-x:auto` (not `overflow:hidden`) so mobile users can swipe to see all columns.

**Flag images:**
```js
function flagImg(code, h) {
  const w = h <= 24 ? 20 : 40;
  return `<img src="https://flagcdn.com/w${w}/${code}.png" ...>`;
}
```
Always pass an ISO code from `TEAM_CODE[teamName]`. Check `TEAM_CODE[name]` exists before calling.

**XSS:** Always wrap user-facing strings in `esc()` when building innerHTML.

---

## Deployment

```bash
git add index.html
git commit -m "Description"
git push origin master
```

GitHub Pages picks up `master` automatically. Live within ~60 seconds of push.

---

## Things that have already been tried / gotchas

- **AllSportsAPI** (`allsportsapi2.p.rapidapi.com`) — dead, returns 404. Do not use it. We replaced it with ESPN.
- **ESPN date range** — `?dates=YYYYMMDD-YYYYMMDD` returns up to ~100 events covering the full tournament in one call. Works without auth.
- **`flex-direction:column` on the sticky header** — breaks `overflow-x:auto` nav on iOS Safari. Use `order:3; width:100%` on `.h-nav` instead.
- **`overflow:hidden` on `.table-wrap`** — prevents horizontal scroll on mobile. It is now `overflow-x:auto`.
- **Draft is locked.** The ceremony already happened. `LOCKED_PARTICIPANTS` and `LOCKED_DRAFT` are the permanent source of truth. Do not add UI to re-run the draft.
