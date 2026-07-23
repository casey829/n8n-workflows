# Daily Journal — Adaptive Morning & Night Prompts

An n8n workflow that generates journaling prompts adapted to what you've
actually been writing, rather than serving the same static questions daily.

## How it works

Two scheduled runs per day. Each pulls recent history from Notion, sends it to
Gemini as context, and writes a fresh page with prompts generated against that
context — morning prompts reference your goals, night prompts reference what
you wrote that same morning.

```
Morning (05:35)  last 14 morning entries + goals DB
                 -> build context -> Gemini -> parse & validate
                 -> create Notion page

Night   (19:30)  last 14 night entries + today's morning entry
                 -> build context -> Gemini -> parse & validate
                 -> create Notion page
```

## Failure handling

Gemini is treated as unreliable by design. If generation fails or returns
malformed output, validation falls through to a static prompt set so the entry
still gets created, and posts to a configurable alert endpoint flagging that
the fallback fired.

The alert sits on a parallel branch, not in sequence — page creation never
waits on it and never depends on it. A bad API day degrades the prompts; it
never skips the journal.

## Setup

1. Import `daily-journal.json`
2. Create credentials: Notion integration, Gemini API key
3. Replace `YOUR_JOURNAL_DB_ID` and `YOUR_GOALS_DB_ID` in both Global
   Configuration nodes
4. Point `YOUR_ALERT_ENDPOINT` at your alerting channel, or delete both Alert
   nodes if you don't want failure notifications
5. Adjust trigger cron expressions to your timezone
6. Re-select credentials on each node, then activate

## Notion database schema

Open either `Create ... Journal Page` node to see the exact property names and
types before wiring up your own database.

## Notes

- Model is pinned to `gemini-2.5-flash` via direct HTTP call rather than the
  n8n Gemini node — swapping models is a one-line URL change
- Notion caps rich text at 2000 characters per object and 100 blocks per array;
  prompt output stays well under this, but longer content would need chunking
- The night trigger cron is `0 30 19 * * *` (19:30). Change it if you want a
  different time — the node name follows the cron, not the reverse
