# n8n Workflows

Twelve n8n workflows across six projects, built for a single Upwork client —
a personal automation suite plus a tax lien aggregation pipeline — and
published with their permission. Every workflow here ran against live systems
— credentials and client-identifying data have been replaced with
placeholders, and all authentication routes through n8n's credential store
rather than the workflow file.

| Project | Flows | What it does | Integrations |
|---|---|---|---|
| [tax-lien-pipeline](./tax-lien-pipeline) | 6 | Hub-and-spoke aggregation of delinquent tax and lien auctions from five unrelated sources into one deduplicated database | Notion, ArcGIS, custom HTML/JSON parsers, authenticated session scraping |
| [job-search](./job-search) | 2 | Daily scraping across two search APIs, then a cover letter, talking points and match score generated per posting against your resume, plus a follow-up reminder queue | SerpApi, JSearch/RapidAPI, Gemini, Notion |
| [daily-journal](./daily-journal) | 1 | Morning and night journaling prompts generated from your own recent entries and goals rather than a static list | Notion, Gemini |
| [daily-os](./daily-os) | 1 | Builds a daily plan page routed by biometric recovery data, selecting workout intensity and audio for each block of the day | Whoop, Spotify, Notion, Gemini |
| [email-triage](./email-triage) | 1 | Classifies incoming mail and routes it to tasks, calendar events or flight records automatically | Gmail, Gemini, Notion, Google Calendar |
| [notion-gcal](./notion-gcal) | 1 | Time-blocks Notion tasks onto Google Calendar with priority colour coding, writing the event ID back to Notion so re-runs update instead of duplicating | Notion, Google Calendar |

## Patterns worth noting

**Credentials never live in the workflow.** Every API key, token and password
routes through n8n credentials — predefined types where they exist, generic
Header/Query/Custom Auth where they don't. Importing any file here and
exporting it again produces nothing sensitive.

**The unhappy path is handled.** The journal workflow degrades to a static
prompt set when the model fails rather than skipping the entry. The job
scraper substitutes a flagged fallback when Gemini returns unparseable output.
The tax lien parsers return a diagnostic row on zero matches — a text sample,
the response keys they actually saw — rather than an empty array that looks
like a clean run. GovEase throws instead, because a missing auth cookie means
the session died, not that there were no auctions.

**Pagination is treated as correctness, not optimization.** The tax lien hub
walks every page of its Notion query, because a deduplication map built from
only the first 100 rows silently re-creates everything older as new.

**Shared schema over point-to-point wiring.** The five tax lien extractors all
normalize to one payload shape, so the ingestion, deduplication and write logic
exists once. Adding a sixth source means writing a parser, not touching the
write path.

## Using these

Each folder has its own README with setup steps, required credentials and the
placeholder values to replace. All workflows import inactive — fill in
configuration and credentials before activating.

Workflows are exported from n8n and import via **Workflows → Import from File**.
Imported workflows carry credential *names* but not bindings, so re-select the
credential on each node after importing.

## Status

Delivered. All twelve workflows were built on an Upwork engagement and are
published with the client's permission, sanitized for public release. There is
nothing to deploy or visit — these are workflow exports, and the import steps
are above.

## License

All rights reserved — see [LICENSE](LICENSE). Published as a portfolio sample
with the client's permission; not licensed for reuse.
