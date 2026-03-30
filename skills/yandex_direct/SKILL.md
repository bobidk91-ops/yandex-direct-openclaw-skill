---
name: yandex_direct
description: Call Yandex Direct API v5/v501 JSON methods (add/update/get/delete/suspend/resume/moderate/reports) using an OAuth access token. Use when the user wants to manage campaigns, ads, keywords, bids, or generate reports via Yandex Direct.
metadata: {"openclaw":{"requires":{"env":["YANDEX_DIRECT_TOKEN","YANDEX_DIRECT_LOGIN"]},"primaryEnv":"YANDEX_DIRECT_TOKEN","emoji":"🎯"}}
homepage: https://yandex.ru/dev/direct/doc/en/concepts/overview
---

# Yandex Direct (API v5 / v501)

## Core idea (covers "all possible calls")

Yandex Direct method calls all share the same HTTP wrapper:

1. Pick a `service` (the path segment in the URL)
2. Pick a `method` (string, e.g. `add`, `update`, `get`, `delete`, `suspend`, `resume`, `moderate`, `generateReport`, etc.)
3. Send `POST` JSON body `{ "method": "...", "params": { ... } }`

So this skill does not hardcode every single method name. It supports **any** call supported by the API by letting the agent pass through `service`, `method`, and `params` correctly (with safety checks below).

## Safety requirements (important)

- Never print or echo `YANDEX_DIRECT_TOKEN` (or any token/private secrets) into logs, tool output, or final answers.
- Only call Yandex Direct JSON endpoints on `api.direct.yandex.com` (production) or `api-sandbox.direct.yandex.ru` (sandbox).
- Validate `service` and `method` before executing (reject anything outside `/^[a-z0-9_]+$/i`).
- If the user didn’t provide `params` (or didn’t provide enough info to build them), ask clarifying questions before calling the API.

## Endpoints

Use one of these base URLs (JSON mode):

- v5 (most services): `https://api.direct.yandex.com/json/v5/{service}`
- v501 (unified performance campaigns when required by Yandex): `https://api.direct.yandex.com/json/v501/{service}`
- sandbox v5: `https://api-sandbox.direct.yandex.ru/json/v5/{service}`
- sandbox v501: `https://api-sandbox.direct.yandex.ru/json/v501/{service}`

## Authorization

The OAuth access token from `YANDEX_DIRECT_TOKEN` must be sent as:

- `Authorization: Bearer <token>`

Yandex Direct also requires these headers on every request:

- `Client-Login: <YANDEX_DIRECT_LOGIN>`
- `Accept-Language: ru` (override with `YANDEX_DIRECT_ACCEPT_LANGUAGE`)
- `Content-Type: application/json; charset=utf-8`

## Request format

Default (most services) send:

```json
{
  "method": "<methodName>",
  "params": { /* method-specific params */ }
}
```

Reports service (`service=reports`) differs: the JSON body is the report `params`
only (no `method` field). The `params` structure follows the “ReportDefinition”
schema (SelectionCriteria, FieldNames, ReportType, DateRangeType, etc.).

## How the agent should execute (Windows-first)

If `exec` is allowed, the agent should call the API using PowerShell (`pwsh`) + `Invoke-RestMethod`.

Execution template (agent should fill in placeholders):

```powershell
$service = "<service>"; # validated
$method  = "<method>";  # validated
$directHost = "api.direct.yandex.com";
if ($env:YANDEX_DIRECT_SANDBOX -eq "true") { $directHost = "api-sandbox.direct.yandex.ru" }
$baseUrl = "https://$directHost";
$useV501  = <true|false>; # set by agent based on user intent or Yandex guidance
$uri = "$baseUrl/json/$(if ($useV501) { 'v501' } else { 'v5' })/$service"

$paramsObject = <PARAMS_AS_POWERSHELL_OBJECT>;
$bodyObject =
    if ($service -eq "reports") { @{ params = $paramsObject } }
    else { @{ method = $method; params = $paramsObject } }
$bodyJson = ($bodyObject | ConvertTo-Json -Depth 80)

$headers = @{
  Authorization = "Bearer $env:YANDEX_DIRECT_TOKEN"
  "Client-Login" = $env:YANDEX_DIRECT_LOGIN
  "Accept-Language" = (if ($env:YANDEX_DIRECT_ACCEPT_LANGUAGE) { $env:YANDEX_DIRECT_ACCEPT_LANGUAGE } else { "ru" })
  "Content-Type" = "application/json; charset=utf-8"
}

Invoke-RestMethod -Method Post -Uri $uri -Headers $headers -ContentType "application/json" -Body $bodyJson
```

## Natural-language to API dispatch (lightweight routing)

When the user request is not “API-first”, infer the most likely `service` and `method`:

- Known Yandex Direct services (examples): `campaigns`, `adgroups`, `ads`, `keywords`, `keywordbids`, `bids`, `bidmodifiers`, `reports`, `clients`, `agencyclients`, `dictionaries`, `retargetinglists`, `sitelinks`, `vcards`, `adimages`, `adextensions`.
- “создать / добавить” -> `add`
- “изменить / обновить” -> `update`
- “удалить / убрать” -> `delete`
- “получить / выгрузить / список / получить параметры” -> `get`
- “остановить / приостановить” -> `suspend`
- “запустить / возобновить” -> `resume`
- “на модерацию / отправить на проверку” (обычно для объявлений) -> `moderate`
- “сгенерировать отчет / statistics / performance report” -> report generation method (e.g. `generateReport`), using the Yandex reports schema for `params`

If there is ambiguity (multiple services match the same wording, or a nonstandard method is mentioned), ask the user:

- which object type they mean (campaigns/ads/keywords/bids/reports/etc.)
- whether they need v5 or v501
- and to provide required IDs (campaignId, adGroupId, etc.) and the desired `FieldNames`/filters

## Required inputs from the agent (before calling)

At minimum, the agent must determine:

- `service`
- `method`
- `params` (as a JSON object)

## Output handling

After the tool call returns:

1. If the response contains Direct errors, present them as a concise list: error code + message + what the agent should change (e.g., missing required fields).
2. Otherwise, summarize what was created/updated/retrieved and include the identifiers the user will need for follow-up calls.
3. For `service=reports`: handle online/offline.
   - If HTTP status is `200`, response body contains TSV.
   - If HTTP status is `201` or `202`, the report is queued/in progress; repeat the exact same request later (wait based on `retryIn` header when present).

## Links (for correctness of params)

- [Yandex Direct API v5 overview](https://yandex.ru/dev/direct/doc/en/concepts/overview)
- [Campaigns service methods list](https://yandex.ru/dev/direct/doc/en/campaigns/campaigns)
- [Ads.get reference (example of `params`/`result` shape)](https://yandex.ru/dev/direct/doc/en/ads/get)
