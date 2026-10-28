# Размещение платформы в Kubernetes

## Deployment и Service

| Компонент             | Deployment      | Service         |
| --------------------- | --------------- | --------------- |
| API Gateway           | `api-gateway`   | `api-gateway`   |
| Configuration Service | `configuration` | `configuration` |
| Delivery Service      | `delivery`      | `delivery`      |
| History Service       | `history`       | `history`       |

`Auth Service` — внешняя зависимость.

## Вход через Ingress

```mermaid
flowchart LR
    client[Админка или система-потребитель] -->|api.flags.example.com| ingress[Ingress<br/>HTTPS]
    ingress --> gateway_service[Service api-gateway]
    gateway_service --> gateway_pod[Pod API Gateway]
    gateway_pod -->|/v1/configurations/*| configuration_service[Service configuration]
    gateway_pod -->|/v1/history/*| history_service[Service history]
    gateway_pod -->|/v1/evaluations/*| delivery_service[Service delivery]
    gateway_pod -->|/v1/auth/*| auth[Внешний Auth Service]
    configuration_service --> configuration_pod[Pod Configuration Service]
    history_service --> history_pod[Pod History Service]
    delivery_service --> delivery_pod[Pod Delivery Service]
```

## ConfigMap и Secret

| Куда        | Что поместить                                                         |
| ----------- | --------------------------------------------------------------------- |
| `ConfigMap` | Адреса сервисов и уровень логирования                                 |
| `Secret`    | Пароли к базам данных и `RabbitMQ`, ключ и сертификат TLS для Ingress |

## Состояние и готовность

| Данные                                      | Где хранятся                 |
| ------------------------------------------- | ---------------------------- |
| Черновики и версии                          | База `Configuration Service` |
| История изменений                           | База `History Service`       |
| Активная версия и отметки обработки событий | База `Delivery Service`      |
| Очереди сообщений                           | `RabbitMQ` вне Pod сервисов  |

Базы сервисов и `RabbitMQ` используют долговременное хранилище, не связанное
с жизнью Pod.
Кеш ответов `Delivery Service` остаётся в памяти Pod: после пересоздания он
заполняется заново при запросах. Активная версия при этом хранится в базе.
Пока проба готовности (`readinessProbe`) не прошла, `Service` не направляет
запросы в Pod:

| Pod                     | Когда готов                           |
| ----------------------- | ------------------------------------- |
| API Gateway             | Маршруты загружены                    |
| `Configuration Service` | Доступна его база                     |
| `History Service`       | Доступна его база                     |
| `Delivery Service`      | Активная версия загружена из его базы |
