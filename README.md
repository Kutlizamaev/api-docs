# Инструкция по подключению к BybitEye Integration API

Документ описывает интеграционные ручки BybitEye для внешней команды автоматизации P2P процессов Bybit.

Интеграция позволяет:

- отправлять в BybitEye данные по сделке и контрагенту;
- получать статистику, теги, жалобы и историю по контрагенту;
- оставлять жалобу/комментарий на контрагента;
- отправлять webhook об изменении статуса сделки.

## 1. Доступ

Production URL:

```text
https://bybiteye.tradecode.tech
```

Каждый запрос к integration API должен содержать headers:

```http
Content-Type: application/json
X-Integration-Token: <integration_token>
```

Токен выдается командой BybitEye отдельно.

Если токен не передан или неверный, API вернет `401`.

## 2. Общая схема работы

Рекомендуемый порядок интеграции:

1. При появлении/обновлении сделки отправить данные в:

```http
POST /v1/integration/bybit/order/check
```

2. Из ответа получить:

- текущий тег контрагента;
- причину тега;
- комментарии/жалобы;
- статистику по сделкам;
- личную историю внешней команды;
- историю других пользователей BybitEye;
- Telegram blacklist/comments.

3. Если нужно оставить жалобу на контрагента, отправить:

```http
POST /v1/integration/bybit/counteragent/complaint
```

4. При изменении статуса сделки отправить webhook:

```http
POST /v1/integration/bybit/order/status-webhook
```

## 3. Проверка сделки и контрагента

```http
POST /v1/integration/bybit/order/check
```

Ручка принимает данные по сделке и контрагенту.

Backend:

- сохраняет или обновляет ордер;
- обновляет данные контрагента;
- связывает ордер с сервисным аккаунтом integration token;
- возвращает информацию по контрагенту.

### Request

```json
{
  "bybitUid": 777000111,
  "counterAgentInfo": {
    "nickname": "prod-test-counterparty",
    "fullName": "Prod Test",
    "bybitId": 111222333,
    "bybitMask": "sa8dee5d1ce*******feaecbb3958794f"
  },
  "orderInfo": {
    "id": 900000000000001001,
    "tokenName": "USDT",
    "currencyId": "RUB",
    "price": 100,
    "quantity": 1,
    "amount": 100,
    "status": "40",
    "orderTime": "2026-06-26T05:00:00Z",
    "type": "SELL"
  }
}
```

### Что передавать

| Поле                         | Описание                                                                                                        |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `bybitUid`                   | UID Bybit аккаунта, с которого ваша команда работает по сделке. |
| `counterAgentInfo.nickname`  | Nickname контрагента.                                                                                           |
| `counterAgentInfo.fullName`  | Имя контрагента.                                                                                                |
| `counterAgentInfo.bybitId`   | UID контрагента на Bybit.                                                                                       |
| `counterAgentInfo.bybitMask` | Маска UID контрагента. Используется для поиска Telegram blacklist/comments.                                     |
| `orderInfo.id`               | ID сделки/ордера Bybit.                                                                                         |
| `orderInfo.tokenName`        | Название токена, например `USDT`.                                                                               |
| `orderInfo.currencyId`       | Валюта, например `RUB`.                                                                                         |
| `orderInfo.price`            | Цена сделки.                                                                                                    |
| `orderInfo.quantity`         | Количество токена.                                                                                              |
| `orderInfo.amount`           | Сумма сделки в fiat.                                                                                            |
| `orderInfo.status`           | Статус сделки. Можно передавать числовой статус Bybit или английский статус.                                    |
| `orderInfo.orderTime`        | Время сделки в ISO-8601 формате.                                                                                |
| `orderInfo.type`             | `SELL` или `BUY`.                                                                                               |

### Response

```json
{
  "bybitUid": 111222333,
  "tag": "BAD",
  "tagReason": "Мороз",
  "nickname": "prod-test-counterparty",
  "stats": {
    "totalOrders": 1,
    "totalOrderFromAccount": 1,
    "lastMonthCountExtension": 1,
    "uniqueMakerCount": 1,
    "lastOrderTime": "2026-06-26T05:00:00Z",
    "cancelledOrders": 1,
    "completedOrders": 0,
    "cancelledOrdersAccount": 1,
    "completedOrdersAccount": 0,
    "totalTeamOrders": 1,
    "canceledTeamOrders": 1,
    "completedTeamOrders": 0,
    "activeOrdersCount": 0
  },
  "teamNote": null,
  "comments": [
    {
      "from": "777000111",
      "comment": "Контрагент долго не отпускал сделку",
      "createdAt": "2026-06-28T04:00:00Z"
    }
  ],
  "personalHistory": [
    {
      "bybitUid": 777000111,
      "amount": 100,
      "orderTime": "2026-06-26T05:00:00Z",
      "status": "Отменено",
      "orderId": 900000000000001001,
      "type": "SELL",
      "wasAppealed": false
    }
  ],
  "otherHistory": [
    {
      "bybitUid": "******111",
      "amount": 100,
      "orderTime": "2026-06-26T05:00:00Z",
      "status": "Отменено",
      "type": "SELL",
      "wasAppealed": false
    }
  ],
  "telegramComments": [],
  "isInTgList": false
}
```

### Как читать ответ

| Поле                       | Описание                                                                                                  |
| -------------------------- | --------------------------------------------------------------------------------------------------------- |
| `bybitUid`                 | UID контрагента.                                                                                          |
| `tag`                      | Текущий тег контрагента: `NONE`, `BAD`, `BLACK`.                                                          |
| `tagReason`                | Причина тега.                                                                                             |
| `nickname`                 | Nickname контрагента.                                                                                     |
| `stats`                    | Общая статистика по контрагенту.                                                                          |
| `teamNote`                 | Командная заметка, если есть.                                                                             |
| `comments`                 | Жалобы/комментарии из BybitEye.                                                                           |
| `personalHistory`          | Личная история вашей команды по этому контрагенту.                                                        |
| `personalHistory.bybitUid` | UID Bybit аккаунта, с которого ваша команда проводила сделку.                                             |
| `otherHistory`             | История других пользователей BybitEye по этому контрагенту. Личные ордера вашей команды сюда не попадают. |
| `otherHistory.bybitUid`    | Замаскированный UID другого пользователя.                                                                 |
| `telegramComments`         | Комментарии из Telegram blacklist/list, найденные по UID, nickname или UID mask.                          |
| `isInTgList`               | `true`, если по контрагенту есть записи в Telegram list.                                                  |

### Пример curl

```bash
curl --location 'https://bybiteye.tradecode.tech/v1/integration/bybit/order/check' \
  --header 'Content-Type: application/json' \
  --header 'X-Integration-Token: <integration_token>' \
  --data '{
    "bybitUid": 777000111,
    "counterAgentInfo": {
      "nickname": "prod-test-counterparty",
      "fullName": "Prod Test",
      "bybitId": 111222333,
      "bybitMask": "sa8dee5d1ce*******feaecbb3958794f"
    },
    "orderInfo": {
      "id": 900000000000001001,
      "tokenName": "USDT",
      "currencyId": "RUB",
      "price": 100,
      "quantity": 1,
      "amount": 100,
      "status": "40",
      "orderTime": "2026-06-26T05:00:00Z",
      "type": "SELL"
    }
  }'
```

## 4. Оставить жалобу на контрагента

```http
POST /v1/integration/bybit/counteragent/complaint
```

Ручка создает жалобу/комментарий на контрагента.

Необходимо передавать `bybitId` контрагента.

Важно: если контрагент еще не был отправлен через `/order/check`, его может не быть в базе. В таком случае сначала нужно вызвать `/order/check`, а потом `/counteragent/complaint`.

### Request по `bybitId`

```json
{
  "bybitId": 111222333,
  "tagReason": "Мороз",
  "comment": "Контрагент долго не отпускал сделку",
  "chatLogs": []
}
```

### Response

```json
{
  "id": "6d9fd7d0-4a8e-4a7c-8c5e-4b7c7f73e6f0",
  "nickname": "prod-test-counterparty",
  "tag": "BAD",
  "tagReason": "Мороз",
  "comment": "Контрагент долго не отпускал сделку"
}
```

### Разрешенные причины жалобы

```text
Мошенник
Мороз
Грязь
Невыполнение условий
Художник
Неадекват
Комментарий
```

Можно передать пустую причину, если нужно оставить обычный комментарий без негативного тега.

### Маппинг причины в тег

| `tagReason`                   | Сохраненный `tag` |
| ----------------------------- | ----------------- |
| `Мороз`                       | `BAD`             |
| `Неадекват`                   | `BAD`             |
| `Комментарий`                 | `NONE`            |
| Пустое значение               | `NONE`            |
| Остальные разрешенные причины | `BLACK`           |

### Пример curl

```bash
curl --location 'https://bybiteye.tradecode.tech/v1/integration/bybit/counteragent/complaint' \
  --header 'Content-Type: application/json' \
  --header 'X-Integration-Token: <integration_token>' \
  --data '{
    "bybitId": 111222333,
    "tagReason": "Мороз",
    "comment": "Контрагент долго не отпускал сделку",
    "chatLogs": []
  }'
```

## 5. Webhook обновления статуса сделки

```http
POST /v1/integration/bybit/order/status-webhook
```

Ручка обновляет статус сделки, которая была ранее сохранена через integration API.

### Request

```json
{
  "orderId": 900000000000001001,
  "status": "40"
}
```

### Что передавать

| Поле       | Обязательно           | Описание                                                                                                                                        |
| ---------- | --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `orderId`  | Да                    | ID сделки/ордера Bybit.                                                                                                                         |
| `status`   | Да                    | Новый статус сделки.                                                                                                                            |

### Response

```json
{
  "orderId": 900000000000001001,
  "status": "Отменено"
}
```

### Пример curl

```bash
curl --location 'https://bybiteye.tradecode.tech/v1/integration/bybit/order/status-webhook' \
  --header 'Content-Type: application/json' \
  --header 'X-Integration-Token: <integration_token>' \
  --data '{
    "orderId": 900000000000001001,
    "status": "40"
  }'
```

## 6. Статусы сделок

Можно передавать числовые статусы Bybit:

| Code  | Значение                    |
| ----- | --------------------------- |
| `10`  | Pending Payment             |
| `20`  | Pending Coin Release        |
| `60`  | In Progress                 |
| `30`  | Appeal In Progress          |
| `90`  | Price expired               |
| `5`   | Pending On-chain            |
| `110` | Objection Validity Period   |
| `100` | Objection Under Review      |
| `40`  | Canceled / `Отменено`       |
| `50`  | Completed / `Завершено`     |
| `70`  | Transaction Failed          |
| `80`  | Abnormal order cancellation |

Также можно передавать английские статусы:

```text
Completed
Canceled
Pending Payment
Pending Coin Release
In Progress
Appeal In Progress
```

## 7. Ошибки

### Не передан токен

HTTP status:

```text
401
```

Response:

```json
{
  "msg": "Integration token is required"
}
```

### Неверный токен

HTTP status:

```text
401
```

Response:

```json
{
  "msg": "Integration token is invalid"
}
```

### Неподдерживаемая причина жалобы

HTTP status:

```text
400
```

Response:

```json
{
  "msg": "Unsupported tagReason 'Some reason'"
}
```

### Контрагент не найден по `bybitId`

HTTP status:

```text
400
```

Response:

```json
{
  "msg": "Counteragent with bybitId 111222333 was not found"
}
```

Что делать:

1. Сначала отправить контрагента через `/v1/integration/bybit/order/check`.
2. После успешного ответа повторить `/v1/integration/bybit/counteragent/complaint`.

### Неизвестный статус сделки

HTTP status:

```text
400
```

Response:

```json
{
  "msg": "Unknown order status '999'"
}
```

## 8. Рекомендации по внедрению

1. Сначала подключить `/order/check`.
2. Проверить, что в ответе приходят `tag`, `tagReason`, `comments`, `personalHistory`, `otherHistory`.
3. После этого подключить `/counteragent/complaint`.
4. Затем подключить `/order/status-webhook`.
5. Всегда передавать один и тот же `orderInfo.id` для одной сделки, чтобы backend обновлял существующий ордер, а не создавал новую запись.
6. Всегда передавать `bybitUid` в корне `/order/check`. Это UID аккаунта, с которого ваша команда работает на Bybit.
7. По возможности передавать `counterAgentInfo.bybitMask`, чтобы BybitEye мог находить Telegram blacklist/comments по маске UID.
