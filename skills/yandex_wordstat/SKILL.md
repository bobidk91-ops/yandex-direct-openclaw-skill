---
name: yandex_wordstat
description: Call Yandex Wordstat API — get search volume (topRequests), dynamics over time (daily/weekly/monthly), regional distribution, and associations for any keyword phrase. Use when the user wants to research demand, pick keywords, check seasonal trends, or compare query popularity across regions.
metadata: {"openclaw":{"requires":{"env":["YANDEX_WORDSTAT_TOKEN"]},"primaryEnv":"YANDEX_WORDSTAT_TOKEN","emoji":"📈"}}
homepage: https://yandex.com/support2/wordstat/ru/content/api-structure
---

# Яндекс Вордстат API

## Ключевые отличия от других Яндекс-скилов

| | Директ | Метрика | Вордстат |
|---|---|---|---|
| Хост | `api.direct.yandex.com` | `api-metrika.yandex.net` | `api.wordstat.yandex.net` |
| Авторизация | `Bearer` | `OAuth` | **`Bearer`** |
| Метод запросов | POST JSON | GET query params | **POST JSON** |
| Токен env | `YANDEX_DIRECT_TOKEN` | `YANDEX_METRIKA_TOKEN` | `YANDEX_WORDSTAT_TOKEN` |

**Важно:** авторизация `Bearer` (как в Директе), не `OAuth`.
Content-Type: `application/json;charset=utf-8`

## Safety requirements

- Никогда не выводить `YANDEX_WORDSTAT_TOKEN` в логи, ответы или файлы.
- Только хост `api.wordstat.yandex.net`.
- В методе `/v1/dynamics` допускается только оператор `+` в поле `phrase`.

## Базовый URL

```
https://api.wordstat.yandex.net
```

## Авторизация

```
Authorization: Bearer <YANDEX_WORDSTAT_TOKEN>
Content-type: application/json;charset=utf-8
```

## Методы API

| Метод | Endpoint | Квота | Назначение |
|---|---|---|---|
| Дерево регионов | `POST /v1/getRegionsTree` | 0 | Получить ID регионов для других методов |
| Топ запросов | `POST /v1/topRequests` | 1 | Популярные запросы + ассоциации по фразе |
| Динамика | `POST /v1/dynamics` | 1 | Тренд спроса по дням/неделям/месяцам |
| Регионы | `POST /v1/regions` | 2 | Распределение запросов по регионам |
| Инфо о пользователе | `POST /v1/userInfo` | 0 | Квоты, лимиты, остаток |

## Параметры методов

### /v1/topRequests — частотность и ассоциации

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `phrase` | string | да* | Одна фраза для анализа |
| `phrases` | string[] | да* | До 128 фраз сразу |
| `numPhrases` | int | нет | Кол-во результатов (default 50, max 2000) |
| `regions` | int[] | нет | ID регионов из getRegionsTree (по умолчанию все) |
| `devices` | string[] | нет | `all`, `desktop`, `phone`, `tablet` (default `all`) |

Возвращает: `totalCount`, массив `topRequests` (фраза + count), массив `associations`.

### /v1/dynamics — динамика спроса во времени

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `phrase` | string | да | Фраза (только оператор `+` допустим) |
| `period` | string | да | `monthly`, `weekly`, `daily` |
| `fromDate` | string | да | `YYYY-MM-DD` (daily — последние 60 дней; weekly/monthly — с 2018-01-01) |
| `toDate` | string | нет | `YYYY-MM-DD` (для monthly — последний день месяца) |
| `regions` | int[] | нет | ID регионов |
| `devices` | string[] | нет | Типы устройств |

Ограничения дат:
- `monthly`: `fromDate` и `toDate` — первое/последнее число месяца
- `weekly`: `fromDate` — понедельник, `toDate` — воскресенье

### /v1/regions — региональное распределение

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `phrase` | string | да | Фраза |
| `regionType` | string | нет | `cities`, `regions`, `all` (default `all`) |
| `devices` | string[] | нет | Типы устройств |

Возвращает: массив `regions` с `regionId`, `count`, `share`, `affinityIndex`.

## Как агент должен выполнять запросы (Windows PowerShell)

### Template A — частотность фразы + похожие запросы

```powershell
$token = $env:YANDEX_WORDSTAT_TOKEN

$body = @{
  phrase     = "поход в приэльбрусье"
  numPhrases = 50
  # regions  = @(213)   # Москва; убрать для всей России
  # devices  = @("all") # all / desktop / phone / tablet
} | ConvertTo-Json -Compress

$result = Invoke-RestMethod `
  -Uri "https://api.wordstat.yandex.net/v1/topRequests" `
  -Method Post `
  -Headers @{ Authorization = "Bearer $token"; "Content-type" = "application/json;charset=utf-8" } `
  -Body ([System.Text.Encoding]::UTF8.GetBytes($body)) `
  -ContentType "application/json;charset=utf-8"

# Топ запросов
Write-Host "Всего запросов по фразе: $($result.totalCount)"
Write-Host "`nТоп запросов:"
$result.topRequests | Select-Object -First 20 | ForEach-Object {
  "{0,-50} {1,8}" -f $_.phrase, $_.count
}

Write-Host "`nАссоциации:"
$result.associations | Select-Object -First 10 | ForEach-Object {
  "{0,-50} {1,8}" -f $_.phrase, $_.count
}
```

### Template B — несколько фраз сразу (до 128)

```powershell
$token = $env:YANDEX_WORDSTAT_TOKEN

$body = @{
  phrases = @(
    "поход в приэльбрусье",
    "треккинг в приэльбрусье",
    "тур в приэльбрусье",
    "горный поход кавказ",
    "восхождение на эльбрус"
  )
  numPhrases = 5
} | ConvertTo-Json -Compress

$results = Invoke-RestMethod `
  -Uri "https://api.wordstat.yandex.net/v1/topRequests" `
  -Method Post `
  -Headers @{ Authorization = "Bearer $token"; "Content-type" = "application/json;charset=utf-8" } `
  -Body ([System.Text.Encoding]::UTF8.GetBytes($body)) `
  -ContentType "application/json;charset=utf-8"

$out = $results | ForEach-Object {
  "$($_.requestPhrase) | totalCount: $($_.totalCount)"
}
[System.IO.File]::WriteAllLines("$env:TEMP\wordstat_compare.txt", $out, [System.Text.Encoding]::UTF8)
$results | Select-Object requestPhrase, totalCount | Format-Table -AutoSize
```

### Template C — сезонная динамика (по месяцам)

```powershell
$token = $env:YANDEX_WORDSTAT_TOKEN

# fromDate — первое число месяца, toDate — последнее
$body = @{
  phrase   = "поход в горы"
  period   = "monthly"
  fromDate = "2025-01-01"
  toDate   = "2026-02-28"
} | ConvertTo-Json -Compress

$result = Invoke-RestMethod `
  -Uri "https://api.wordstat.yandex.net/v1/dynamics" `
  -Method Post `
  -Headers @{ Authorization = "Bearer $token"; "Content-type" = "application/json;charset=utf-8" } `
  -Body ([System.Text.Encoding]::UTF8.GetBytes($body)) `
  -ContentType "application/json;charset=utf-8"

$out = $result.dynamics | ForEach-Object {
  "$($_.date) | $($_.count) запросов | доля: $([math]::Round($_.share * 100, 3))%"
}
[System.IO.File]::WriteAllLines("$env:TEMP\wordstat_dynamics.txt", $out, [System.Text.Encoding]::UTF8)
Get-Content "$env:TEMP\wordstat_dynamics.txt"
```

### Template D — динамика по неделям (последние 8 недель)

```powershell
$token = $env:YANDEX_WORDSTAT_TOKEN

# fromDate — понедельник, toDate — воскресенье
$body = @{
  phrase   = "горный поход"
  period   = "weekly"
  fromDate = "2026-02-02"
  toDate   = "2026-03-29"
} | ConvertTo-Json -Compress

$result = Invoke-RestMethod `
  -Uri "https://api.wordstat.yandex.net/v1/dynamics" `
  -Method Post `
  -Headers @{ Authorization = "Bearer $token"; "Content-type" = "application/json;charset=utf-8" } `
  -Body ([System.Text.Encoding]::UTF8.GetBytes($body)) `
  -ContentType "application/json;charset=utf-8"

$result.dynamics | ForEach-Object {
  "Неделя $($_.date): $($_.count)"
}
```

### Template E — региональное распределение

```powershell
$token = $env:YANDEX_WORDSTAT_TOKEN

$body = @{
  phrase     = "горный поход кавказ"
  regionType = "cities"   # cities / regions / all
} | ConvertTo-Json -Compress

$result = Invoke-RestMethod `
  -Uri "https://api.wordstat.yandex.net/v1/regions" `
  -Method Post `
  -Headers @{ Authorization = "Bearer $token"; "Content-type" = "application/json;charset=utf-8" } `
  -Body ([System.Text.Encoding]::UTF8.GetBytes($body)) `
  -ContentType "application/json;charset=utf-8"

# Нужно сопоставить regionId с именами через getRegionsTree
$result.regions | Sort-Object count -Descending | Select-Object -First 15 | ForEach-Object {
  "regionId: $($_.regionId) | count: $($_.count) | share: $([math]::Round($_.share*100,2))% | affinityIndex: $([math]::Round($_.affinityIndex,1))"
}
```

### Template F — получить дерево регионов (для фильтрации)

```powershell
$token = $env:YANDEX_WORDSTAT_TOKEN

$regions = Invoke-RestMethod `
  -Uri "https://api.wordstat.yandex.net/v1/getRegionsTree" `
  -Method Post `
  -Headers @{ Authorization = "Bearer $token"; "Content-type" = "application/json;charset=utf-8" } `
  -Body "{}"  `
  -ContentType "application/json;charset=utf-8"

# Найти нужный регион по имени
$regions | ConvertTo-Json -Depth 10 | Out-File "$env:TEMP\wordstat_regions.json" -Encoding utf8
# Основные ID: Москва=213, Санкт-Петербург=2, Краснодарский край=10995
Write-Host "Дерево регионов сохранено в $env:TEMP\wordstat_regions.json"
```

### Template G — проверить квоты

```powershell
$token = $env:YANDEX_WORDSTAT_TOKEN

$info = Invoke-RestMethod `
  -Uri "https://api.wordstat.yandex.net/v1/userInfo" `
  -Method Post `
  -Headers @{ Authorization = "Bearer $token"; "Content-type" = "application/json;charset=utf-8" } `
  -Body "{}" `
  -ContentType "application/json;charset=utf-8"

Write-Host "Логин: $($info.userInfo.login)"
Write-Host "Лимит запросов/сек: $($info.userInfo.limitPerSecond)"
Write-Host "Дневной лимит: $($info.userInfo.dailyLimit)"
Write-Host "Остаток сегодня: $($info.userInfo.dailyLimitRemaining)"
```

## Маршрутизация: что делать при разных запросах

| Пользователь говорит | Метод | Шаблон |
|---|---|---|
| частотность / сколько ищут / спрос | `/v1/topRequests` | Template A |
| похожие запросы / ассоциации / что ещё ищут | `/v1/topRequests` (поле `associations`) | Template A |
| сравнить несколько фраз / какой запрос популярнее | `/v1/topRequests` с `phrases` | Template B |
| сезонность / динамика / когда пик спроса | `/v1/dynamics`, `period=monthly` | Template C |
| недельный тренд / растёт или падает | `/v1/dynamics`, `period=weekly` | Template D |
| из каких городов / регионов ищут | `/v1/regions` | Template E |
| ключевые слова для Директа / подобрать семантику | `/v1/topRequests`, затем анализ | Template A→B |
| квота / сколько запросов осталось | `/v1/userInfo` | Template G |

## Связка с Директом и Метрикой

Вордстат дополняет другие скилы:

1. **Подбор ключей для Директа**: topRequests → отфильтровать релевантные → добавить в кампанию через `yandex_direct`
2. **Сезонность**: dynamics → понять когда поднимать ставки в Директе
3. **Регионы**: regions (affinityIndex) → настроить гео-корректировки ставок
4. **Аудит**: сравнить топ запросов Вордстата с реальными ключами в кампаниях

## Операторы Вордстата (в поле phrase)

| Оператор | Пример | Что делает |
|---|---|---|
| `+слово` | `поход +в горы` | Учитывает стоп-слово обязательно |
| `"фраза"` | `"треккинг приэльбрусье"` | Точное соответствие (фиксирует словоформу) |
| `!слово` | `!поход` | Фиксирует конкретную форму слова |
| `-слово` | `поход -Эльбрус` | Исключает слово |
| `[а б]` | `[поход горы]` | Фиксирует порядок слов |

**Внимание:** в `/v1/dynamics` допускается только оператор `+`.

## Квоты

| Метод | Стоимость | Лимиты |
|---|---|---|
| `/v1/topRequests` | 1 единица | до 128 фраз за раз |
| `/v1/dynamics` | 1 единица | — |
| `/v1/regions` | 2 единицы | — |
| `/v1/getRegionsTree` | 0 | — |
| `/v1/userInfo` | 0 | — |

Квота обновляется в полночь по московскому времени.

## Обработка ошибок

- Если в `phrases` для какой-то фразы ошибка — элемент будет содержать поле `error` вместо данных.
- `period=monthly`: `fromDate` обязан быть первым числом месяца, иначе ошибка.
- `period=weekly`: `fromDate` обязан быть понедельником.

## Ссылки

- [Структура API Вордстата](https://yandex.com/support2/wordstat/ru/content/api-structure)
- [Операторы Вордстата](https://yandex.com/support2/wordstat/ru/content/operators)
- [Интерфейс Вордстата](https://wordstat.yandex.ru)
