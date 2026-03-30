# Reference: Yandex Metrica API

Compact reference for the `yandex_metrika` OpenClaw skill.

## Авторизация

```
Authorization: OAuth <YANDEX_METRIKA_TOKEN>
```

**Важно:** `OAuth`, не `Bearer` (в отличие от Яндекс.Директа).

## Базовый URL

```
https://api-metrika.yandex.net
```

## Основные endpoints

| Endpoint | Метод | Назначение |
|---|---|---|
| `/stat/v1/data` | GET | Таблица: визиты, хиты, цели, UTM |
| `/stat/v1/data.csv` | GET | То же в CSV (для Excel) |
| `/stat/v1/data/bytime` | GET | Динамика по времени |
| `/stat/v1/data/comparison` | GET | Сравнение двух сегментов/периодов |
| `/stat/v1/data/drilldown` | GET | Дерево отчёта |
| `/management/v1/counters` | GET | Список счётчиков пользователя |
| `/management/v1/counter/{id}` | GET | Параметры счётчика |
| `/management/v1/counter/{id}/goals` | GET/POST | Цели счётчика |
| `/management/v1/counter/{id}/logrequests` | POST | Создать задачу Logs API |
| `/management/v1/counter/{id}/logrequest/{reqId}` | GET | Статус задачи |
| `/management/v1/counter/{id}/logrequest/{reqId}/part/{N}/download` | GET | Скачать часть логов |
| `/management/v1/counter/{id}/logrequest/{reqId}/clean` | POST | Удалить (освободить квоту) |

## Параметры /stat/v1/data

| Параметр | Описание |
|---|---|
| `ids` | ID счётчика (или несколько через запятую) |
| `metrics` | Метрики через запятую (макс. 20) |
| `dimensions` | Группировки через запятую (макс. 10) |
| `date1` / `date2` | `YYYY-MM-DD`, `today`, `yesterday`, `NdaysAgo` |
| `filters` | Сегментация (до 10 измерений, 20 условий) |
| `sort` | Имя метрики; `-` перед именем — по убыванию |
| `limit` | Строк (макс. 100 000, default 100) |
| `offset` | Смещение с 1 |
| `preset` | Готовый шаблон отчёта |
| `lang` | `ru` — названия городов и источников на русском |
| `accuracy` | Уровень семплирования |
| `timezone` | `+03:00` и т.д. (по умолчанию — TZ счётчика) |

## Префиксы метрик и группировок

**Нельзя смешивать в одном запросе:**

| Префикс | Тип данных | Примеры |
|---|---|---|
| `ym:s:` | Визиты (sessions) | `ym:s:visits`, `ym:s:users`, `ym:s:date`, `ym:s:bounceRate`, `ym:s:avgVisitDurationSeconds` |
| `ym:pv:` | Просмотры страниц (pageviews) | `ym:pv:pageviews`, `ym:pv:URL` |

В `filters` допускается другой префикс.

## Популярные пресеты (preset=)

| Пресет | Что показывает |
|---|---|
| `sources_summary` | Сводка по источникам трафика |
| `sources_search_phrases` | Поисковые фразы |
| `sources_direct` | Данные Яндекс.Директа |
| `sources_social` | Социальные сети |
| `tech_platforms` | Платформы и браузеры |
| `expenses_by_source` | Рекламные расходы (если загружены) |

Полный список: [yandex.ru/dev/metrika/ru/stat/presets](https://yandex.ru/dev/metrika/ru/stat/presets)

## Часто используемые метрики (ym:s:)

| Метрика | Значение |
|---|---|
| `ym:s:visits` | Визиты |
| `ym:s:users` | Уникальные посетители |
| `ym:s:pageViews` | Просмотры страниц |
| `ym:s:bounceRate` | Процент отказов |
| `ym:s:avgVisitDurationSeconds` | Среднее время на сайте (сек) |
| `ym:s:avgPageViews` | Глубина просмотра |

## Часто используемые группировки (ym:s:)

| Группировка | Значение |
|---|---|
| `ym:s:date` | День визита |
| `ym:s:trafficSource` | Источник трафика |
| `ym:s:searchEngine` | Поисковая система |
| `ym:s:searchPhrase` | Поисковая фраза |
| `ym:s:regionCountry` | Страна |
| `ym:s:regionCity` | Город |
| `ym:s:deviceCategory` | Тип устройства |
| `ym:s:operatingSystem` | ОС |
| `ym:s:browser` | Браузер |
| `ym:s:UTMSource` | UTM source |
| `ym:s:UTMMedium` | UTM medium |
| `ym:s:UTMCampaign` | UTM campaign |

## Квоты и лимиты

| Ограничение | Значение |
|---|---|
| Запросов к /stat/v1/data | 200 / 5 мин на пользователя |
| Запросов к Logs API с IP | 10 / сек |
| Хранилище Logs на счётчик | 10 ГБ (очищать после загрузки!) |
| Метрик в одном запросе | 20 |
| Группировок в одном запросе | 10 |

## Обработка ошибок

| HTTP | Значение | Действие |
|---|---|---|
| 400 | Неверные параметры | Проверить совместимость prefix'ов, имена из справочника |
| 401 | Неверный токен | Проверить `YANDEX_METRIKA_TOKEN` |
| 403 | Нет доступа к счётчику | Проверить права доступа к счётчику |
| 429 | Превышена квота | Ждать, сократить частоту запросов |

## PowerShell — важные моменты

- `Invoke-RestMethod` работает корректно для Метрики (в отличие от Директа — там нужен `HttpWebRequest`)
- Заголовок авторизации: `-Headers @{ Authorization = "OAuth $token" }`
- Для сохранения с кириллицей: `[System.IO.File]::WriteAllLines(path, content, UTF8)`
- Для CSV: использовать `Invoke-WebRequest` и сохранять бинарный контент
