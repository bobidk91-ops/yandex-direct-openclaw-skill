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
2. Pick a `method` (string, e.g. `add`, `update`, `get`, `delete`, `suspend`, `resume`, `moderate`, etc.)
3. Send `POST` JSON body `{ "method": "...", "params": { ... } }`

This skill supports **any** call supported by the API by passing through `service`, `method`, and `params` dynamically.

## Safety requirements

- Never print or echo `YANDEX_DIRECT_TOKEN` (or any token/private secrets) into logs, tool output, or final answers.
- Only call Yandex Direct JSON endpoints on `api.direct.yandex.com` (production) or `api-sandbox.direct.yandex.ru` (sandbox).
- Validate `service` and `method` before executing (reject anything outside `/^[a-z0-9_]+$/i`).
- If the user didn't provide `params` (or didn't provide enough info to build them), ask clarifying questions before calling the API.

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
- `Accept-Language: ru` (override with `YANDEX_DIRECT_ACCEPT_LANGUAGE` env var)
- `Content-Type: application/json; charset=utf-8`

## Request format

Default (most services):

```json
{
  "method": "<methodName>",
  "params": { }
}
```

Reports service (`service=reports`) differs — body has only `params`, **no `method` field**:

```json
{
  "params": {
    "SelectionCriteria": { "DateFrom": "2026-01-01", "DateTo": "2026-01-07", "Filter": [] },
    "FieldNames": ["Date", "CampaignName", "Impressions", "Clicks", "Cost"],
    "ReportName": "my_report",
    "ReportType": "CAMPAIGN_PERFORMANCE_REPORT",
    "DateRangeType": "CUSTOM_DATE",
    "Format": "TSV",
    "IncludeVAT": "YES",
    "IncludeDiscount": "NO"
  }
}
```

## Reports: SelectionCriteria and Filter

**Important:** `SelectionCriteria` for reports accepts only `DateFrom`, `DateTo`, and `Filter` array.
`CampaignIds`, `AdGroupIds`, etc. are **NOT** direct fields of `SelectionCriteria` — filter by them using the `Filter` array:

```json
"SelectionCriteria": {
  "DateFrom": "2026-03-23",
  "DateTo":   "2026-03-29",
  "Filter": [
    { "Field": "CampaignId", "Operator": "IN", "Values": ["706701050"] }
  ]
}
```

Available `Filter.Operator` values: `EQUALS`, `NOT_EQUALS`, `IN`, `NOT_IN`, `LESS_THAN`, `GREATER_THAN`.

## Report types and valid FieldNames

Use the correct `ReportType` for the fields you need:

| ReportType | Key fields available |
|---|---|
| `CAMPAIGN_PERFORMANCE_REPORT` | `Date`, `CampaignId`, `CampaignName`, `Impressions`, `Clicks`, `Ctr`, `AvgCpc`, `Cost` |
| `ADGROUP_PERFORMANCE_REPORT` | `Date`, `AdGroupId`, `AdGroupName`, `CampaignName`, `Impressions`, `Clicks`, `Ctr`, `AvgCpc`, `Cost` |
| `AD_PERFORMANCE_REPORT` | `Date`, `AdId`, `AdGroupName`, `CampaignName`, `Headline`, `Impressions`, `Clicks`, `Ctr`, `AvgCpc`, `Cost` |
| `CRITERIA_PERFORMANCE_REPORT` | `Date`, `CampaignName`, `AdGroupName`, `Criterion`, `CriterionType`, `Impressions`, `Clicks`, `Ctr`, `AvgCpc`, `Cost` |
| `SEARCH_QUERY_PERFORMANCE_REPORT` | `Date`, `CampaignName`, `AdGroupName`, `Query`, `Impressions`, `Clicks`, `Ctr`, `AvgCpc`, `Cost` |
| `ACCOUNT_PERFORMANCE_REPORT` | `Date`, `Impressions`, `Clicks`, `Ctr`, `AvgCpc`, `Cost` |

**Do not mix fields from different report types** — Yandex will return error 4000.

For **keyword-level** stats use `CRITERIA_PERFORMANCE_REPORT` with field `Criterion` (not `Keyword`).
For **search query** stats use `SEARCH_QUERY_PERFORMANCE_REPORT` with field `Query` (not `Keyword`).

## How the agent should execute (Windows-first)

If `exec` is allowed, call the API using PowerShell. Use the templates below (fill in placeholders).

### Template A — regular services (non-reports)

```powershell
$token  = $env:YANDEX_DIRECT_TOKEN
$login  = $env:YANDEX_DIRECT_LOGIN
$lang   = if ($env:YANDEX_DIRECT_ACCEPT_LANGUAGE) { $env:YANDEX_DIRECT_ACCEPT_LANGUAGE } else { "ru" }
$host   = if ($env:YANDEX_DIRECT_SANDBOX -eq "true") { "api-sandbox.direct.yandex.ru" } else { "api.direct.yandex.com" }
$useV501 = $false   # set $true for unified performance campaigns
$service = "<service>"   # e.g. campaigns, keywords, adgroups, ads, bids
$method  = "<method>"    # e.g. get, add, update, delete, suspend, resume

$uri = "https://$host/json/$(if ($useV501) { 'v501' } else { 'v5' })/$service"

$bodyObj   = @{ method = $method; params = <PARAMS_AS_POWERSHELL_HASHTABLE> }
$bodyBytes = [System.Text.Encoding]::UTF8.GetBytes(($bodyObj | ConvertTo-Json -Depth 80 -Compress))

$req = [System.Net.HttpWebRequest]::Create($uri)
$req.Method = "POST"
$req.ContentType = "application/json; charset=utf-8"
$req.ContentLength = $bodyBytes.Length
$req.Headers["Authorization"]   = "Bearer $token"
$req.Headers["Client-Login"]    = $login
$req.Headers["Accept-Language"] = $lang

$s = $req.GetRequestStream()
$s.Write($bodyBytes, 0, $bodyBytes.Length)
$s.Close()

try {
  $resp   = $req.GetResponse()
  $reader = New-Object System.IO.StreamReader($resp.GetResponseStream(), [System.Text.Encoding]::UTF8)
  $result = $reader.ReadToEnd()
  $reader.Close()
  $result | ConvertFrom-Json
} catch [System.Net.WebException] {
  $errReader = New-Object System.IO.StreamReader($_.Exception.Response.GetResponseStream(), [System.Text.Encoding]::UTF8)
  $errBody   = $errReader.ReadToEnd()
  $errReader.Close()
  Write-Error "HTTP $([int]$_.Exception.Response.StatusCode): $errBody"
}
```

### Template B — reports service (with polling for offline mode)

```powershell
$token  = $env:YANDEX_DIRECT_TOKEN
$login  = $env:YANDEX_DIRECT_LOGIN
$lang   = if ($env:YANDEX_DIRECT_ACCEPT_LANGUAGE) { $env:YANDEX_DIRECT_ACCEPT_LANGUAGE } else { "ru" }
$host   = if ($env:YANDEX_DIRECT_SANDBOX -eq "true") { "api-sandbox.direct.yandex.ru" } else { "api.direct.yandex.com" }
$uri    = "https://$host/json/v5/reports"

$bodyObj = @{
  params = @{
    SelectionCriteria = @{
      DateFrom = "<YYYY-MM-DD>"
      DateTo   = "<YYYY-MM-DD>"
      Filter   = @(
        @{ Field = "CampaignId"; Operator = "IN"; Values = @("<campaign_id>") }
      )
    }
    FieldNames    = @("Date", "CampaignName", "Criterion", "Impressions", "Clicks", "Ctr", "AvgCpc", "Cost")
    ReportName    = "<unique_report_name>"
    ReportType    = "CRITERIA_PERFORMANCE_REPORT"
    DateRangeType = "CUSTOM_DATE"
    Format        = "TSV"
    IncludeVAT    = "YES"
    IncludeDiscount = "NO"
  }
}

$bodyBytes = [System.Text.Encoding]::UTF8.GetBytes(($bodyObj | ConvertTo-Json -Depth 80 -Compress))

$maxRetries = 15
$attempt    = 0

do {
  $attempt++
  $req = [System.Net.HttpWebRequest]::Create($uri)
  $req.Method = "POST"
  $req.ContentType = "application/json; charset=utf-8"
  $req.ContentLength = $bodyBytes.Length
  $req.Headers["Authorization"]   = "Bearer $token"
  $req.Headers["Client-Login"]    = $login
  $req.Headers["Accept-Language"] = $lang

  $s = $req.GetRequestStream()
  $s.Write($bodyBytes, 0, $bodyBytes.Length)   # NOTE: third arg must be $bodyBytes.Length, not 0
  $s.Close()

  try {
    $resp   = $req.GetResponse()
    $reader = New-Object System.IO.StreamReader($resp.GetResponseStream(), [System.Text.Encoding]::UTF8)
    $tsv    = $reader.ReadToEnd()
    $reader.Close()
    # Save to file (avoids terminal encoding issues with Cyrillic)
    [System.IO.File]::WriteAllText("$env:TEMP\direct_report.tsv", $tsv, [System.Text.Encoding]::UTF8)
    Write-Host "Report ready. Saved to $env:TEMP\direct_report.tsv"
    $tsv
    break
  } catch [System.Net.WebException] {
    $code = [int]$_.Exception.Response.StatusCode
    if ($code -in 201, 202) {
      $retryIn = $_.Exception.Response.Headers["retryIn"]
      $wait    = if ($retryIn) { [int]$retryIn } else { 10 }
      Write-Host "HTTP $code — report queued. Attempt $attempt/$maxRetries, waiting ${wait}s..."
      Start-Sleep -Seconds $wait
    } else {
      $errReader = New-Object System.IO.StreamReader($_.Exception.Response.GetResponseStream(), [System.Text.Encoding]::UTF8)
      $errBody   = $errReader.ReadToEnd()
      $errReader.Close()
      Write-Error "HTTP $code`: $errBody"
      break
    }
  }
} while ($attempt -lt $maxRetries)
```

## Natural-language to API dispatch

When the user request is not "API-first", infer the most likely `service` and `method`:

Known services: `campaigns`, `adgroups`, `ads`, `keywords`, `keywordbids`, `bids`, `bidmodifiers`,
`reports`, `clients`, `agencyclients`, `dictionaries`, `retargetinglists`, `sitelinks`, `vcards`,
`adimages`, `adextensions`.

| User says | method |
|---|---|
| создать / добавить | `add` |
| изменить / обновить | `update` |
| удалить / убрать | `delete` |
| получить / выгрузить / список | `get` |
| остановить / приостановить | `suspend` |
| запустить / возобновить | `resume` |
| на модерацию | `moderate` |
| статистика / отчёт / расход | `reports` service (Template B) |
| ставки по ключевым словам | `keywords` service, method `get`, FieldNames includes `Bid`, `ContextBid` |

If there is ambiguity, ask the user:

- which object type (campaigns/ads/keywords/bids/reports/etc.)
- whether they need v5 or v501
- required IDs (campaignId, adGroupId, etc.) and desired FieldNames/filters

## Required inputs (before calling)

At minimum, the agent must determine:

- `service`
- `method` (empty string for reports)
- `params` (as a PowerShell hashtable / JSON object)

## Output handling

After the tool call returns:

1. If the response contains Direct errors, present them: error code + message + what to change.
2. Otherwise, summarize what was created/updated/retrieved with the relevant identifiers.
3. For `service=reports`: save TSV to a temp file with UTF-8 encoding, then read and display.

## Cost values

All monetary amounts in the API (`Cost`, `Bid`, `ContextBid`, etc.) are in **microroubles**.
Divide by `1 000 000` to get roubles.

## Links

- [Yandex Direct API v5 overview](https://yandex.ru/dev/direct/doc/en/concepts/overview)
- [Campaigns service](https://yandex.ru/dev/direct/doc/en/campaigns/campaigns)
- [Reports service](https://yandex.ru/dev/direct/doc/en/reports/reports)
- [Report field reference](https://yandex.ru/dev/direct/doc/en/reports/fields-list)
- [Ads.get reference](https://yandex.ru/dev/direct/doc/en/ads/get)
