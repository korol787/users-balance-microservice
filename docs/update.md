# Изменение баланса пользователя

Изменить баланс пользователя с указанным UUID. Если счета пользователя с указанным UUID не существует и баланс
изменяется в положительную сторону, то его счет будет создан и пополнен.

**URL** : `/v1/deposits/update`

**Метод** : `POST`

**Формат запроса**

Если параметр `amount` - положительное число, то происходит пополнение счета, иначе - списание со счета.

```json
{
  "owner_id"   : "[строка, UUID]",
  "amount"     : "[число]",
  "description": "[строка, опционально, до 100 символов]"
}
```

**Пример запроса**

```json
{
  "owner_id": "8c5593a0-37d3-11ec-8d3d-0242ac130001",
  "amount": 1000,
  "description": "VISA top-up"
}
```

## Ответ - успех

**Код** : `200 OK`

**Пример ответа**: транзакция, отражающая соответсвующее пополнение/списание со счета.

```json
{
  "id": 1,
  "sender_id": "00000000-0000-0000-0000-000000000000",
  "recipient_id": "8c5593a0-37d3-11ec-8d3d-0242ac130001",
  "amount": 1000,
  "description": "VISA top-up",
  "transaction_date": "2021-11-10T13:43:10.0899004Z"
}
```

## Ответ - ошибка

**Причина** : Параметры запроса некорректны.

**Код** : `400 BAD REQUEST`

**Пример ответа** :

```json
{
  "status": 400,
  "message": "There is some problem with the data you submitted.",
  "details": [
    {
      "field": "owner_id",
      "error": "must be a valid UUID"
    }
  ]
}
```

### Или

**Причина** : Получен запрос на списание со счета, пользователь не имеет достаточно средств.

**Код** : `403 FORBIDDEN`

**Пример ответа**

```json
{
  "status": 403,
  "message": "Insufficient funds to perform operation."
}
```
