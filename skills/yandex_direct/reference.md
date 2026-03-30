# Reference: Yandex Direct API wrapper (v5 / v501)

Compact "how to call" reference for the `yandex_direct` OpenClaw skill.
All requests use the same wrapper: `service` + `method` + `params`
(with one exception — `reports` uses a different body format).

## Base URLs (JSON)

- v5: `https://api.direct.yandex.com/json/v5/{service}`
- v501: `https://api.direct.yandex.com/json/v501/{service}`
- sandbox v5: `https://api-sandbox.direct.yandex.ru/json/v5/{service}`
- sandbox v501: `https://api-sandbox.direct.yandex.ru/json/v501/{service}`

## Required headers (every request)

| Header | Value |
|---|---|
| `Authorization` | `Bearer <YANDEX_DIRECT_TOKEN>` |
| `Client-Login` | `<YANDEX_DIRECT_LOGIN>` |
| `Accept-Language` | `ru` (or `YANDEX_DIRECT_ACCEPT_LANGUAGE` env var) |
| `Content-Type` | `application/json; charset=utf-8` |

Never put the token into prompts or logs.

## Default wrapper (most services)

```json
{
  "method": "<methodName>",
  "params": { }
}
```

Example: `campaigns.get`

```json
{
  "method": "get",
  "params": {
    "SelectionCriteria": {},
    "FieldNames": ["Id", "Name", "Status", "State"]
  }
}
```

## Keywords with bids: `keywords.get`

```json
{
  "method": "get",
  "params": {
    "SelectionCriteria": { "CampaignIds": [706701050] },
    "FieldNames": ["Id", "Keyword", "Bid", "ContextBid", "Status", "State", "AdGroupId"]
  }
}
```

`Bid` and `ContextBid` are in **microroubles** — divide by 1 000 000 to get roubles.

## Reports service (special case)

Body has **no `method` field** — only `params`:

```json
{
  "params": {
    "SelectionCriteria": {
      "DateFrom": "2026-03-23",
      "DateTo":   "2026-03-29",
      "Filter": [
        { "Field": "CampaignId", "Operator": "IN", "Values": ["706701050"] }
      ]
    },
    "FieldNames": ["Date", "CampaignName", "Criterion", "Impressions", "Clicks", "Ctr", "AvgCpc", "Cost"],
    "ReportName": "my_unique_report_name",
    "ReportType": "CRITERIA_PERFORMANCE_REPORT",
    "DateRangeType": "CUSTOM_DATE",
    "Format": "TSV",
    "IncludeVAT": "YES",
    "IncludeDiscount": "NO"
  }
}
```

### SelectionCriteria for reports — important rules

`SelectionCriteria` accepts only these top-level fields: `DateFrom`, `DateTo`, `Filter`.

`CampaignIds` / `AdGroupIds` / etc. are **NOT** direct fields — use the `Filter` array:

```json
"Filter": [
  { "Field": "CampaignId", "Operator": "IN", "Values": ["123456"] }
]
```

Available filter operators: `EQUALS`, `NOT_EQUALS`, `IN`, `NOT_IN`, `LESS_THAN`, `GREATER_THAN`.

### Report types and compatible FieldNames

| ReportType | Key FieldNames |
|---|---|
| `CAMPAIGN_PERFORMANCE_REPORT` | `Date`, `CampaignId`, `CampaignName`, `Impressions`, `Clicks`, `Ctr`, `AvgCpc`, `Cost` |
| `ADGROUP_PERFORMANCE_REPORT` | `Date`, `AdGroupId`, `AdGroupName`, `CampaignName`, `Impressions`, `Clicks`, `Ctr`, `AvgCpc`, `Cost` |
| `AD_PERFORMANCE_REPORT` | `Date`, `AdId`, `AdGroupName`, `CampaignName`, `Headline`, `Impressions`, `Clicks`, `Ctr`, `AvgCpc`, `Cost` |
| `CRITERIA_PERFORMANCE_REPORT` | `Date`, `CampaignName`, `AdGroupName`, `Criterion`, `CriterionType`, `Impressions`, `Clicks`, `Ctr`, `AvgCpc`, `Cost` |
| `SEARCH_QUERY_PERFORMANCE_REPORT` | `Date`, `CampaignName`, `AdGroupName`, `Query`, `Impressions`, `Clicks`, `Ctr`, `AvgCpc`, `Cost` |
| `ACCOUNT_PERFORMANCE_REPORT` | `Date`, `Impressions`, `Clicks`, `Ctr`, `AvgCpc`, `Cost` |

Do **not** mix fields from different report types — Yandex returns error 4000.

Use `CRITERIA_PERFORMANCE_REPORT` + `Criterion` for **keyword-level** stats (not `Keyword`).
Use `SEARCH_QUERY_PERFORMANCE_REPORT` + `Query` for **search query** stats (not `Keyword`).

### Online / offline report behavior

| HTTP status | Meaning | Action |
|---|---|---|
| `200` | Report ready | Response body is TSV |
| `201` | Report queued (first time) | Repeat same request after `retryIn` seconds |
| `202` | Report in progress | Repeat same request after `retryIn` seconds |
| `400` | Bad request | Fix params and try again |

## Common method names

| Method | Description |
|---|---|
| `add` | Create objects |
| `update` | Modify objects |
| `delete` | Delete objects |
| `get` | Retrieve parameters |
| `suspend` | Pause |
| `resume` | Resume |
| `archive` | Archive (campaigns) |
| `unarchive` | Unarchive (campaigns) |
| `moderate` | Send ads for review |

## PowerShell execution notes

- Use `[System.Net.HttpWebRequest]` — **not** `Invoke-RestMethod`, which garbles UTF-8 and throws on 201/202.
- Always write `$s.Write($bodyBytes, 0, $bodyBytes.Length)` — **not** `0` as the third argument.
- To avoid Cyrillic garbling in terminal, save response to file with `[System.IO.File]::WriteAllText(path, content, UTF8)` and read back.
- All monetary values (`Cost`, `Bid`, `AvgCpc`, etc.) are in **microroubles** — divide by 1 000 000.

## Error handling

If the API returns an error:

1. Show the Direct error code(s) and message(s) — without leaking the token.
2. Identify the missing/invalid parts of `params`.
3. Ask the user for the missing fields or propose corrected JSON.

Common errors:

| Code | Meaning |
|---|---|
| 52 | Missing or invalid token |
| 58 | API access not confirmed (pending application approval) |
| 4000 | Invalid request params (wrong field names, wrong values) |
| 8000 | Unknown or unsupported field for this SelectionCriteria / ReportType |
