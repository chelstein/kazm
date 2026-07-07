# KAZM Marcus Moon — Overnight DJ Breaks

An n8n workflow that writes a fresh Marcus Moon talk break every hour of the
overnight shift, voices it with ElevenLabs, and drops it into rotation. Every
fact he says — weather, songs, local events — is fetched and verified by the
workflow *before* the script is written, so nothing is ever hallucinated.

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
time he just closes on the station name, so his name never feels forced. The director:

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
