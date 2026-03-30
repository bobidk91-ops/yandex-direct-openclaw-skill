# Яндекс.Директ API — спецификация для заявки

## Цель интеграции
Приложение выполняет автоматизированные операции с Яндекс.Директ через **Yandex Direct API v5/v501 (JSON)**:
- управление рекламными сущностями (кампании/группы/объявления/ключевые фразы/ставки и корректировки) — методы `add/update/delete/get/suspend/resume/...`
- получение статистики/отчетов через сервис **`reports`**

Все запросы выполняются **от имени пользователя** с использованием OAuth access token.

## Авторизация (OAuth)
Для каждого запроса используется OAuth 2.0 access token, отправляемый в HTTP-заголовке:
- `Authorization: Bearer <access_token>`

Дополнительно (требуемые заголовки Яндекс.Директ):
- `Client-Login: <YANDEX_DIRECT_LOGIN>`
- `Accept-Language: ru`
- `Content-Type: application/json; charset=utf-8`

## Базовые URL (JSON режим)
Production:
- `https://api.direct.yandex.com/json/v5/{service}`
- `https://api.direct.yandex.com/json/v501/{service}`

Sandbox:
- `https://api-sandbox.direct.yandex.ru/json/v5/{service}`
- `https://api-sandbox.direct.yandex.ru/json/v501/{service}`

Сервис выбирается по объекту/домену API, URL строится как `/{service}`.

## Формат запроса (основные сервисы)
Для большинства сервисов (campaigns/ads/keywords/...) тело запроса имеет форму:

```json
{
  "method": "<methodName>",
  "params": { /* method-specific params */ }
}
```

## Формат запроса (Reports)
Для `service=reports` JSON тело запроса **отличается** — используется только `params`:

```json
{
  "params": {
    /* ReportDefinition: SelectionCriteria, FieldNames, ReportType, DateRangeType, ... */
  }
}
```

## Онлайн/офлайн обработка отчета (reports)
- HTTP `200`: отчет сразу возвращается в теле (TSV).
- HTTP `201` / `202`: отчет формируется/в очереди; запрос нужно повторить позже.
  - при наличии заголовка `retryIn` повторить после указанного интервала.

## Минимальный набор входных данных для вызова
Приложение определяет:
- `service` (например `campaigns`, `ads`, `keywords`, `reports`, ...)
- `method` (например `get`, `add`, `update`, `delete`, `suspend`, `resume`, ...)
- `params` (JSON-структура параметров конкретного метода)

## Безопасность
- Токен доступа используется только в HTTP-заголовке и не передается в prompts.
- Секреты (token) хранятся и передаются приложению безопасно (не логируются).

## Полезные ссылки (официальные)
- Обзор Direct API v5: https://yandex.ru/dev/direct/doc/en/concepts/overview
- Campaigns service: https://yandex.ru/dev/direct/doc/en/campaigns/campaigns
- Ads.get (пример структуры): https://yandex.ru/dev/direct/doc/en/ads/get
- Reports service: https://yandex.ru/dev/direct/doc/reports/reports.html
- Как сформировать отчет: https://yandex.ru/dev/direct/doc/ru/how-to

