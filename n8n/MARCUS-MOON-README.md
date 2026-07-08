# KAZM Marcus Moon — Overnight DJ Breaks

An n8n workflow that writes a fresh Marcus Moon talk break every hour of the
overnight shift, voices it with ElevenLabs, and drops it into rotation. Every
fact he says — weather, songs, local events — is fetched and verified by the
workflow *before* the script is written, so nothing is ever hallucinated.

## The DJ Engine (multi-DJ)

Marcus is no longer the only voice. **`kazm-dj-engine.json`** (live n8n workflow
`KAZM DJ Engine — multi-DJ breaks`, id `YZaIzgSsLen6ldSX`) generates breaks for
any DJ in the roster using the same verified-facts machinery — weather,
AzuraCast history, the site's event/concert/festival/movie feeds, website
spotlights, the variety director, self-ID — while the persona, voice, length,
and output file come from a per-DJ profile.

- **Roster (the control panel):** `/Charles Helstein/DJ Roster/djs.json` in
  Dropbox — edit it there and the *next break* picks up the change; no
  workflow edits. Fields per DJ: `voiceId` + `ttsModel` (ElevenLabs; set
  `audioTags: true` only for v3 voices — tags like `[laughing]` are kept in
  the script and performed), `persona` (character only — the engine adds
  time-of-day delivery, weather modes, and the anti-hallucination hard
  rules), `wordRange`, `dropboxPath` (the break file MegaSeg plays),
  `selfIdName` (spelled how it should be *spoken* — e.g. Kaley is written
  "Callie"), `stationIdLine` (exact required sign-off or empty for a natural
  mention), `active`. A reference copy lives at `dj-roster.example.json`.
- **Make a break:** `…/webhook/kazm-dj-now?k=moon-2026&dj=burt`
  (`canyonjack`, `kaley`, …). The MP3 overwrites that DJ's `dropboxPath` —
  point a MegaSeg playlist/event at the file exactly like the Quiet Storm
  DJ VO pattern.
- **Read a DJ's logbook:** `…/webhook/kazm-dj-log?k=moon-2026&dj=burt`
  (omit `dj` to list known DJs). Openings/angles/spotlight memory is kept
  **per DJ** in `staticData.djs`, so each personality varies independently.
- Marcus's dedicated workflow continues to run the Quiet Storm (it also does
  MegaSeg playlist control, which is per-show wiring); the engine covers
  everyone else.

**Dayparts & rotation song control.** The engine runs the daytime schedule
itself: a cron (`21 6-23 * * 1-5`, America/Phoenix, **weekdays only** to
match the MegaSeg events) fires **once an hour** (6 breaks per shift)
and `Select the DJ` maps the hour to the on-duty host — **Burt 6a–noon,
Canyon Jack noon–6p, Kaley 6p–midnight** (hours 6–11 / 12–17 / 18–23). Each
run also downloads the MegaSeg Database and picks **4 songs from the daytime
rotation** — weighted 60% `* A. HEAVY ROTATION` / 30% `***** B. MEDIUM` /
10% `********* C. LOW` (pools ~151/151/328), skipping recently-aired titles
and a shared 60-song no-repeat memory — one **lead-in** and three **after**,
then rewrites **that DJ's own MegaSeg playlist** (`Playlists/Burt DJ VO`,
`Canyon Jack DJ VO`, `Kaley DJ VO` — from the roster's `playlistName`, or
`<name> DJ VO`): lead-in row → the DJ's VO row → the three songs. The DJ can
back-announce the lead-in, tease the next songs, or bracket both — all
guaranteed, same as Marcus.

**MegaSeg Playlist Rules compliance.** The station runs Playlist Rules
(artist separation 30 min, title separation 30 min, "prevent tracks from
playing in the same hour as yesterday"), and MegaSeg will substitute any
inserted song that violates them — which would falsify the DJ's
back-announce/tease. So every pick is screened (`ruleSafe`, never relaxed)
with margin to spare: nothing played in the last **90 minutes** (per the
library's `15]` Last Played), nothing that played **yesterday during this
same clock hour**, **distinct artists and titles within the block**, no
artist heard on air in the last hour (AzuraCast history), and no artist with
any library track played in the last **60 minutes**. Freshness preferences
(recently-aired titles, the 60-song memory) can relax if pools run dry; the
rule screen cannot. Verified offline: 300 blocks picked against the real
database, zero rule conflicts.

**Shift-opening intro.** The **first break of each shift** (6:21 AM /
12:21 PM / 6:21 PM run) is the DJ's show open, and it runs long —
**90–150 words (~30–60 seconds)** regardless of the DJ's normal break
length (override per DJ with roster `introWordRange`). They greet Sedona
and the Verde Valley, introduce themselves by name (self-ID forced on,
placed early), tell listeners they're aboard **for the next six hours**,
and sell the ride ahead — great music, hit after hit, you're in for a
treat — in their own words, explicitly told to phrase it differently from
their recent openings so no two days sound alike. That break never leads
with a song — weather, an event, or the website is woven into the
greeting. Subsequent breaks are normal length.

**Per-shift song quota.** Each daypart DJ does **at least 3 song
lead-in/lead-out breaks per 6-break shift** (only counted when song control
is armed, so every back-announce/tease is true on air). The director tracks
`staticData.djs.<id>.shiftSongs` per day: while the quota is outstanding the
`song` angle carries extra weight (blocked back-to-back but not the full
3-break memory), and if the shift is running out of breaks the remaining ones
are **forced** to song leads — simulation over 20k shifts lands exactly 3
every time, spread out. The other 3 breaks draw from **weather (exact temps
when it leads), local events, concerts, movies, and the website** — daypart
DJs never do pure-reflection breaks, and they **may casually reference the
day and part of day** ("this Tuesday afternoon") though never a precise clock
time. Before MegaSeg is wired (`voReady` false) shifts are all non-song
topics — no promises the playout can't keep.

**Bed music (no more dry reads).** Daypart breaks are mixed over an
instrumental bed before delivery: `Voice the break` → save to disk →
pick a **random bed** from Dropbox **`/Charles Helstein/DJ Beds/`** (drop
audio files there to change the sound — currently `70s Vintage Rock
Main.mp3`; each break also drops in at a random spot in the bed so
repeats never sound identical) → `Mix the bed` runs **ffmpeg**
(static build self-installed at `/home/node/.n8n/bin/`, survives
restarts) — the bed opens on its own for 1.5s
(level 0.40), makes **one smooth 0.6s dip** as the voice comes in,
holds at a **constant 0.16 under the whole read** (no dynamic
ducking/pumping), and fades out 2.2s right after the last word;
timing comes from ElevenLabs **character timestamps** (the TTS call
uses `/with-timestamps`, and `Unpack the voice` extracts the exact
first/last-word times), so the dip and fade hug the real speech, not
the file length. Voice levels untouched, 160kbps output. Delivery
itself is tunable per DJ via roster `voiceSettings` — ElevenLabs
supports `speed` 0.7–1.2 (Kaley runs 1.1), plus stability /
similarity_boost / style / use_speaker_boost. **Fail-safe:** every mix-chain node
continues on error and `Choose audio` falls back to the dry voice —
an empty bed folder, a missing ffmpeg, or a mix error can never stop
a break. Marcus is not bed-mixed (his show has its own bed-music
category in MegaSeg). Phase 2 (planned): ffmpeg-analyzed intro/outro
talk-over markers on rotation songs.

**Rules Off/On window (no substitutions, ever).** On top of the
rule-safe picker, two static recurring events in Main Events (same
`Rules Off` / `Rules On` actions the station's 3 O'Clock 3 Pack uses)
bracket every DJ VO block: **`:24 Past the hour → Rules Off`** (about a
minute before the `:25:13` insert) and **`:40 Past the hour → Rules On`**.
Same times every hour, no per-hour events needed. Note the `Past the
hour` format fires **every hour, every day** — including overnight and
weekends when no DJ block airs — so rules are off :24–:40 each hour
across the board; MegaSeg's own picks in that window are unscreened.
If the last block song is ever swapped at play time after :40, nudge
the Rules On event later (e.g. `:50 Past the hour`).

**Self-arming (`voReady`).** Song control only activates for a DJ once their
break MP3 exists in **MegaSeg's own library** (the engine looks the file up
in the database by its Mac location, e.g. `…:DJ Breaks:Burt Break.mp3`, and
needs the record to build the VO playlist row). Until then breaks still
generate and upload, but no playlist is written and no songs are promised —
so scripts are never inaccurate during rollout. **One-time setup in MegaSeg —
DONE (2026-07-07):** the three break files were added to the MegaSeg library
(uncategorized, so auto-rotation never plays a bare voice break), and all 18
Main Events were written directly into the events file
(`MegaSeg copy/Events/Main Events` — a plain-text list; the Priority flag is
a 🚩 character, day spec `Mon/Tue/Wed/Thu/Fri`, and times use a narrow
no-break space before AM/PM; original preserved alongside as
`Main Events backup 2026-07-07 pre DJ VO`). Events are **Mon–Fri only,
Priority**, one per hour at `:25:13`, inserting the playlist that matches
the daypart (the n8n run at `:21` stays ~4 minutes ahead, and `:25:13`
keeps the DJ block clear of the `:22` AZ State News inserts). MegaSeg
reloads the file on relaunch or via the daily
`Switch Events: Main Events` at 5:56 AM:

| Playlist to insert   | Event times (M–F, Priority)                                       |
| -------------------- | ----------------------------------------------------------------- |
| `Burt DJ VO`         | 6:25:13 AM, 7:25:13, 8:25:13, 9:25:13, 10:25:13, 11:25:13 AM      |
| `Canyon Jack DJ VO`  | 12:25:13 PM, 1:25:13, 2:25:13, 3:25:13, 4:25:13, 5:25:13 PM       |
| `Kaley DJ VO`        | 6:25:13 PM, 7:25:13, 8:25:13, 9:25:13, 10:25:13, 11:25:13 PM      |

## Two files in this folder

- **`marcus-moon-dj-brain-v2.json`** — snapshot of the **live production
  workflow** (`KAZM Marcus Moon — DJ Brain v2`, n8n id `TXLMR5mFONtoMZg3`).
  This is what is actually on air. It writes hourly overnight breaks **and** a
  weekday show intro, pulls facts from NWS weather + live AzuraCast now-playing
  and history + the mellowmountainradio.com site feeds (events, concerts,
  festivals, movie showtimes) + a human-curated events board, writes the script
  with **Claude** (`claude-opus-4-8`, adaptive thinking, via plain HTTP Request
  nodes), voices it with ElevenLabs, and overwrites an MP3 in **Dropbox** that
  MegaSeg plays on air. Start here.

  > **Song data note:** MegaSeg's `ComingUp.html` feed is no longer used — it
  > went stale (frozen since March 2026) and AzuraCast publishes no
  > `playing_next` for this station, so there is no reliable "coming up next"
  > source. Song talk is therefore driven entirely by **live AzuraCast data**:
  > the current track (`now_playing`, guarded by `elapsed > 90s` so an ad mid-play
  > is never named) and recently-played history. Marcus may tease "more music on
  > the way" but never names an unlisted next song.
- **`marcus-moon-overnight-dj.json`** — the earlier design reference (uploads
  to AzuraCast, Open-Meteo weather, single overnight flow). Kept for history;
  the rest of this README below the next section describes that design.

## The live workflow (`marcus-moon-dj-brain-v2.json`)

**Import / credentials.** Import the file, then bind three n8n credentials —
they are referenced by id in the JSON and hold no secrets in the file itself:

| Node(s) | Credential (type) | Holds |
|---|---|---|
| `Claude writes the break`, `Claude writes the intro` | `Anthropic API Key (Marcus Moon)` (HTTP Header Auth) | `x-api-key` header → Anthropic key |
| `Marcus finds his voice`, `Marcus voices the intro` | `ElevenLabs API Key (Marcus Moon)` (HTTP Header Auth) | `xi-api-key` header → ElevenLabs key |
| `Quiet Storm Liner`, `Quiet Storm Intro` | `Dropbox Connect Ry` (Dropbox OAuth2) | Dropbox account that MegaSeg reads |

**The DJ-brain nodes call Claude directly over HTTP** (`POST
https://api.anthropic.com/v1/messages`) rather than through a langchain node.
Request shape:

```json
{
  "model": "claude-opus-4-8",
  "max_tokens": 4096,
  "thinking": { "type": "adaptive" },
  "system": "<persona + hard rules>",
  "messages": [{ "role": "user", "content": "<fact sheet>" }]
}
```

With adaptive thinking on, the response `content[]` array leads with a
`thinking` block, so the `Clean the copy` / `Clean the intro` nodes select the
first `type === "text"` block rather than assuming `content[0]`.

**Recently-played fact hygiene.** AzuraCast's play history interleaves music
with ads, sweepers, station IDs, news, and local spots, and appends album
names / remaster tags onto song titles. `Build the fact sheet` cleans this so
Marcus only ever references real songs:

- `cleanTitle()` strips the `- Album` suffix and `(2013 Remaster)`-style tags.
- `isBumper()` drops non-music entries using a **duration** signal (real songs
  run 150s+; ads / IDs / spots are ≤60s) plus an artist denylist
  (blank artist, `Live365`, `Mellow Mountain Radio`, `Station ID`,
  `App Announcement`) and a title-token backstop.

**Current events (live from the station site).** Three `Fetch …` HTTP nodes
pull event feeds from the public `chelstein/mellowmountainradio` repo
(`raw.githubusercontent.com/.../main/{library-events,concerts,festivals}.json`),
chained `Ask the airwaves → Fetch library events → Fetch concerts →
Fetch festivals → Build the fact sheet`. GitHub raw serves `.json` as
`text/plain`, so the node body arrives unparsed under `.data`; a `feed()`
helper in `Build the fact sheet` `JSON.parse`s it. The fact sheet then exposes
two event slots, each with its own mention flag (same anti-hallucination model
as everything else — real fetched facts only):

- **Around Sedona** — the human-curated events board plus Community Library
  Sedona events, within 21 days. A curated-board entry wins outright; otherwise
  it rotates among the soonest few.
- **Upcoming show in Arizona** — concerts + festivals filtered to `state ==
  "AZ"`, within 60 days, Verde-Valley cities preferred. Rotates among the
  soonest few on-brand shows (the concerts feed is already genre-curated to the
  station's soft/yacht-rock format).

All feed fetches are `onError: continueRegularOutput`, so a feed being down
never blocks a break.

**Reliability.** Every external call retries automatically: the critical path
(Claude, ElevenLabs, both Dropbox uploads) retries **3×** with 5s between
tries; the data fetches (weather, AzuraCast, all four site feeds) retry **2×**
with 3s between tries, then fail soft as before. Worst case with retries still
finishes in ~1 minute, well inside the 4-minute schedule lead. If a run fails
outright despite retries, MegaSeg simply replays the previous break — the one
known failure mode to keep an eye on, since a replayed break can state stale
"right now" facts.

**On-air leak filter.** `Clean the copy` / `Clean the intro` reject any script
that leaks prompt scaffolding before it can be voiced — "as an AI", "fact
sheet", "mention flag", "angle for this break", "self-ID", "timeBucket".

**Website spotlights.** Marcus also promotes the station site's features. A
curated list of ~14 real pages lives in `Build the fact sheet` — each with a
factual one-line hook — e.g. **Chakras & Tarot** (the full 78-card deck + a
card of the day), **Sound Healing** (the KAZM Harmonic Stack — binaural tone
sessions), **Astrology**, **Cosmic Conditions**, the **Song Time Machine**, the
**Listeners' Lounge**, **Jeep Trails**, **Events & Adventures** (trail / creek /
ski conditions), **Movies** (Sedona Film Festival), **Seen around Sedona**
(wildlife), **Contests**, and more. When a spotlight is featured he names the
page and points listeners to `mellowmountainradio.com`. Recently-used
spotlights are tracked in `staticData` (last 6) so he doesn't repeat.

**The director (variety engine).** So no two breaks feel the same, a "director"
in `Build the fact sheet` picks a **fresh lead angle** for every break instead
of rolling each subject independently. The candidate angles are: `weather`,
`song` (the track on the air now *or* a recently-played one, live from
AzuraCast — one slot, equal weight to every other topic), `local` (Sedona
happening), `concert` (Arizona show), `movie` (Sedona Film Festival screening,
live from `showtimes.json`), `website` (a site spotlight), and `reflection`
(no facts at all — pure overnight mood). Each angle is available only if its
data exists, and is weighted by time of day (overnight favors reflection /
website; drive times favor weather).

> **Songs are one topic, not the default.** An earlier version split songs into
> two angles (now-playing + recently-played), which made music surface roughly
> twice as often as any other subject. They're now a single `song` angle on
> equal footing (~15% of breaks, same as weather / local / concert / movie /
> website), and songs are excluded from the *secondary* slot — so a non-song
> break never also drifts into a song. Result: a genuinely different lead every
> hour, with music as just one of the things Marcus might talk about.

**Self-identification.** Independent of the angle, `mSelfID` rolls **~40%** per
break; when it hits, the fact sheet tells Marcus to work his on-air name into
the sign-off ("this is Marcus Moon on Mellow Mountain Radio"). The rest of the
time he just closes on the station name, so his name never feels forced.

**MegaSeg playlist control (the workflow is the music director).** MegaSeg's
data folder syncs through Dropbox (`/Charles Helstein/MegaSegDropbox/MegaSeg
copy/`), and its playlists are plain TSV files that can be overwritten on the
fly. Each break run now:

1. **Fetch the music library** — downloads `Library/MegaSeg Database`
   (MegaSeg's numbered-field record format: `01]` name, `02]` artist, `05]`
   time, `06]` category, `45]` UniqueID, `49]` Mac-style location, records
   split by `99* ----`).
2. **Pick the next songs** — filters to the `+Quiet Storm` / `+Quiet Storm 10`
   categories (~80 tracks), excludes recently-aired titles (AzuraCast) and
   recently-scheduled UniqueIDs (`staticData.scheduledSongs`, last 32), and
   picks **4**: one **lead-in** that plays directly into the VO, and three
   that play right after it.
3. Marcus's fact sheet gains guaranteed song facts: the **lead-in** ("playing
   right before this break — back-announce it") and the **coming-up songs**
   (locked into the playout — tease as on the way). The song angle rolls one
   of three styles: lead-out only, lead-in tease only, or the classic bracket
   ("that was X… and I've got Y on the way"). All are guaranteed-accurate
   because the same run writes the playlist. The old "playing right now"
   phrasing was removed — it was recorded minutes before airing and couldn't
   survive the delay.
4. After the VO uploads, **Compose + Update the DJ VO playlist** rewrites the
   `Playlists/Quiet Storm DJ VO` file: header + the **lead-in song row** + the
   VO track row (verbatim) + `:cat +Quiet Storm Bed Music` + the 3 explicit
   song rows (25-column TSV, exact Location + UniqueID from the database) +
   `:load Quiet Storm` as a cushion so music never runs dry between breaks.

MegaSeg's Events insert this playlist at 4:20/4:40/5:10/5:30 AM; the n8n runs
fire ~4 minutes earlier, so the fresh VO and song picks are always in place.
The playlist write happens only after the VO upload succeeds — a failed run
leaves the previous playlist intact. The director:

- Picks a **primary** angle, **excluding the last 3 used** (anti-repeat memory
  in `staticData.recentAngles`) — so consecutive breaks never lead with the
  same subject.
- ~35% of the time adds **one** light secondary angle (never a second song
  angle, never `reflection`).
- Hands the model an explicit `ANGLE FOR THIS BREAK:` instruction plus **only
  the facts for the chosen angle(s)** — everything else is omitted from the
  sheet, so the hard "use only listed facts" rule structurally prevents drift.

The result: sometimes weather, sometimes a song, sometimes a movie or concert
or website feature, sometimes just Marcus musing into the quiet — a different
combination every time. A live simulation over 3,000 breaks showed all eight
angles well-distributed and **0% back-to-back repeats**. Feeds:
`Ask the airwaves → Fetch library events → Fetch concerts → Fetch festivals →
Fetch showtimes → Build the fact sheet`.

**Schedule (aligned to the MegaSeg Quiet Storm show, AZ local time).** The
workflow's timezone is pinned to `America/Phoenix` (Arizona has no DST, so the
crons fire at the intended AZ time year-round regardless of the n8n server's
timezone). Each generation fires ~4 minutes ahead of when MegaSeg plays the
file, leaving room for Claude + ElevenLabs + the Dropbox upload/sync to finish:

| MegaSeg plays (AZ) | Content | n8n fires | Dropbox file |
|---|---|---|---|
| 4:00:02a | Quiet Storm Intro | 3:56a (`Intro time`) | `Quiet Storm Show Intro V2.mp3` |
| 4:20:13a | DJ VO break | 4:16a (`Break time strikes`) | `qsvo.mp3` |
| 4:40:13a | DJ VO break | 4:36a | `qsvo.mp3` |
| 5:10:13a | DJ VO break | 5:06a | `qsvo.mp3` |
| 5:30:13a | DJ VO break | 5:26a | `qsvo.mp3` |

Both triggers run **Mon–Fri only** (`* * 1-5`), matching the show's weekday
schedule (the 5:00a Station IDs slot and the 5:55a outro are handled by MegaSeg,
not this workflow). If the Quiet Storm ever airs weekends, change the two
schedule triggers' day-of-week from `1-5` to `*`.

**Test it** with the same webhook as below
(`.../webhook/kazm-moon-now?k=moon-2026`); the result MP3 lands in the Dropbox
path on `Quiet Storm Liner`, and the last 10 scripts are visible via the
logbook webhook.

---

_The sections below document the earlier `marcus-moon-overnight-dj.json`
design._

## How it works

```
10pm–4am hourly (Phoenix)          ┌────────────────────────────────────────┐
        │                          │  FACT SHEET (only truth Marcus knows)  │
        ▼                          │  • AzuraCast: just played / recent /   │
  Ask the airwaves ──────────────▶ │    on deck (live now-playing API)      │
  Check the Sedona sky ──────────▶ │  • Open-Meteo: Sedona temp, sky, low,  │
  Events board (human-curated) ──▶ │    tomorrow's high                     │
                                   │  • Events you posted to the board      │
                                   │  • Phoenix time + hour-of-night mood   │
                                   └───────────────┬────────────────────────┘
                                                   ▼
                        Claude writes the script (persona prompt, hard rules:
                        "use ONLY the fact sheet", 60–110 words, no AI tells,
                        varied openings — it remembers its last 4)
                                                   ▼
                        Clean the copy (strip stage directions/markdown,
                        reject leaks, length check)
                                                   ▼
                        ElevenLabs TTS → Marcus Moon voice (rWyjfFeMZ6PxkHqD3wGC)
                                                   ▼
                        Upload MP3 to AzuraCast "DJ Breaks/" + assign playlist
                                                   ▼
                        Sign the logbook + DELETE the previous break
                        (stale weather never re-airs)
```

## Setup (one time)

1. **Import**: n8n → Workflows → Import from File → `marcus-moon-overnight-dj.json`.
2. **Paste two API keys** (open the node, edit the header value):
   - `Marcus writes the script` → `x-api-key` → your Anthropic API key
     (console.anthropic.com).
   - `Marcus finds his voice` → `xi-api-key` → your ElevenLabs API key.
3. **Create the playlist in AzuraCast**: Playlists → Add: name it **DJ Breaks**,
   type *General Rotation*, play **once every 3 songs** (tune to taste),
   schedule it **10:00 PM – 6:00 AM** so he only talks on his own shift.
   Note the playlist's numeric ID (it's in the URL when you edit it).
4. **Set the playlist ID**: open the `Package the tape` node and change
   `const PLAYLIST_ID = 0;` to that number.
5. **Activate** the workflow.

## Test it right now (any time of day)

```
https://n8n.mellowmountainradio.com/webhook/kazm-moon-now?k=moon-2026
```

That runs the full pipeline once. Then check what he said:

```
https://n8n.mellowmountainradio.com/webhook/kazm-moon-log?k=moon-2026
```

…and listen in AzuraCast → Music Files → `DJ Breaks/`.

## The events board (how Sedona happenings stay accurate)

Marcus only mentions events a human posted — no scraping, no guessing.
All from your browser address bar:

```
# add one
.../webhook/kazm-moon-events?k=moon-2026&do=add&date=2026-07-12&name=Concert in the Park&place=Posse Grounds&note=starts at 6, free

# list (each entry shows its index i)
.../webhook/kazm-moon-events?k=moon-2026

# remove entry 0
.../webhook/kazm-moon-events?k=moon-2026&do=remove&i=0
```

Past-dated events auto-expire. Events within the next 10 days are eligible;
when the board has entries, the flavor rotation leans into them more often.

## Design decisions (and the dials you can turn)

- **Accuracy by construction**: the LLM never fetches or recalls facts. It
  styles a fact sheet the workflow verified. The prompt bans artist trivia
  outright — vibe talk only — because trivia is the one place a model can
  hallucinate confidently.
- **"Up next" honesty**: break playout timing floats, so AzuraCast's
  `playing_next` is teased as "on the way tonight," never promised as the
  literal next song. Recently played songs are past facts and always safe.
- **One break live at a time**: each run deletes the previous MP3 from
  AzuraCast after uploading the new one, so an hour-old temperature can't
  replay at 4am. Archive of what aired lives in the logbook (last 40 scripts).
- **Anti-AI-voice measures**: persona system prompt, 60–110 word ear-first
  writing, spelled-out numbers, remembered openings so he never starts the
  same way twice, a rotating flavor (weather / song vibe / listener love /
  night thoughts / events) so consecutive breaks differ in *subject*, not
  just wording, plus a cleanup pass that rejects any script leaking
  "as an AI" or stage directions.
- **Hours**: cron `2 22,23,0,1,2,3,4 * * *` (Phoenix). Edit the schedule node
  to change the shift.
- **Voice settings**: `eleven_multilingual_v2`, stability 0.55, similarity
  0.75, style 0.35. For a cheaper/faster voice at slightly lower quality,
  switch `model_id` to `eleven_turbo_v2_5` in `Marcus finds his voice`.
- **Cost ballpark**: 7 breaks/night ≈ 7 short Claude calls + ~7 × 100 words of
  ElevenLabs TTS (~4–5k characters/night against your ElevenLabs quota).

## Shared secrets in this workflow

The webhook key `moon-2026` gates the test trigger, events board, and logbook.
Change it in `Check the key`, `Tend the board`, and `Read the logbook` if you
want a different one.
