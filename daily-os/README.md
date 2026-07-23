# Daily OS — Biometric-Routed Daily Plan

Builds a Notion page each morning that plans the day around recovery data
rather than a fixed template. Workout intensity and the audio for each block
are selected from the previous night's Whoop recovery score and yesterday's
self-reported workout rating.

```
05:00  fetch Whoop recovery
    -> pull yesterday's workout debrief from Notion
    -> Gemini picks workout difficulty + audio profile
    -> route to playlists, start wake-up audio on Spotify
    -> write the day's plan page to Notion
```

## Setup

1. Create credentials: Whoop (OAuth2), Spotify (OAuth2), Notion, Gemini
2. Replace `YOUR_TASKS_DB_ID` in both Notion nodes
3. Replace the eight `YOUR_*_PLAYLIST` placeholders in `Biometric State & Router`
   with your own Spotify playlist IDs
4. Adjust the cron and the block times in the page template to your schedule
5. Re-select credentials on each node, then activate

## Notes

- The Whoop node uses `continueErrorOutput`, so a Whoop outage doesn't kill the
  run — the router falls back to default recovery and strain values and the
  page still gets written. The Gemini prompt reads the score defensively for
  the same reason.
- `Play Spotify Audio` calls the Web API player endpoint, which requires an
  active Spotify device. With no device open it returns 404 — expected
  behaviour, not a broken workflow.
- Timezone is set on the workflow, not per node. Change it in workflow settings
  rather than editing each date expression.
