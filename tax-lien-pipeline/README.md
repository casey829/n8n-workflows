# Tax Lien Pipeline

Hub-and-spoke n8n system that aggregates delinquent property tax and lien
auctions from five unrelated sources into a single deduplicated Notion
database.

Each source publishes in a different shape — one JSON API, one HTML storefront
behind a two-step ID lookup, one ArcGIS feature service, one authenticated
scrape, one static government page. Every extractor normalizes to a shared
schema and hands off to one ingestion hub, so adding a sixth source means
writing one parser rather than touching the write path.

```
01 US Treasury      ─┐
02 GovDeals         ─┤
03 Bid4Assets       ─┼──►  00 Hub  ──►  standardize ──► dedupe ──┬──► create in Notion
04 GovEase          ─┤                                            └──► update in Notion
05 Pinal ArcGIS     ─┘
```

## Shared schema

Every extractor emits items in this shape. The hub does the rest.

| Field | Notes |
|---|---|
| `property_address` | Falls back to parcel/APN when no street address exists |
| `county`, `state` | `N/A` and `''` are tolerated |
| `source` | Used as the Notion select value |
| `status` | Active / Upcoming / Sold / Closed / Withdrawn |
| `current_bid` | Number; strings get stripped of currency formatting |
| `bidding_date`, `bidding_time`, `sale_date` | Free text — sources disagree on format |
| `auction_url` | Deep link where the source exposes one |
| `unique_id` | Optional; the hub regenerates it as `source\|county\|address` |

## Deduplication

The hub queries the existing Notion database, builds a map keyed on
`unique_id`, then routes each incoming item three ways:

- not in the map → **create**
- in the map but bid or status differs → **update** those fields only
- otherwise → **skip**

`Fetch Existing Entries` uses cursor pagination (`start_cursor` in the body,
stopping when `has_more` is false, capped at 50 pages) and `Deduplicate`
flattens across every page. Both halves matter: Notion caps `page_size` at 100,
so a single unpaginated query would build the map from the first 100 rows only
and silently re-create everything older as new.

## Setup

1. Import `00-hub-notion-ingestion.json` first and note its workflow ID
2. Import the extractors, then set `YOUR_HUB_WORKFLOW_ID` in each `Send to Hub`
   node to the ID from step 1
3. Replace `YOUR_TAX_LIEN_DB_ID` in the hub's `Fetch Existing Entries` and
   `Create in Notion` nodes
4. Create a Notion credential and re-select it on the hub's three Notion nodes
5. Build the Notion database — property names and types are visible in the
   `Create in Notion` node

Extractor-specific setup below. All workflows import inactive.

## 01 — US Treasury

Scrapes the Treasury real property auction index. No auth. Regex-based, so it
breaks when the page layout changes — the parser returns a `text_sample` on
zero matches to make that obvious.

Treasury doesn't publish state, bid or per-listing URLs on the index page, so
those emit as empty/zero and the auction URL points back at the index.

## 02 — GovDeals

Calls the JSON search API behind govdeals.com. The four placeholder header
values (`YOUR_GOVDEALS_SUBSCRIPTION_KEY`, `YOUR_GOVDEALS_API_KEY`,
`YOUR_SESSION_ID`, `YOUR_CORRELATION_ID`) are the site's own front-end keys —
read them from your browser's network tab. They rotate, so expect to refresh
them.

## 03 — Bid4Assets

Two-stage. `County Config` holds a hand-maintained list of county storefront
slugs; each storefront's HTML is parsed for a `StorefrontId` and its collection
IDs, which then drive the auctions API call. Update the slug list monthly from
bid4assets.com/county-tax-sales — expired county sales just stop returning
data.

Filters out withdrawn, sold and closed listings before they reach the hub.

## 04 — GovEase

The only source requiring a login. Flow:

```
Get Login Page → extract CSRF token + session cookie
              → Submit Login (credential-injected)
              → build .ASPXAUTH cookie
              → fetch auctions HTML → parse
```

Credentials never touch the workflow file. Create a **Custom Auth** credential
named `GovEase Login` containing the form field names and values:

```json
{
  "body": {
    "Email": "your-account@example.com",
    "Password": "your-password"
  }
}
```

The node supplies `__RequestVerificationToken` and `RememberMe`; the credential
supplies the account fields.

Both code nodes throw on failure rather than passing empty data downstream — a
missing `.ASPXAUTH` cookie or an undersized HTML response surfaces as a failed
execution instead of a silent zero-row run.

**Check the site's terms of service before running this.** Automated access to
an authenticated area is commonly prohibited, and the account being used is the
one that gets banned.

## 05 — Pinal County ArcGIS

Queries a public ArcGIS MapServer layer. Field names vary between ArcGIS
deployments, so the parser matches attributes by keyword (`parcel`/`apn`/`pin`,
`address`/`situs`/`location`) rather than exact keys. Builds a deep link into
the county assessor where a parcel number is available.

Output is capped at 10 records per run — raise the `slice` in `Parse ArcGIS
Data` if you want the full layer.

## Known limits

- The hub caps pagination at 50 pages (5,000 rows). Past that, raise
  `maxRequests`.
- `unique_id` is derived from source + county + address. Sources that reformat
  an address between runs will produce a second row rather than an update.
- Extractor schedules use n8n's default interval; set explicit cron expressions
  per source before running them all against the same Notion database.
