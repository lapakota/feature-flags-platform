# Синхронные и асинхронные взаимодействия

## Сценарии и выбор sync или async

| Сценарий                                              | Пользователь ждёт ответ                    | Модель | Обоснование                                              |
| ----------------------------------------------------- | ------------------------------------------ | ------ | -------------------------------------------------------- |
| Получение состояний флагов и конфигов                 | Да, через систему-потребитель              | Sync   | Состояния нужны приложению для текущего запроса          |
| Открытие списка, страницы флага или истории           | Да                                         | Sync   | Пользователь ждёт данные на экране                       |
| Сохранение черновика и настройка rollout              | Да                                         | Sync   | Нужно сразу показать, сохранены ли изменения             |
| Публикация или rollback: проверка и сохранение версии | Да, подтверждение создания версии          | Sync   | Нужно сообщить номер созданной версии или ошибку         |
| Передача версии в контур чтения и запись истории      | Нет, в админке пока показано «Применяется» | Async  | Ответ админке не должен ждать обновления других сервисов |

## Оркестрация и события

- Оркестрация: `Configuration Service` проверяет право через `Auth Service`,
  проверяет конфигурацию и сохраняет новую версию при публикации или rollback.
- События: `Delivery Service` получает PublishedConfiguration и обновляет
  конфигурацию; `History Service` получает ConfigurationChanged и записывает
  изменение. Они работают независимо друг от друга.

## Контракты

### Получение списка опубликованных флагов

Версия контракта: 1. Все поля обязательны.

```http
GET /v1/configurations/{configurationId}/flags
Authorization: Bearer <token>
```

| Поле                   | Где передаётся | Тип     | Описание                         |
| ---------------------- | -------------- | ------- | -------------------------------- |
| configurationId        | Путь и ответ   | string  | Идентификатор набора настроек    |
| Authorization          | Заголовок      | string  | Токен пользователя админки       |
| version                | Ответ          | integer | Номер опубликованной версии      |
| flags                  | Ответ          | array   | Список флагов, может быть пустым |
| flags[].key            | Ответ          | string  | Ключ флага                       |
| flags[].enabled        | Ответ          | boolean | Включён ли флаг                  |
| flags[].rolloutPercent | Ответ          | integer | Rollout от 0 до 100% с шагом 1%  |

Ответ:

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "configurationId": "storefront",
  "version": 67,
  "flags": [
    {
      "key": "new-checkout",
      "enabled": true,
      "rolloutPercent": 25
    }
  ]
}
```

Ошибки: 401 — токен недействителен, 403 — нет права на просмотр,
404 — конфигурация или опубликованная версия не найдена,
503 — проверка прав недоступна.

### Событие PublishedConfiguration

Версия контракта: 1. Все поля обязательны.

После публикации или отката `Configuration Service` отправляет событие
в `Delivery Service`. В нём передаются все флаги и JSON-конфиги этой версии,
а не только изменения или ссылка на данные. Дополнительно запрашивать настройки
у `Configuration Service` не нужно.

| Поле | Тип | Описание |
|---|---|---|
| contractVersion | integer | Версия схемы события: 1 |
| eventId | string | Уникальный идентификатор события; при повторной отправке не меняется |
| occurredAt | string | Время публикации в UTC, формат ISO 8601 |
| configurationId | string | Идентификатор набора настроек |
| version | integer | Номер опубликованной версии; при откате тоже увеличивается |
| flags | array | Полный список флагов, может быть пустым |
| flags[].key | string | Ключ флага |
| flags[].enabled | boolean | false отключает флаг независимо от rollout |
| flags[].rolloutPercent | integer | Rollout от 0 до 100% с шагом 1% |
| flags[].variants | array | Варианты A/B-теста; пустой список, если теста нет |
| flags[].variants[].key | string | Ключ варианта |
| flags[].variants[].weightPercent | integer | Доля пользователей варианта внутри rollout: от 0 до 100% |
| configs | array | Полный список JSON-конфигов, может быть пустым |
| configs[].key | string | Ключ конфига |
| configs[].schema | object | JSON Schema 2020-12 целиком, без ссылок на внешние схемы ($ref) |
| configs[].value | JSON | Значение конфига, которое соответствует schema |

Ключи не должны повторяться внутри одного списка. Если у флага есть варианты,
их доли в сумме дают 100%. Новая конфигурация полностью заменяет старую:
флаги и конфиги, которых в ней нет, больше не используются. При откате
передаются старые настройки, но с новым номером версии.

```json
{
  "contractVersion": 1,
  "eventId": "01JQ8X4J6W9Y7R2K5M3N1P0ABD",
  "occurredAt": "2026-09-04T10:15:30Z",
  "configurationId": "storefront",
  "version": 67,
  "flags": [
    {
      "key": "new-checkout",
      "enabled": true,
      "rolloutPercent": 25,
      "variants": [
        { "key": "control", "weightPercent": 50 },
        { "key": "test", "weightPercent": 50 }
      ]
    }
  ],
  "configs": [
    {
      "key": "checkout",
      "schema": {
        "$schema": "https://json-schema.org/draft/2020-12/schema",
        "type": "object",
        "properties": {
          "buttonText": { "type": "string" }
        },
        "required": ["buttonText"],
        "additionalProperties": false
      },
      "value": {
        "buttonText": "Оформить заказ"
      }
    }
  ]
}
```

### Событие ConfigurationChanged

Версия контракта: 1. Все поля обязательны. Событие идёт из
`Configuration Service` в `History Service` после публикации или rollback.

| Поле            | Тип              | Описание                                                  |
| --------------- | ---------------- | --------------------------------------------------------- |
| contractVersion | integer          | Версия схемы: 1                                           |
| eventId         | string           | Уникальный идентификатор события                          |
| occurredAt      | string           | Время в UTC, формат ISO 8601                              |
| configurationId | string           | Идентификатор набора настроек                             |
| version         | integer          | Номер созданной версии                                    |
| action          | string           | published или rolledBack                                  |
| sourceVersion   | integer или null | Номер исходной версии для rollback; при публикации — null |
| actorId         | string           | Идентификатор автора действия                             |

```json
{
  "contractVersion": 1,
  "eventId": "01JQ8X4J6W9Y7R2K5M3N1P0ABC",
  "occurredAt": "2026-09-04T10:15:30Z",
  "configurationId": "storefront",
  "version": 67,
  "action": "published",
  "sourceVersion": null,
  "actorId": "admin-228"
}
```
