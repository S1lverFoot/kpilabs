# Лабораторна робота №3: Sequence-діаграми сценаріїв

Вихідний код усіх діаграм у форматі PlantUML: [`sequences.puml`](sequences.puml).

## Сценарій 1: Успішна оплата замовлення
```mermaid
sequenceDiagram
    autonumber
    actor Customer as Зареєстрований клієнт
    participant UI as StorefrontUI
    participant OrderSvc as OrderService
    participant CatSvc as CatalogService
    participant Gateway as PaymentGateway

    Customer->>+UI: submitOrderAndPay(cartItems, 1000)
    UI->>+OrderSvc: submitOrder(userId, cartItems)
    OrderSvc->>+CatSvc: validateStockAndPrices(cartItems)
    CatSvc-->>-OrderSvc: itemsWithPrices(available=true)
    OrderSvc->>OrderSvc: createOrder(userId, "Очікування оплати")

    opt У каталозі є акційні товари (REQ-4)
        OrderSvc->>OrderSvc: applyPromotionalDiscount(itemsWithPrices)
    end

    OrderSvc->>+Gateway: authorizePayment(orderId, 1000)
    Gateway-->>-OrderSvc: paymentApproved(transactionId)

    alt Транзакцію підтверджено
        loop Для кожної позиції замовлення
            OrderSvc->>OrderSvc: fixHistoricalUnitPrice(orderItemId, unitPrice)
            OrderSvc->>+CatSvc: decrementProductStock(productId, quantity)
            CatSvc-->>-OrderSvc: stockUpdated()
        end
        OrderSvc->>OrderSvc: updateOrderStatus(orderId, "Сплачено")
        OrderSvc-->>UI: orderPaidSuccess(orderId, "Сплачено")
    else Помилка фіксації оплати
        OrderSvc->>OrderSvc: updateOrderStatus(orderId, "Очікування оплати")
        OrderSvc-->>UI: paymentPending(orderId)
    end

    deactivate OrderSvc
    UI-->>-Customer: showOrderConfirmed("Сплачено")
```

## Сценарій 2: Відмова шлюзу
```mermaid
sequenceDiagram
    autonumber
    actor Customer as Зареєстрований клієнт
    participant UI as StorefrontUI
    participant OrderSvc as OrderService
    participant CatSvc as CatalogService
    participant Gateway as PaymentGateway

    Customer->>+UI: clickPayOrder(orderId)
    UI->>+OrderSvc: processPayment(orderId)
    OrderSvc->>+CatSvc: lockOrderItems(orderId)
    CatSvc-->>-OrderSvc: itemsLocked("Очікування оплати")
    OrderSvc->>+Gateway: authorizePayment(orderId, amount)
    Gateway-->>-OrderSvc: paymentDeclined("Declined")

    alt Банк відхилив транзакцію (Declined)
        OrderSvc->>OrderSvc: keepOrderStatus(orderId, "Очікування оплати")
        OrderSvc-->>UI: paymentDeclinedError("Очікування оплати")
    else Помилка з'єднання зі шлюзом
        OrderSvc->>OrderSvc: keepOrderStatus(orderId, "Очікування оплати")
        OrderSvc-->>UI: gatewayTimeoutError()
    end

    opt Повторна спроба оплати клієнтом
        UI-->>Customer: enableRetryPaymentButton(orderId)
    end

    deactivate OrderSvc
    UI-->>-Customer: showDeclinedWarning()
```

## Сценарій 3: Товару бракує на складі
```mermaid
sequenceDiagram
    autonumber
    actor Customer as Зареєстрований клієнт
    participant UI as StorefrontUI
    participant OrderSvc as OrderService
    participant CatSvc as CatalogService

    Customer->>+UI: submitCheckout(cartItems)
    UI->>+OrderSvc: createOrder(userId, cartItems)

    loop Для кожного товару в кошику
        OrderSvc->>+CatSvc: checkAvailableStock(productId)
        CatSvc-->>-OrderSvc: stockQuantity
    end

    alt Залишок хоча б одного товару == 0
        OrderSvc->>OrderSvc: rejectOrderCreation("OutOfStock")
        OrderSvc-->>UI: orderRejectedOutOfStock(missingItems)
    else Усі товари є в наявності
        OrderSvc->>OrderSvc: createPendingOrder(userId, "Очікування оплати")
        OrderSvc-->>UI: orderCreated(orderId)
    end

    opt Пошук доступних аналогів
        UI->>+CatSvc: findAvailableAlternatives(missingItems)
        CatSvc-->>-UI: suggestedProducts
    end

    deactivate OrderSvc
    UI-->>-Customer: showOutOfStockWarning()
```

## Сценарій 4: Повторне натискання «Оплатити»
```mermaid
sequenceDiagram
    autonumber
    actor Customer as Зареєстрований клієнт
    participant UI as StorefrontUI
    participant OrderSvc as OrderService
    participant Gateway as PaymentGateway

    Customer->>+UI: clickPayButton(orderId, idempotencyKey)
    UI->>+OrderSvc: initiatePayment(orderId, idempotencyKey)
    OrderSvc->>OrderSvc: checkTransactionState(orderId, idempotencyKey)

    alt Транзакція вже виконується (Processing)
        OrderSvc->>OrderSvc: ignoreDuplicateClick(orderId, idempotencyKey)
        OrderSvc-->>UI: currentProcessingStatus("Processing")
    else Перший запит на оплату
        OrderSvc->>OrderSvc: markOrderProcessing(orderId, "Processing")
        OrderSvc->>+Gateway: authorizePayment(orderId, amount)
        Gateway-->>-OrderSvc: paymentResult
        OrderSvc-->>UI: paymentCompleted(paymentResult)
    end

    opt Повторне натискання під час очікування відповіді
        UI->>+OrderSvc: getPaymentStatus(orderId)
        OrderSvc-->>-UI: status("Processing")
    end

    deactivate OrderSvc
    UI-->>-Customer: displayCurrentProcessingStatus()
```