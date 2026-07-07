# KAZM Marcus Moon — Overnight DJ Breaks

An n8n workflow that writes a fresh Marcus Moon talk break every hour of the
overnight shift, voices it with ElevenLabs, and drops it into AzuraCast
rotation. Every fact he says — weather, songs, local events — is fetched and
verified by the workflow *before* the script is written, so nothing is ever
hallucinated.

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
