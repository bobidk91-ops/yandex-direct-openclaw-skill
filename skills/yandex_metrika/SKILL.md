---
name: yandex_metrika
description: Call Yandex Metrica API — stat reports (visits, pageviews, goals, UTM, sources, geo, devices), management (counters, goals CRUD), Logs API (raw data export). Use when the user wants analytics, traffic data, conversions, or counter management via Yandex Metrica.
metadata: {"openclaw":{"requires":{"env":["YANDEX_METRIKA_TOKEN"]},"primaryEnv":"YANDEX_METRIKA_TOKEN","emoji":"📊"}}
homepage: https://yandex.ru/dev/metrika
---

# Яндекс.Метрика (API stat / management / Logs)

## Ключевые отличия от Яндекс.Директа

| | Яндекс.Директ | Яндекс.Метрика |
|---|---|---|
| Хост | `api.direct.yandex.com` | `api-metrika.yandex.net` |
| Авторизация | `Authorization: Bearer <token>` | `Authorization: OAuth <token>` |
| Метод запросов | POST + JSON body | **GET** + query params (stat) / POST (management) |
| Токен env | `YANDEX_DIRECT_TOKEN` | `YANDEX_METRIKA_TOKEN` |

**Важно:** слово в заголовке — `OAuth`, **не** `Bearer`.

## Safety requirements

- Никогда не выводить `YANDEX_METRIKA_TOKEN` в логи, ответы, файлы.
- Только хост `api-metrika.yandex.net`.
- Имена `metrics`/`dimensions` — только из официального справочника, не придумывать.

## Базовый URL

```
https://api-metrika.yandex.net
```

## Авторизация

Каждый запрос несёт заголовок:

```
Authorization: OAuth <YANDEX_METRIKA_TOKEN>
```

## Разделы API

| Задача | Endpoint | Метод |
|---|---|---|
| Отчёт (визиты, хиты, цели) | `/stat/v1/data` | GET |
| Динамика по времени | `/stat/v1/data/bytime` | GET |
| Сравнение сегментов | `/stat/v1/data/comparison` | GET |
| Drilldown (дерево) | `/stat/v1/data/drilldown` | GET |
| Список счётчиков | `/management/v1/counters` | GET |
| Цели счётчика | `/management/v1/counter/{id}/goals` | GET / POST |
| Параметры счётчика | `/management/v1/counter/{id}` | GET |
| Logs API — создать задачу | `/management/v1/counter/{id}/logrequests` | POST |
| Logs API — статус задачи | `/management/v1/counter/{id}/logrequest/{reqId}` | GET |
| Logs API — скачать | `/management/v1/counter/{id}/logrequest/{reqId}/part/{partN}/download` | GET |
| Logs API — удалить | `/management/v1/counter/{id}/logrequest/{reqId}/clean` | POST |

## Правила префиксов metrics/dimensions

В одном запросе `/stat/v1/data` **нельзя** смешивать разные префиксы в `metrics`/`dimensions`:

| Префикс | Что считает | Пример metric |
|---|---|---|
| `ym:s:` | Визиты (sessions) | `ym:s:visits`, `ym:s:users`, `ym:s:bounceRate` |
| `ym:pv:` | Хиты (pageviews) | `ym:pv:pageviews`, `ym:pv:URL` |

В параметре `filters` другой префикс допустим.

## Как агент должен выполнять запросы (Windows PowerShell)

### Template A — GET-запрос к stat

```powershell
$token  = $env:YANDEX_METRIKA_TOKEN
$cid    = "<COUNTER_ID>"   # из env YANDEX_METRIKA_COUNTER_ID или спросить у пользователя

$params = [ordered]@{
  ids         = $cid
  metrics     = "ym:s:visits,ym:s:users"
  dimensions  = "ym:s:date"
  date1       = "7daysAgo"
  date2       = "yesterday"
  sort        = "ym:s:date"
  limit       = 100
  lang        = "ru"
}

$query = ($params.GetEnumerator() | ForEach-Object {
  "$([Uri]::EscapeDataString($_.Key))=$([Uri]::EscapeDataString($_.Value))"
}) -join "&"

$uri = "https://api-metrika.yandex.net/stat/v1/data?$query"

$result = Invoke-RestMethod -Uri $uri `
  -Headers @{ Authorization = "OAuth $token" } `
  -Method Get

# Сохранить для чтения с правильной кодировкой
$result | ConvertTo-Json -Depth 10 | Out-File "$env:TEMP\metrika_result.json" -Encoding utf8
$result
```

### Template B — получить список счётчиков

```powershell
$token = $env:YANDEX_METRIKA_TOKEN

$counters = Invoke-RestMethod `
  -Uri "https://api-metrika.yandex.net/management/v1/counters?per_page=50" `
  -Headers @{ Authorization = "OAuth $token" } `
  -Method Get

$out = $counters.counters | ForEach-Object { "$($_.id) | $($_.name) | $($_.site)" }
[System.IO.File]::WriteAllLines("$env:TEMP\metrika_counters.txt", $out, [System.Text.Encoding]::UTF8)
$counters.counters | Select-Object id, name, site | Format-Table -AutoSize
```

### Template C — отчёт в CSV (для Excel)

```powershell
$token = $env:YANDEX_METRIKA_TOKEN
$cid   = "<COUNTER_ID>"

$uri = "https://api-metrika.yandex.net/stat/v1/data.csv?" +
  "ids=$cid&metrics=ym:s:visits,ym:s:users,ym:s:bounceRate&" +
  "dimensions=ym:s:date&date1=30daysAgo&date2=yesterday&sort=ym:s:date&lang=ru"

$bytes = (Invoke-WebRequest -Uri $uri -Headers @{ Authorization = "OAuth $token" }).Content
[System.IO.File]::WriteAllBytes("$env:TEMP\metrika_report.csv", $bytes)
Write-Host "CSV сохранён: $env:TEMP\metrika_report.csv"
```

### Template D — топ страниц по просмотрам

```powershell
$token = $env:YANDEX_METRIKA_TOKEN
$cid   = "<COUNTER_ID>"

$result = Invoke-RestMethod `
  -Uri ("https://api-metrika.yandex.net/stat/v1/data?" +
        "ids=$cid&dimensions=ym:pv:URL&metrics=ym:pv:pageviews&" +
        "sort=-ym:pv:pageviews&limit=20&date1=30daysAgo&date2=yesterday&lang=ru") `
  -Headers @{ Authorization = "OAuth $token" } `
  -Method Get

$result.data | ForEach-Object {
  "$($_.dimensions[0].name) — $($_.metrics[0]) просмотров"
}
```

### Template E — источники трафика (пресет)

```powershell
$token = $env:YANDEX_METRIKA_TOKEN
$cid   = "<COUNTER_ID>"

$result = Invoke-RestMethod `
  -Uri "https://api-metrika.yandex.net/stat/v1/data?ids=$cid&preset=sources_summary&date1=7daysAgo&date2=yesterday&lang=ru" `
  -Headers @{ Authorization = "OAuth $token" } `
  -Method Get

$result.data | ForEach-Object {
  "$($_.dimensions[0].name) — визиты: $($_.metrics[0])"
}
```

### Template F — цели счётчика

```powershell
$token = $env:YANDEX_METRIKA_TOKEN
$cid   = "<COUNTER_ID>"

$goals = Invoke-RestMethod `
  -Uri "https://api-metrika.yandex.net/management/v1/counter/$cid/goals" `
  -Headers @{ Authorization = "OAuth $token" } `
  -Method Get

$out = $goals.goals | ForEach-Object { "$($_.id) | $($_.name) | $($_.type)" }
[System.IO.File]::WriteAllLines("$env:TEMP\metrika_goals.txt", $out, [System.Text.Encoding]::UTF8)
$goals.goals | Select-Object id, name, type | Format-Table -AutoSize
```

### Template G — Logs API (сырые визиты)

```powershell
$token = $env:YANDEX_METRIKA_TOKEN
$cid   = "<COUNTER_ID>"
$base  = "https://api-metrika.yandex.net/management/v1/counter/$cid"

# 1. Создать задачу
$body = @{
  date1    = "2026-03-23"
  date2    = "2026-03-29"
  source   = "visits"   # или "hits"
  fields   = "ym:s:date,ym:s:clientID,ym:s:visits,ym:s:pageViews,ym:s:lastTrafficSource"
} | ConvertTo-Json

$task = Invoke-RestMethod -Uri "$base/logrequests" `
  -Headers @{ Authorization = "OAuth $token"; "Content-Type" = "application/json" } `
  -Method Post -Body $body

$reqId = $task.log_request.request_id
Write-Host "Задача создана: $reqId"

# 2. Ждать готовности (poll)
do {
  Start-Sleep -Seconds 10
  $status = Invoke-RestMethod -Uri "$base/logrequest/$reqId" `
    -Headers @{ Authorization = "OAuth $token" } -Method Get
  Write-Host "Статус: $($status.log_request.status)"
} while ($status.log_request.status -notin "processed","processing_failed")

# 3. Скачать части (part 0, 1, …)
if ($status.log_request.status -eq "processed") {
  $parts = $status.log_request.parts
  for ($i = 0; $i -lt $parts.Count; $i++) {
    $data = Invoke-RestMethod -Uri "$base/logrequest/$reqId/part/$i/download" `
      -Headers @{ Authorization = "OAuth $token" } -Method Get
    [System.IO.File]::WriteAllText("$env:TEMP\logs_part$i.tsv", $data, [System.Text.Encoding]::UTF8)
    Write-Host "Part $i сохранён"
  }
  # 4. ОБЯЗАТЕЛЬНО очистить после скачивания
  Invoke-RestMethod -Uri "$base/logrequest/$reqId/clean" `
    -Headers @{ Authorization = "OAuth $token" } -Method Post
  Write-Host "Logs очищены (квота освобождена)"
}
```

## Маршрутизация: что делать при разных запросах

| Пользователь говорит | Что делать | Шаблон |
|---|---|---|
| визиты / посетители / сеансы | stat/v1/data, `ym:s:visits,ym:s:users` | Template A |
| по дням / динамика | добавить `dimensions=ym:s:date` или bytime | Template A |
| сравни два периода | stat/v1/data/comparison | Template A (url bytime/comparison) |
| топ страниц / просмотры | `ym:pv:URL` + `ym:pv:pageviews` | Template D |
| откуда трафик / источники | `preset=sources_summary` | Template E |
| органика / поиск | `preset=sources_search_phrases` | Template E |
| Директ / реклама | `preset=sources_direct` | Template E |
| UTM-метки | dimensions по UTM (справочник) | Template A |
| какие счётчики / id счётчика | management/v1/counters | Template B |
| цели / конверсии | management goals + stat с метриками целей | Template F + A |
| CSV / Excel | суффикс .csv в URL | Template C |
| сырые данные / логи | Logs API: create→poll→download→clean | Template G |
| выгрузить за месяц | Logs API (visits или hits) | Template G |

## Обработка ID счётчика

1. Если в env есть `YANDEX_METRIKA_COUNTER_ID` — использовать его и написать «использую счётчик по умолчанию».
2. Иначе — спросить пользователя или выполнить Template B (список счётчиков) и попросить выбрать.

## Обработка ошибок

| HTTP код | Значение |
|---|---|
| 200 | Успех |
| 400 | Неверные параметры (не та метрика/группировка, смешаны ym:s: и ym:pv:) |
| 401 | Неверный или отсутствующий токен |
| 403 | Нет доступа к счётчику |
| 429 | Превышена квота — `200 запросов/5 мин` на stat/v1/data — подождать |

Если ошибка 400 — проверить совместимость prefix'ов `ym:s:` и `ym:pv:`, уточнить имена из справочника.

## Лимиты stat API

- Метрик в одном запросе: **до 20**
- Группировок в одном запросе: **до 10**
- Строк на страницу: **до 100 000** (default 100)
- Квота: **200 запросов / 5 мин** на `/stat/v1/data`
- Logs API хранилище: **10 ГБ** на счётчик (обязательно очищать после скачивания)

## Данные о расходах и конверсиях

Если нужна связка с Директом (расходы + конверсии), используй:

- В stat: `preset=sources_direct` или `direct_client_logins=<login>` для данных по Директу
- Для импорта расходов (если данные загружены в Метрику): `preset=expenses_by_source`

## Ссылки

- [Примеры запросов stat](https://yandex.ru/dev/metrika/ru/stat/examples)
- [Справочник метрик и группировок](https://yandex.ru/dev/metrika/ru/stat/)
- [Шаблоны отчётов (presets)](https://yandex.ru/dev/metrika/ru/stat/presets)
- [Management API](https://yandex.ru/dev/metrika/ru/management/)
- [Logs API](https://yandex.ru/dev/metrika/ru/logs/)
