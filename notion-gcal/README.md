# Notion Tasks → Google Calendar Time Blocking

Polls a Notion task database every five minutes and time-blocks anything with a
deadline onto Google Calendar, colour-coded by priority. Tracks the created
event ID back on the Notion page so subsequent runs update the existing event
instead of creating duplicates.

```
Watch Notion (5 min)
  -> process task: skip if done, skip if no deadline
                   parse date, map priority to colour
  -> has GCal event ID?
       yes -> update event
       no  -> create event -> write ID + link back to Notion
```

## Setup

1. Create credentials: Notion, Google Calendar
2. Replace `YOUR_TASKS_DB_ID` on the `Watch Notion Tasks` trigger
3. Re-select credentials on each node, then activate

## Required Notion properties

| Property | Type | Purpose |
|---|---|---|
| `To-do` | title | Event summary |
| `Done` | checkbox | Completed tasks are skipped |
| `Deadline` | date | Event start/end |
| `Priority` | select | Colour coding (High/Urgent, Medium, Low) |
| `GCal Event ID` | rich text | Written back; drives update vs create |
| `GCal Link` | url | Written back for convenience |

## Notes

- Date-only deadlines become a one-hour block at 09:00. Edit the offset in
  `Process Task` if you want different default behaviour — note it's hardcoded
  to a fixed UTC offset rather than the workflow timezone, so it doesn't follow
  daylight saving.
- Completing a task stops it syncing but does not delete the calendar event.
  The skip branch in `Process Task` is where you'd add deletion.
- Five-minute polling on a large database is a lot of Notion API calls. Widen
  the interval if you hit rate limits.
