# Reference: Yandex Wordstat API

Compact reference for the `yandex_wordstat` OpenClaw skill.

## Авторизация

```
Authorization: Bearer <YANDEX_WORDSTAT_TOKEN>
Content-type: application/json;charset=utf-8
```

`Bearer` (как в Директе), **не** `OAuth` (как в Метрике).

## Базовый URL

```
https://api.wordstat.yandex.net
```

Все методы: `POST` + JSON body.

## Методы

| Endpoint | Квота | Назначение |
|---|---|---|
| `POST /v1/topRequests` | 1 | Топ запросов + ассоциации, последние 30 дней |
| `POST /v1/dynamics` | 1 | Динамика спроса daily/weekly/monthly |
| `POST /v1/regions` | 2 | Распределение по регионам |
| `POST /v1/getRegionsTree` | 0 | Список регионов и их ID |
| `POST /v1/userInfo` | 0 | Квоты пользователя |

## /v1/topRequests

```json
{
  "phrase": "горный поход кавказ",
  "numPhrases": 50,
  "regions": [213],
  "devices": ["all"]
}
```

Или несколько фраз сразу (до 128):

```json
{
  "phrases": ["треккинг приэльбрусье", "поход эльбрус", "тур кавказ"],
  "numPhrases": 10
}
```

Ответ: `requestPhrase`, `totalCount`, `topRequests[]` (phrase+count), `associations[]` (phrase+count).

## /v1/dynamics

```json
{
  "phrase": "горный поход",
  "period": "monthly",
  "fromDate": "2025-01-01",
  "toDate": "2026-02-28"
}
```

| period | fromDate | toDate |
|---|---|---|
| `monthly` | Первое число месяца | Последнее число месяца |
| `weekly` | Понедельник | Воскресенье |
| `daily` | Любая дата (макс. 60 дней назад) | Любая |

Ответ: массив `dynamics[]` с `date`, `count`, `share`.

Только оператор `+` допустим в `phrase`.

## /v1/regions

```json
{
  "phrase": "поход приэльбрусье",
  "regionType": "cities",
  "devices": ["all"]
}
```

`regionType`: `cities` / `regions` / `all`

Ответ: массив `regions[]` с `regionId`, `count`, `share`, `affinityIndex`.

`affinityIndex > 100` = повышенный интерес в регионе относительно среднего по стране.

## Популярные ID регионов

| ID | Регион |
|---|---|
| 213 | Москва |
| 2 | Санкт-Петербург |
| 10995 | Краснодарский край |
| 20785 | Кабардино-Балкарская Республика |
| 11004 | Ставропольский край |
| 11005 | Карачаево-Черкесская Республика |
| 1 | Россия (все регионы) |

## Операторы в phrase

| Оператор | Пример | Эффект |
|---|---|---|
| `+слово` | `поход +в горы` | Стоп-слово обязательно |
| `"фраза"` | `"треккинг приэльбрусье"` | Точное соответствие |
| `!слово` | `!поход` | Фиксирует форму слова |
| `-слово` | `поход -эльбрус` | Исключить слово |
| `[а б]` | `[горный поход]` | Фиксирует порядок |

В `/v1/dynamics` — только оператор `+`.

## Квоты

- Лимит запросов в секунду: индивидуальный (см. `/v1/userInfo`)
- Дневной лимит: индивидуальный, обновляется в полночь МСК
- `/v1/regions` стоит **2 единицы** (дорогой метод)

## Как использовать совместно с Директом

```
Wordstat topRequests  →  список ключей  →  добавить в кампанию Direct (keywords.add)
Wordstat dynamics     →  сезонный пик   →  поднять ставки в Direct на этот период
Wordstat regions      →  affinityIndex  →  настроить гео-корректировки ставок в Direct
```

## Типичные сценарии

| Задача | Что вызвать |
|---|---|
| Подобрать семантику для тура | `topRequests` по основным фразам |
| Найти минус-слова | `topRequests` → отфильтровать нерелевантные |
| Когда поднимать бюджет | `dynamics monthly` — найти пиковые месяцы |
| Откуда целевая аудитория | `regions` по основным запросам |
| Сравнить спрос по нескольким направлениям | `topRequests` с `phrases[]` |
