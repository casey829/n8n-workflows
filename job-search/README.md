# Job Search Automation

Two workflows covering the front and back of a job search: finding and tailoring
applications, then chasing the ones that went quiet.

## 1. Job Scraper & AI Resume Tailoring

Pulls fresh postings from two search APIs each weekday morning, deduplicates
against an existing Notion pipeline, filters for relevance and seniority, then
uses Gemini to produce a tailored summary, cover letter, talking points and
match score per posting.

```
Daily 8AM weekdays
  -> build search terms
  -> SerpApi (Google Jobs) + JSearch (RapidAPI)   [parallel, merged]
  -> flatten & deduplicate
  -> check against existing Notion rows
  -> filter by relevance, seniority, staleness
  -> limit
  -> loop: Gemini tailoring -> sanitize -> save to Notion -> wait
```

### Credentials

| Credential | Type | Configuration |
|---|---|---|
| `SerpApi` | Query Auth | Name `api_key`, value your SerpApi key |
| `RapidAPI JSearch` | Header Auth | Name `X-RapidAPI-Key`, value your RapidAPI key |
| `Gemini API key` | Query Auth | Name `key`, value your Gemini API key |
| `Notion account` | Notion API | Your Notion integration token |

### Configuration

Fill three fields on the `Global Configuration` node:

- `JOB_PIPELINE_DB_ID` — Notion database holding the pipeline
- `CANDIDATE_NAME` — stops the model signing off the cover letter
- `CANDIDATE_RESUME` — plain-text resume the model tailors against

The resume lives in config rather than inside the prompt, so the workflow runs
for any candidate without editing node internals.

Edit `Define Search Terms` for your roles and locations.

### Notes

- Uses HTTP Request rather than the SerpApi community node so it imports with
  no extra installs. The built-in `SerpApi (Google Search)` node is deprecated
  and is an AI-agent sub-node, not usable inline in a scheduled workflow.
- Gemini is called with a `responseSchema`, so output is structured JSON.
  `Sanitize AI Data` still handles unusable responses and substitutes a flagged
  fallback so the run continues.
- Both search APIs are metered and billable. The `Limit` node caps how many
  postings reach the tailoring step per run — raise it deliberately.

## 2. Follow-Up Reminder

Runs weekday mornings against the same Notion database. Finds anything still
marked `Applied` whose follow-up date has passed and flips it to
`Follow-Up Ready`.

Deliberately does not send anything. It queues candidates for a human to review
and send, because an automated follow-up to a hiring manager is worse than no
follow-up.

Set `JOB_PIPELINE_DB_ID` on its `Global Configuration` node to the same
database the scraper writes to. Requires `Status` (select) and `Follow-Up Date`
(date) properties.
