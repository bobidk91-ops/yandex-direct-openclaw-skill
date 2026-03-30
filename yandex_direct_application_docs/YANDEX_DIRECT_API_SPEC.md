# Спецификация интеграции с Яндекс.Директ API (v5 / v501, JSON)

## 1. Назначение приложения
Приложение выполняет автоматизированные вызовы к официальному API Яндекс.Директ для:
- управления рекламными объектами (кампании, группы, объявления, ключевые фразы, ставки и корректировки) через стандартные методы сервисов: `add/update/delete/get/suspend/resume` и дополнительные lifecycle-методы, доступные конкретному сервису;
- получения статистики и формирования отчетов через сервис `reports`.

Все операции выполняются **от имени пользователя** через OAuth 2.0 access token.

## 2. Аутентификация (OAuth 2.0)
В каждом запросе используется OAuth access token в HTTP-заголовке:
- `Authorization: Bearer <access_token>`

## 3. Обязательные заголовки Yandex Direct
Для всех JSON-запросов приложение передает следующие заголовки:
- `Authorization: Bearer <access_token>`
- `Client-Login: <YANDEX_DIRECT_LOGIN>`
- `Accept-Language: ru`
- `Content-Type: application/json; charset=utf-8`

Где:
- `YANDEX_DIRECT_LOGIN` — логин пользователя/представителя в Яндекс.Директ, для которого выдан токен.
- Язык по умолчанию `ru` (при необходимости может переопределяться настройкой приложения).

## 4. Базовые URL (JSON mode)
Формат запроса строится одинаково для всех сервисов; отличается только домен/версии.

Production:
- v5: `https://api.direct.yandex.com/json/v5/{service}`
- v501: `https://api.direct.yandex.com/json/v501/{service}`

Sandbox:
- v5: `https://api-sandbox.direct.yandex.ru/json/v5/{service}`
- v501: `https://api-sandbox.direct.yandex.ru/json/v501/{service}`

Выбор `service` соответствует нужному объекту/функции (например `campaigns`, `ads`, `keywords`, `reports`, и т.д.).

## 5. Формат тела запроса

### 5.1. Стандартные сервисы (most services)
Для большинства сервисов тело запроса имеет вид:

```json
{
  "method": "<methodName>",
  "params": { /* method-specific parameters */ }
}
```

### 5.2. Сервис `reports` (особенность)
Для `service=reports` поле `method` в JSON теле **не используется** (используется только структура параметров отчета):

```json
{
  "params": {
    /* ReportDefinition */
  }
}
```

### 5.3. Структура параметров отчета (`reports`)
Параметры формируются по схеме **ReportDefinition** и обычно включают:
- `SelectionCriteria` (например даты `DateFrom/DateTo` + `Filter` при необходимости);
- `FieldNames` (список полей/столбцов отчета);
- `ReportName` (название отчета);
- `ReportType` (тип отчета);
- `DateRangeType`;
- `Format` (в настоящее время `TSV`);
- `IncludeVAT` (и `IncludeDiscount`, если применимо).

## 6. Обработка отчетов online/offline
Сервер может обрабатывать отчеты в двух режимах:
- `HTTP 200`: отчет готов, тело ответа содержит результат в TSV.
- `HTTP 201` или `HTTP 202`: отчет еще формируется/в очереди; нужно повторить **тот же запрос** позже.
  - При наличии заголовка `retryIn` приложение ожидает указанное время перед повтором.

## 7. Sandbox/Production
Приложение поддерживает переключение на sandbox (test) домены по требованию процесса разработки:
- при использовании sandbox доменов используется `api-sandbox.direct.yandex.ru`;
- при использовании production используется `api.direct.yandex.com`.

## 8. Минимальные входные данные для выполнения вызова
Для любого вызова приложение должно определить:
- `service` (сервис/объект: например `campaigns`, `ads`, `keywords`, `reports`, и т.д.);
- `method` (для большинства сервисов; для `reports` не используется);
- `params` (JSON объект с параметрами конкретного вызова или схемы отчета).

## 9. Примечания по безопасности
- OAuth токен применяется только в заголовке `Authorization` и не передается пользователю/в prompts.
- Для запросов используется только HTTPS.

## 10. Полезные официальные ссылки
- [Overview Direct API v5](https://yandex.ru/dev/direct/doc/en/concepts/overview)
- [Access tokens](https://yandex.ru/dev/direct/doc/en/concepts/auth-token)
- [Campaigns service](https://yandex.ru/dev/direct/doc/en/campaigns/campaigns)
- [Ads.get (пример структуры)](https://yandex.ru/dev/direct/doc/en/ads/get)
- [Reports service](https://yandex.ru/dev/direct/doc/reports/reports.html)
- [Как сформировать отчет](https://yandex.ru/dev/direct/doc/ru/how-to)

