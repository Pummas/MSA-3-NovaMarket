# Реестр событий Saga-хореографии оформления заказа NovaMarket

## Структура события

```json
{
  "eventType": "OrderCreated",
  "sagaId": "ea43b91f-4b91-4f9d-8c2a-7d1e5f3a9b04",
  "correlationId": "99bc11ef-a102-4d7e-9f3b-2c8d6e4a1f05",
  "timestamp": "2026-07-25T15:34:56Z",
  "payload": {
    "orderId": "7712a3c1-5e8f-4b2d-9a7c-1f4e6b8d2a93",
    "userId": "5501f7b2-3c9d-4a8e-b1f5-6d2c8a4e9b37",
    "amount": 8000,
    "sellerId": "883d4a1e-7b2c-4f9a-8e3d-5c1f7b9a2d64",
    "paymentMethod": "CARD_BOUND",
    "paymentCardId": "b7f2c9a4-1e5d-4b8c-9a3f-6d2e8b4a1c75",
    "deliveryMethod": "COURIER",
    "deliveryAddress": "г. Санкт-Петербург, Невский проспект, д. 1",
    "items": [
      {
        "productId": "c5a1e8b3-9f4d-4a2c-8b7e-1d6f3a9c5e28",
        "sku": "SKU-102",
        "name": "Наушники беспроводные",
        "price": 4000,
        "quantity": 2,
        "subtotal": 8000
      }
    ]
  }
}
```

### Ключевые поля

- `eventType` - тип события
- `sagaId` - UUID конкретного бизнес-процесса Saga. Используется участниками хореографии для восстановления контекста
  шагов.
- `correlationId` - UUID пользовательского запроса для систем мониторинга (Jaeger, OpenTelemetry).
- `orderId`, `userId`, `sellerId` - UUID сущностей домена.
- `paymentMethod` - способ оплаты: `CARD`, `CARD_BOUND` (привязанная карта → автооплата), `SBP`, `WALLET`.
- `paymentCardId` - UUID привязанной карты. Присутствует только если `paymentMethod = CARD_BOUND`.
- `deliveryMethod` - способ получения: `COURIER`, `PICKUP` (самовывоз), `PICKUP_POINT` (пункт выдачи).
- `deliveryAddress` - адрес доставки.
- `items[]` - позиции заказа:
    - `productId` - UUID товара в Catalog Service
    - `sku` - артикул продавца (Stock Keeping Unit)
    - `name` - название товара на момент заказа (снапшот)
    - `price` - цена за единицу на момент заказа (снапшот)
    - `quantity` - количество
    - `subtotal` - `price × quantity`
- `amount` - итоговая сумма заказа (`sum(subtotal)`)

> **Почему `name` и `price` внутри `items`:** Даже если продавец позже изменит цену или
> название в каталоге, заказ должен хранить ту версию, на которую покупатель согласился.

---

## 1. Успешный сценарий

| Этап бизнес-процесса        | Тип события | Название события     | Описание доменного факта                                                                         |
|:----------------------------|:------------|:---------------------|:-------------------------------------------------------------------------------------------------|
| Создание заказа             | domain      | `OrderCreated`       | Заказ провалидирован и сохранён в БД Order-сервиса в статусе «Ожидает оплаты».                   |
| Резервирование товаров      | domain      | `InventoryReserved`  | Order-сервис подтвердил резерв товара.                                                           |
| Обработка платежа           | domain      | `PaymentSucceeded`   | Платёжный шлюз подтвердил успешное списание средств выбранным способом оплаты.                   |
| Передача заказа в логистику | domain      | `DeliveryDispatched` | Логистический провайдер принял заявку и вернул трек-номер. Заказ в статусе «Передан в доставку». |
| Доставка заказа             | domain      | `DeliveryCompleted`  | Логистический провайдер подтвердил доставку. Заказ в статусе «Доставлен».                        |

---

## 2. Сценарии с ошибками

| Этап бизнес-процесса             | Тип события | Название события             | Описание доменного факта                                                        |
|:---------------------------------|:------------|:-----------------------------|:--------------------------------------------------------------------------------|
| Проверка остатков на складе      | failure     | `InventoryReservationFailed` | На складе отсутствует нужное количество товара.                                 |
| Авторизация платежа              | failure     | `PaymentFailed`              | Ошибка списания средств (недостаточно денег, блокировка карты, отказ банка).    |
| Ожидание ответа платёжного шлюза | timeout     | `PaymentGatewayTimeout`      | Превышено максимальное время ожидания ответа от внешнего платёжного шлюза.      |
| Интеграция с логистикой          | failure     | `DeliveryRequestFailed`      | Внешняя служба доставки отклонила адрес, габариты или не смогла принять заявку. |

---

## 3. Компенсационные действия

| Этап бизнес-процесса                   | Тип события  | Название события          | Описание доменного факта                                                      |
|:---------------------------------------|:-------------|:--------------------------|:------------------------------------------------------------------------------|
| Освобождение зарезервированных товаров | compensation | `InventoryReleased`       | Ранее зарезервированные товары возвращены в доступный остаток склада.         |
| Возврат средств покупателю             | compensation | `RefundSucceeded`         | Инициирован и подтверждён возврат денежных средств на исходный способ оплаты. |
| Фиксация сбоя процесса                 | compensation | `OrderCancelled`          | Финальная точка Саги при ошибке: заказ переведён в статус «Отменён».          |
| Оповещение о неудаче                   | compensation | `ErrorNotificationPushed` | Покупателю отправлено Push-уведомление с причиной отмены заказа.              |

---

## Диаграмма переходов

```text
Успешный сценарий:
OrderCreated → InventoryReserved → PaymentSucceeded → DeliveryDispatched → DeliveryCompleted

Ошибка 1 - нет товара:
OrderCreated → InventoryReservationFailed → OrderCancelled

Ошибка 2 - платёж не прошёл:
OrderCreated → InventoryReserved → PaymentFailed
    → InventoryReleased (compensation) → OrderCancelled

Ошибка 2b - таймаут платёжного шлюза:
OrderCreated → InventoryReserved → PaymentGatewayTimeout
    → InventoryReleased (compensation) → OrderCancelled

Ошибка 3 - логистика отказала:
OrderCreated → InventoryReserved → PaymentSucceeded → DeliveryRequestFailed
    → RefundSucceeded     (compensation)
    → InventoryReleased   (compensation)
    → OrderCancelled
```