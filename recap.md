# MLB Site — Project Recap

A running log of everything built, fixed, and investigated across this project. The site is a set of static HTML/JS pages (no backend) deployed via GitHub Pages, pulling live data from MLB's official Stats API.

**Pages:** `index.html` (schedule sheet), `edge.html` (daily picks + backtest), `guarantee.html` (locks + wildcards), `parlays.html` (parlay builder), `bestbet.html` (4/5/6-leg combos), `streaks.html` (win/loss streaks + closer reliability), `picks.html` (saved-picks tracker).

---

## 1. Architecture migration (the foundational fix)

ESPN's unofficial scoreboard endpoint — which the whole site originally ran on — started hard-blocking browser requests with a CORS/403 error, a server-side block no client-side retry could work around.

- Migrated all five original pages from `site.api.espn.com` to MLB's **official Stats API** (`statsapi.mlb.com`), which supports CORS and a date-range query.
- Replaced the old ~150 separate day-by-day ESPN requests with a **single request** covering the whole season.
- Rebuilt team-ID mapping (MLB's numeric IDs → 3-letter abbreviations), doubleheader/duplicate detection (now backed by the real `gamePk` and `gameDate`), and All-Star Game exclusion (via `gameType === 'R'`).

---

## 2. `index.html` — Schedule Sheet

**Core logic:**
- **Season fetch:** one request for the whole season's completed games (`gameType === 'R'` and `status.detailedState === 'Final'`), deduped by `eventTime|away|home` so a real doubleheader (different timestamps) survives while an accidental duplicate wouldn't.
- **H2H record (`computeH2H(teamA, teamB, asOfDate)`):** counts every meeting between the two teams *strictly before* `asOfDate` (excludes the row's own game and any later meetings — this is the fix for the "hidden tie" bug). If tied, breaks the tie using each team's win % **as of that date** (`STANDINGS_BY_DATE[asOfDate]`, built once during `backfillSeasonRecords`), not today's final standings.
- **Tier classification (`classifyTierFromH2HText`):** parsed straight from the H2H text already computed for that row — `margin = winsFor - winsAgainst`; **Strong** if margin ≥ 3, **Moderate** if 1-2, **Coinflip** if tied, **No data** if no meetings yet. Same thresholds as Edge.
- **Column G display rule:** once a game is final, shows the *actual* winner (not the pre-game pick) with the loser in parentheses, colored green if the pre-game pick matched the real winner and red if it didn't.

**Features & fixes:**
- **Game start time** shown next to the date (fixed twice — once for today's live-merged rows, once for historical rows where `eventTime` was being silently stripped from the data pipeline).
- **Scores** added to the Winner column (small text) for every game, not just today's.
- **H2H Record column (F):** now shows the actual winner of that specific game in small parentheses next to the H2H text.
- **H2H Outcome Winner column (G) — redesigned:** shows the *actual winner* of the game (not the pre-game pick) with the loser in parentheses, colored **green** when the pre-game H2H pick was correct and **red** when it wasn't. Before a game is played, it falls back to showing the pending pick.
- **`(pre-tied)` tag:** flags any completed game whose H2H series was tied *before* the game started (a 50%-backtested coin-flip tiebreak), so a correct or incorrect result doesn't look like a "real" H2H signal when it wasn't one.
- **Tier tag:** the same Strong/Moderate/Coinflip/No-data classification used on Edge now shown in small text right after the (H)/(A) marker.
- **Doubleheader badge:** any game sharing a date+matchup with another game gets a small "DH · Game 1/2" label (confirmed via a real example — a rain-delayed Tigers–Guardians game made up same-day).
- **Point-in-time H2H tiebreak fix:** the tiebreak used to use *today's* final standings for every historical row; now uses each team's record *as of that row's own date* — backtested improvement (56.4% vs. 50.8% on tied-H2H games).
- **Deeper H2H tally fix:** the win-loss tally for a matchup used to include that game's own result before checking whether it was tied — meaning a game that itself broke a tie could never show as having started tied. Fixed to only count meetings strictly before that row's date.
- **Sortable Date column** in the "Today & Live" view (ascending/descending by actual game time, since every row there shares the same date).
- **Auto-refresh gap fix:** the page used to only check "is anything live right now" to decide its refresh cadence, missing the gap between an evening's earlier games finishing and the last couple of night games starting. Now also checks "is anything starting soon."
- **H2H stat boxes** (top of the sheet): Prev Day, Last 5 Days, and Season — each showing `W=X L=X` and a win %, all **excluding pre-tied (coinflip) games** from the tally.
- **"My Picks" checkbox** in the H2H Record column — saves a straight pick to the shared Picks page.

## 3. `edge.html` — Daily Picks + Backtest

**Core logic:**
- **Tier classification (`classifyMatchup`):** same margin rule as the schedule sheet (Strong ≥3, Moderate 1-2, Coinflip tied, No data none) — computed from a *live*, non-point-in-time H2H tally (uses the full season as of today for every row, unlike the schedule sheet's fix). Coinflip ties break via overall season win %.
- **Pitcher signal (`applyPitcherSignal`):** compares the two probable starters' rolling ERA (last 90 days). On a **Strong or Moderate** pick, flags `confirm` if the lower-ERA starter's team matches the H2H pick, `warn` if not — informational only, doesn't change the pick or its displayed rate. (Originally Strong-only; extended to Moderate after a dedicated backtest showed the effect there — 70.5% agree vs. 55.9% disagree — is actually *larger* than on Strong.) On Coinflip/No-data, an ERA gap ≥ **2.00** promotes the game to tier `pitcher_edge` with the lower-ERA team as the pick (threshold and ~72% historical rate are from a one-time backtest, not recomputed live).
- **Closer chip:** shown only when `status === 'in'` and `|awayScore - homeScore| ≤ 3`. Pulls each team's primary closer from the league's top-30 save leaders and their season blown-save rate.
- **Combo generation:** exhaustively enumerates every possible N-leg combination from the eligible pool (Strong always included; Moderate/Pitcher-edge added via the "include secondary tier" toggle; live games excluded by default) and ranks by the product of each leg's tier hit rate — not a random sample (a prior random-sample version could, and did, miss the actual best combo), so the true best combo is always found.
- **Backtest tiles:** live tally of `pick === actual winner` for every classified game this season, per tier, plus a chronological "last 10 of this tier" sub-rate, color-flagged vs. the season average.

**Features & fixes:**
- Pitcher ERA window updated from 60 to 90 days after a re-backtest confirmed the 2.00 threshold held at the larger sample.
- **"Pick's hit rate when picked" column:** per-team historical reliability whenever that team was the H2H pick, independent of the tier average. Highlighted green at ≥70%.
- **Exclude live games from combos** checkbox (default on) — a combo you'd place before first pitch shouldn't include a game already underway.
- **Auto-refresh added** (the page previously only fetched once at load — needed for the closer chip to actually appear/disappear as a game tightens).
- Sortable Tier hit rate / Pick's hit rate columns.
- **"My Picks" checkboxes** — one per straight pick, one inside every generated combo card.
- Pitcher confirm/warn signal extended from Strong-only to also cover Moderate tier (see backtest below).

## 4. `guarantee.html` — Locks + Wildcards

**Core logic:**
- **Lock pick (`pickLockedGame`):** if exactly one team in the matchup is in the user-selected top-10, that team is the automatic pick (by rank mismatch). If both are top-10, picks by H2H record; if H2H is also tied (or has no meetings), falls back to a **home-split vs. road-split** tiebreak (this team's home win % vs. the other team's road win %).
- **Wildcard leader (`computeH2HOutcome`):** same H2H-then-standings-tiebreak logic as the schedule sheet's H2H leader, applied to games where neither team is top-10.
- **Guarantee Combos:** enumerates every possible outcome (2 branches per selected wildcard game, so N wildcards = 2^N combos), each paired with all the locked picks. Combined estimate = product of each leg's own historical hit rate (locks from `LOCK_TEAM_PICK_RATES`, wildcards from `WILD_TEAM_PICK_RATES`).
- **Team pick-rate backtests:** since the Lock rule depends on which top-10 teams the user has selected *today* (unknowable for past days), the historical backtest uses the **current final top-10** as a consistent stand-in for every past day — a deliberate, documented simplification, not a literal replay of past selections.

**Features & fixes:**
- **"Hit rate when picked" columns** added to both the Top-10 Lock picks and Wildcard games.
- **Combined estimate** added to Guarantee Combos (previously had no percentage at all) — product of each leg's own historical hit rate.
- **"My Picks" checkbox** inside every generated combo.

## 5. `parlays.html` — Parlay Builder

**Core logic:**
- **Pick (`pickForGame`):** blends each team's season win % with their H2H win % against tonight's specific opponent — **70% season, 30% H2H** — whichever team has the higher blended score is the pick; that same blended score is the leg's "confidence" used for ranking.
- **Combo generation:** for every leg count from 4 up to the user's max, exhaustively enumerates all combinations *within that pool/size* and keeps the top N — sizes are ranked **separately**, never mixed together. (This was a real bug: an earlier version ranked every size together in one shared list — since every leg's confidence is <1, that shared ranking was always dominated by the shortest size, silently making "max legs" and pool size do nothing. Rebuilt to match how Best Bet was already structured.)

**Features & fixes:**
- Combo generator randomness bug fixed (same exhaustive-search approach as Edge).
- **"My Picks" checkbox** inside every generated parlay.

## 6. `bestbet.html` — 4/5/6-Leg Combos

**Core logic:**
- **Pick (`pickForGame`):** prefers a real H2H lead when one exists (picks the H2H leader outright). If H2H is tied or absent, falls back to a 50/50 blend of season win % and last-10-games win %. Confidence for ranking: when H2H decided the pick, weights H2H win-share 65% / season+L10 blend 35%; otherwise just the season+L10 blend.
- **Combo generation:** three independent calls, one each for 4-leg, 5-leg, and 6-leg, each exhaustively searched and ranked *within its own fixed size* — this is why Best Bet never had the "shorter always wins" problem Parlays had.

**Features & fixes:**
- Combo generator randomness bug fixed (same exhaustive-search approach as Edge).

## 7. `streaks.html` — New Page

**Core logic:**
- **Streak probability (`probAtLeastOneStreak`):** a small Markov-chain-style DP over "current run length" states (0 to N-1) — for each remaining game, extends the run with probability `p` or resets to 0 with probability `1-p`; once a run hits length N its probability mass is removed from the "hasn't happened yet" pool. Games-remaining is estimated as `162 - gamesPlayedSoFar`. Validated against two known closed-form cases (N=1 and N=games) before trusting it.
- **Closer table:** pulls the league's top-30 save leaders in one request, then their full season pitching line in one batched follow-up. "Games ago" for the last blown save is computed from that pitcher's full game log (chronological, one request per closer with at least one blown save), counting appearances since the most recent game where `blownSaves ≥ 1`.

**Features:**
- Every team's actual longest win/loss streak this season, current streak, and streak length adjustable from 3–10 games.
- Closer Reliability table: saves, blown saves, blown rate, ERA, and games-since-last-blown-save.
- Sortable columns throughout; headers wrap onto multiple lines with fixed column widths so the table fits without horizontal scrolling.

## 8. `picks.html` — New Page

**Core logic:**
- **Storage:** a single `localStorage` key (`mlb_site_picks_v1`) holding a JSON array. Each entry is either `type: 'straight'` (`away`, `home`, `pick`, `date`, `detail`, `source`) or `type: 'combo'` (`legs: [{away, home, pick}]`, `combinedEstimate`, `date`, `source`). Every saving page uses the identical `id` scheme per pick (e.g. `edge-{date}-{away}-{home}` or `edge-combo-{date}-{legsSignature}`) so checking/unchecking the same box anywhere stays in sync with this page.
- **No live data fetching** — this page only ever reads/writes `localStorage`; everything it shows was already computed on the page that saved it. Only persists in the specific browser it was saved in — doesn't sync across devices.

**Features:**
- Two sections: **Straight Picks** (table) and **Combo & Parlay Picks** (cards), each removable individually, plus a Clear All button.
- PDF export covering everything saved, reusing the same `html2pdf` approach as the Parlay Builder's export.

## 9. Site-wide fix — "today" and game times now use US Eastern, not the viewer's clock

**The bug:** every page computed "today" (which date's slate to show, where to draw the season fetch's end date, pick-ID keys, etc.) using `new Date()` in the *viewer's own browser timezone* — not MLB's. MLB's schedule data is anchored to US Eastern time (a game's `officialDate` reflects the US baseball day). A viewer well ahead of US time (Europe, say) could have their local calendar day roll over while it's still "today" by MLB's own reckoning — silently pulling the wrong day's games, miscategorizing rows, or producing a pick ID that didn't match what another page (or the same page, later) considered "today." Game times had the same issue in reverse: `formatGameTime()` silently converted a game's start time to the viewer's local clock, so a late game could display a time that rolls into a different calendar day than the date printed right next to it.

**The fix, applied identically across all seven pages:**
- A shared `dateStrET(date)` / `todayStr()` helper, built on `Intl.DateTimeFormat` with `timeZone: 'America/New_York'` — correctly handles the EDT/EST switch automatically, so it stays accurate across the whole season (including any November postseason dates, which fall on the EST side of the switch).
- Every inline "today" computation — season-fetch end dates, today's-slate date parameters, pitcher-ERA rolling-window bounds, `localStorage` pick-ID date keys, and the Parlay Builder / Picks page PDF export's date and filename — now goes through this shared helper instead of the viewer's local clock.
- `formatGameTime()` (on `index.html` and `edge.html`) now always shows a game's start time in US Eastern, explicitly labeled ("EDT"/"EST"), instead of silently converting to the viewer's own timezone.
- **Deliberately left as local time** (not part of this bug, correctly excluded): the "file last modified" version-tag footers, `index.html`'s "Updated [time]" refresh indicator, and `picks.html`'s "added [time]" timestamp on each saved pick — all three describe a *local event* (a page refresh, a click) with no MLB-anchored date to match, so showing them in the viewer's own clock is correct, not a bug.

**Verified before shipping:** tested `todayStr()` and `formatGameTime()` against constructed moments that genuinely cross a day boundary between US Eastern and far-flung timezones (Paris, Tokyo, Auckland) — confirmed each one correctly resolves to the true US Eastern date/time rather than the system's local one, including the trickier 90-day pitcher-ERA window math on Edge.

---

## Confirmed non-bugs (investigated, ruled out)

- **DET–CLE showing twice on Sept 4:** a real, rain-delayed doubleheader — confirmed both via distinct final scores and, later, directly by you (Progressive Field rain delay).
- **Sept 3's unusually light schedule (9 games):** confirmed directly against MLB's own API — a genuinely light day league-wide, not missing data.

## Key backtested findings along the way

- **H2H margin matters, but only past a threshold:** Strong (3+ lead) ≈ 82%, Moderate (1-2) ≈ 64%, Coinflip (tied) ≈ 50% — a real, large gap, not a smooth gradient.
- **Post-game streaks don't predict the next game:** ~50.3% after a win vs. ~49.7% after a loss — essentially no signal, so this was never built into a page.
- **Starting pitcher ERA gap needs to be large to matter:** under 2.00 ERA difference is close to noise; 2.00+ hits ~70%+ on its own, and improves an already-Strong pick from ~81% to ~84% when it agrees.
- **Closers show a large, real reliability gap:** top-10 vs. bottom-10 (by saves) blown-save rate, ~8% vs. ~27%.
- **Parlays cost real probability, even with strong picks:** an unfiltered 2-leg parlay drops from 68.6% (straight) to 46.3% combined; even a Strong-only 2-leg parlay drops from 82% to ~69%.
- **Point-in-time standings tiebreak beats "current" standings for historical accuracy:** 56.4% vs. 50.8% on this season's tied-H2H games.
- **Home/road split strength adds nothing beyond plain season win %:** 58.2% vs. 57.6% baseline — well within noise (both clearly beat "always pick home," 52.6%, but neither beats the other). Not built into the site.
- **Rest days show a plausible but not-yet-proven effect:** teams with 2+ days rest won 52.2% vs. 49.7% for teams on a normal 1-day turnaround — a real direction, but the gap isn't statistically solid yet at this sample size (~30% chance of being noise). Not built into the site; worth re-testing later in the season as the sample grows.
- **Pitcher-ERA agreement matters even more on Moderate-tier picks than on Strong:** 70.5% when the pitcher signal agrees with the Moderate H2H pick vs. 55.9% when it disagrees — a 14.6-point gap, larger than Strong tier's 6.2-point version of the same test, and statistically solid on this sample. This is what justified extending the confirm/warn chip to Moderate tier.

---

*This file is a snapshot as of when it was generated — it won't update automatically as new work gets done. Ask for a refresh if it's fallen behind.*
