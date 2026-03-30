# Reference: Yandex Direct API wrapper (v5 / v501)

This file is a compact “how to call” reference for the `yandex_direct` OpenClaw skill.
It intentionally avoids listing every single Yandex Direct method in the API
because all requests can be executed through the same wrapper:

`service` + `method` + `params` (with one exception: `reports` JSON bodies).

## Base URLs (JSON)

- v5: `https://api.direct.yandex.com/json/v5/{service}`
- v501: `https://api.direct.yandex.com/json/v501/{service}`
- sandbox v5: `https://api-sandbox.direct.yandex.ru/json/v5/{service}`
- sandbox v501: `https://api-sandbox.direct.yandex.ru/json/v501/{service}`

## Authorization

Use the OAuth access token in header for every request:

- `Authorization: Bearer <token>`

In this skill, the token is expected in environment variable:

- `YANDEX_DIRECT_TOKEN`

Additionally, Yandex Direct requires these headers on every request:

- `Client-Login: <YANDEX_DIRECT_LOGIN>`
- `Accept-Language: ru` (override with `YANDEX_DIRECT_ACCEPT_LANGUAGE`)
- `Content-Type: application/json; charset=utf-8`

Never put the token into prompts or logs.

## Default wrapper (most services)

For most services, use this JSON body:

```json
{
  "method": "<methodName>",
  "params": { /* method-specific parameters */ }
}
```

Example (simplified): `ads.get`

```json
{
  "method": "get",
  "params": {
    "SelectionCriteria": { "Ids": [123456] },
    "FieldNames": ["Id", "CampaignId", "AdGroupId", "Type", "TextAd"]
  }
}
```

## Reports service (special case)

For `service=reports`, the JSON request body is the report `params` only:

```json
{
  "params": {
    "SelectionCriteria": { /* DateFrom/DateTo + filters */ },
    "FieldNames": ["AdGroupId", "Year" /* ... */],
    "ReportName": "my_report_name",
    "ReportType": "ACCOUNT_PERFORMANCE_REPORT",
    "DateRangeType": "ALL_TIME",
    "Format": "TSV",
    "IncludeVAT": "NO",
    "IncludeDiscount": "NO"
  }
}
```

Report generation is described by the “report specification” schema
(SelectionCriteria, Filter, FieldNames, Page, OrderBy, etc.).

Online/offline behavior:

- HTTP `200`: report is returned immediately in the response body (TSV).
- HTTP `201` / `202`: report is queued/in progress; repeat the exact same request later.
  - If the response includes `retryIn` header, wait based on it.

## Common method names (cross-service)

Yandex Direct services usually follow a set of standard lifecycle operations:

- `add`: create objects
- `update`: modify objects
- `delete`: delete objects
- `get`: retrieve object parameters
- `suspend`: pause
- `resume`: restart

Some services add extra methods:

- `archive` / `unarchive` (common for campaigns)
- `moderate` (commonly for ads review)
- report generation typically uses the `reports` service wrapper above (body differs)

## What the agent must determine

Before executing an API call, the agent needs:

1. `service` (URL path segment), e.g. `campaigns`, `ads`, `adgroups`, `keywords`, `reports`, etc.
2. `method` (string), when applicable (not used for `reports`)
3. `params` (JSON object) containing required keys for that method

If the user doesn’t provide enough data to build `params`, ask follow-up questions.

## Error handling expectation

If the API returns an error, the agent should:

1. Show the Direct error code(s) and message(s) (without leaking token)
2. Identify the missing/invalid parts of `params`
3. Ask the user for the missing fields or propose corrected JSON
