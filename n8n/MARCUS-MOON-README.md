# KAZM Marcus Moon — DJ Brain v2

A merge of the original "DJ BRAIN TEST" workflow with a redesigned,
accuracy-first pipeline. Same delivery your playout already expects
(Dropbox → MegaSeg), same schedules, same data sources — plus guardrails
that keep every spoken fact verified and every break sounding different
from the last one.

## What it inherits from DJ BRAIN TEST

- **Delivery**: ElevenLabs MP3 overwrites the same Dropbox files MegaSeg
  plays — `/Charles Helstein/Quite Storm Show/qsvo.mp3` (breaks) and
  `Quiet Storm Show Intro V2.mp3` (intro).
- **Schedules**: breaks at 4:20, 4:40, 5:10, 5:30 AM; show intro weekdays
  at 3:59 AM (America/Phoenix).
- **Weather**: NWS `forecast.weather.gov` current observation for Sedona.
- **Coming up**: MegaSeg's own `ComingUp.html` from Google Cloud Storage —
  accurate to the actual playout, with the artist alias cleanup
  ("Hall and Oates", "CSNY", …).
- **Writer**: the OpenAI node (gpt-4.1-mini) with your existing credential.
- Time-of-day tone buckets, exact/descriptive/mixed weather modes, and the
  per-daypart mention probabilities.

## What v2 adds / fixes

1. **Mention-policy bug fix**: the original computed
   `mentionWeather/Song/Artist` flags but never put them in the prompt —
   the model never saw them. v2 rolls the same probabilities and hands the
   model explicit YES/NO flags it must follow.
2. **Facts-only guardrails**: one fact-sheet node gathers everything
   (NWS, ComingUp, AzuraCast recently-played, events board), and the
   prompt's hard rule is "if it's not on the sheet, it doesn't exist."
   Artist trivia is banned outright — vibe talk only.
3. **Anti-repetition memory**: the last 4 break openings (and last 3 intro
   openings) are stored and fed back as "do not start like these." The
   intro prompt also forbids parroting the example's "From the Red Rocks…"
   opening every day.
4. **Recently played**: pulled live from AzuraCast history, so Marcus can
   also react to what just aired (past tense = always accurate).
5. **Script validation**: a cleanup node strips stage directions/markdown,
   rejects AI-tells, and refuses to voice anything too short — a bad
   generation fails loudly instead of airing.
6. **Quiet Storm persona**: overnight/late-night/early-morning buckets get
   the soft, deep, unhurried, flirty-but-never-sleazy register; daytime
   buckets keep the original warm/upbeat Marcus. "Never say stay mellow"
   survives.
7. **Events board** (new): human-curated local happenings via webhook —
   no scraping, no hallucinated events. Auto-expires past dates.
8. **Logbook** (new): last 60 scripts kept; readable via webhook.
9. **Voice settings**: explicit `eleven_multilingual_v2` +
   stability 0.55 / similarity 0.75 / style 0.35 (the original used
   defaults). Tune in the two ElevenLabs nodes.
10. **On-demand test webhook**: generate a break any time of day.

## Import & setup

1. n8n → Workflows → **Import from File/URL** → `marcus-moon-overnight-dj.json`.
2. Reselect credentials (imports never carry them):
   - `DJ Brain for Marcus Moon` and `DJ Brain for the intro` → your
     existing **OpenAI** credential.
   - `Quiet Storm Liner` and `Quiet Storm Intro` → your existing
     **Dropbox OAuth2** credential.
3. Paste your **ElevenLabs key** into the `xi-api-key` header of
   `Marcus finds his voice` and `Marcus voices the intro` — it's the same
   key sitting in the old workflow's "Marcus Moon1" node (deliberately not
   committed to this repo).
4. Activate v2 and **deactivate the old "DJ BRAIN TEST"** so both aren't
   writing `qsvo.mp3`.

## Test it (any time of day)

```
https://n8n.mellowmountainradio.com/webhook/kazm-moon-now?k=moon-2026
```

Runs the full break pipeline once and overwrites `qsvo.mp3` in Dropbox.
Read what he said:

```
https://n8n.mellowmountainradio.com/webhook/kazm-moon-log?k=moon-2026
```

## The events board

```
# add
.../webhook/kazm-moon-events?k=moon-2026&do=add&date=2026-07-12&name=Concert in the Park&place=Posse Grounds&note=starts at 6, free

# list (entries show index i)
.../webhook/kazm-moon-events?k=moon-2026

# remove entry 0
.../webhook/kazm-moon-events?k=moon-2026&do=remove&i=0
```

Events within the next 10 days become eligible; each break has a 40%
chance of mentioning the soonest one (only ever the details you typed).

## Dials worth knowing

- **Schedules**: edit the two schedule-trigger nodes.
- **Mention odds per daypart**: the `policies` table in
  `Build the fact sheet`.
- **Persona / rules**: the `system` string in `Build the fact sheet`
  (breaks) and `Build the intro brief` (intro).
- **Length**: 70–90 words ≈ 30 seconds; in the same system strings.
- **Webhook key**: `moon-2026`, used by the test trigger, events board,
  and logbook (`Check the key`, `Tend the board`, `Read the logbook`).
- **Switching the writer to Claude later**: replace the two OpenAI nodes
  with an HTTP Request node POSTing to `https://api.anthropic.com/v1/messages`
  (model `claude-opus-4-8`, headers `x-api-key` + `anthropic-version:
  2023-06-01`) — the fact-sheet output (`system`/`user`) is already
  provider-agnostic.
