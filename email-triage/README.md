# Email Triage — AI Routing to Notion & Calendar

Watches Gmail, classifies each message with Gemini, and routes the result:
action items become Notion tasks, dated commitments become calendar events,
and flight confirmations get both a calendar entry and a travel record.

```
Gmail trigger
  -> extract fields
  -> Gemini classifier (structured output)
  -> parse & validate
  -> switch on type
       todos    -> split -> Notion tasks
       dates    -> split -> Google Calendar
       flights  -> split -> Calendar + Notion flight log
  -> parse errors logged rather than thrown
```

## Setup

1. Create credentials: Gmail, Gemini API key, Notion, Google Calendar
2. Replace `YOUR_TODO_DB_ID` and `YOUR_FLIGHTS_DB_ID` in the Notion nodes
3. Adjust the Gmail trigger filter — it polls the inbox by default, which on a
   busy account means a lot of classifier calls
4. Re-select credentials on each node, then activate

## Notes

- One email can produce several outputs; the split nodes fan a single
  classification into multiple task or event records.
- `Check Parse Errors` catches malformed model output and routes it to a log
  branch instead of failing the execution, so one unparseable email doesn't
  stop the rest of the batch.
- Every Gemini call costs money and the trigger fires per message. Filter the
  trigger before running this on a high-volume inbox.
