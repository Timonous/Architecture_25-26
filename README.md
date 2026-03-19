# **Либенсон А. М. РИС-22-1**

# **Проектирование архитектуры программных систем**

# ***Лабораторная работа №4***

## 

## **Тема:**

## Проектирование REST API

## **Цель:**

## Получить опыт проектирования программного интерфейса.

---

## 1. Проектные решения

### Решение 1 — Архитектурный стиль: REST

**Описание:** API реализован в стиле REST (Representational State Transfer). Каждый ресурс идентифицируется URL-адресом, а действия над ним определяются HTTP-методами (GET, POST, PUT, DELETE).

**Обоснование:** REST является стандартом де-факто для публичных и внутренних API микросервисов. Он хорошо поддерживается мобильными клиентами (React Native), веб-клиентами (React) и легко интегрируется с API Gateway (Kong / AWS API Gateway).

---

### Решение 2 — Версионирование через URL-префикс

**Описание:** Все эндпоинты API содержат префикс версии: `/api/v1/...`. При выходе новой версии добавляется префикс `/api/v2/...`, старая версия продолжает работать параллельно в течение переходного периода.

**Обоснование:** Версионирование через URL — наиболее очевидный и явный способ, понятный без изучения документации. Он позволяет клиентам (мобильным приложениям) постепенно мигрировать на новую версию без принудительного обновления.

**Пример:**
```
/api/v1/orders      — текущая версия
/api/v2/orders      — будущая версия (при необходимости)
```

---

### Решение 3 — Формат данных: JSON

**Описание:** Все запросы и ответы используют формат JSON (Content-Type: `application/json`). Клиент обязан передавать заголовок `Content-Type: application/json` в запросах с телом (POST, PUT). Сервер всегда возвращает `Content-Type: application/json`.

**Обоснование:** JSON — универсальный формат для веб и мобильных приложений, нативно поддерживается JavaScript/TypeScript. Альтернативы (XML, Protobuf) избыточны для данного сценария.

---

### Решение 4 — Аутентификация через JWT Bearer Token

**Описание:** Все эндпоинты (кроме публичных) требуют передачи JWT-токена в заголовке `Authorization`. Токен проверяется на уровне API Gateway до передачи запроса в сервис заказов.

**Формат заголовка:**
```
Authorization: Bearer <jwt_token>
```

**Обоснование:** JWT позволяет хранить роль пользователя (`customer`, `kitchen_staff`, `franchise_owner`) непосредственно в токене, что исключает необходимость дополнительного запроса к сервису аутентификации при каждом вызове. Это соответствует принципу stateless REST.

**Структура payload JWT:**
```json
{
  "sub": "user-uuid",
  "role": "customer",
  "franchiseId": null,
  "exp": 1700000000
}
```

---

### Решение 5 — Единая структура ответов

**Описание:** Все ответы API, как успешные, так и ошибочные, возвращаются в единой обёртке. Это упрощает обработку ответов на клиенте.

**Формат успешного ответа:**
```json
{
  "success": true,
  "data": { ... },
  "meta": {
    "timestamp": "2025-03-18T10:00:00Z"
  }
}
```

**Формат ответа с пагинацией:**
```json
{
  "success": true,
  "data": [ ... ],
  "meta": {
    "timestamp": "2025-03-18T10:00:00Z",
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 150,
      "totalPages": 8
    }
  }
}
```

**Формат ответа с ошибкой:**
```json
{
  "success": false,
  "error": {
    "code": "ORDER_NOT_FOUND",
    "message": "Заказ с указанным ID не найден",
    "details": null
  },
  "meta": {
    "timestamp": "2025-03-18T10:00:00Z"
  }
}
```

**Обоснование:** Единая структура позволяет клиенту всегда обращаться к `response.data` для получения данных и к `response.error` для обработки ошибок, не меняя логику парсинга в зависимости от эндпоинта.

---

### Решение 6 — Использование UUID как идентификаторов ресурсов

**Описание:** Все идентификаторы ресурсов (заказы, пользователи, позиции) представляются в формате UUID v4.

**Пример:** `3fa85f64-5717-4562-b3fc-2c963f66afa6`

**Обоснование:** UUID исключает возможность угадывания идентификаторов (в отличие от автоинкрементных числовых ID), что повышает безопасность. Кроме того, UUID можно генерировать на клиенте без обращения к серверу, что упрощает оптимистичные обновления в мобильном приложении.

---

### Решение 7 — Коды HTTP-статусов по семантике операции

**Описание:** API использует стандартные HTTP-статусы в строгом соответствии с семантикой операции.

| Код | Ситуация |
|-----|----------|
| `200 OK` | Успешный GET, PUT |
| `201 Created` | Успешный POST (создание ресурса) |
| `204 No Content` | Успешный DELETE |
| `400 Bad Request` | Ошибка валидации входных данных |
| `401 Unauthorized` | Отсутствует или истёк JWT-токен |
| `403 Forbidden` | Токен валиден, но роль не имеет прав |
| `404 Not Found` | Ресурс не найден |
| `409 Conflict` | Конфликт (например, недопустимый переход статуса) |
| `422 Unprocessable Entity` | Данные корректны по формату, но не по бизнес-логике |
| `500 Internal Server Error` | Непредвиденная ошибка сервера |

**Обоснование:** Корректные HTTP-статусы позволяют API Gateway, мобильным клиентам и системам мониторинга реагировать на ответы без разбора тела ответа.

---

### Решение 8 — Пагинация через query-параметры

**Описание:** Все эндпоинты, возвращающие списки, поддерживают пагинацию через query-параметры `page` и `limit`. Значения по умолчанию: `page=1`, `limit=20`. Максимальное значение `limit=100`.

**Пример запроса:**
```
GET /api/v1/orders?page=2&limit=10
```

**Обоснование:** Ограничение выборки защищает сервер от запросов, возвращающих тысячи записей. Для мобильного приложения пагинация обеспечивает быструю загрузку первого экрана истории заказов.

---

### Решение 9 — Разграничение доступа на основе ролей (RBAC)

**Описание:** Каждый эндпоинт доступен только определённым ролям. Роль извлекается из JWT-токена. Попытка вызова эндпоинта с недостаточными правами возвращает `403 Forbidden`.

| Роль | Доступные операции |
|------|--------------------|
| `customer` | Создать заказ, просмотреть свои заказы |
| `kitchen_staff` | Просмотреть заказы франшизы, обновить статус заказа |
| `franchise_owner` | Просмотреть все заказы франшизы, назначить курьера |
| `courier` | Просмотреть назначенные заказы |

**Обоснование:** RBAC реализован на уровне самого сервиса (а не только на уровне Gateway), что обеспечивает защиту даже при обходе Gateway во внутренней сети.

---

## 2. Общие соглашения API

**Base URL:** `https://api.sandwich-network.com/api/v1`

**Обязательные заголовки для всех запросов:**

| Заголовок | Значение | Обязателен |
|-----------|----------|------------|
| `Authorization` | `Bearer <jwt_token>` | Да (кроме публичных) |
| `Content-Type` | `application/json` | Да (для POST, PUT) |
| `Accept` | `application/json` | Рекомендуется |

**Формат дат:** ISO 8601 (`2025-03-18T10:30:00Z`)

**Коды ошибок бизнес-логики:**

| Код | Описание |
|-----|----------|
| `ORDER_NOT_FOUND` | Заказ не найден |
| `INVALID_STATUS_TRANSITION` | Недопустимый переход статуса |
| `ORDER_ALREADY_PAID` | Заказ уже оплачен |
| `FRANCHISE_NOT_FOUND` | Франшиза не найдена |
| `VALIDATION_ERROR` | Ошибка валидации полей |
| `ACCESS_DENIED` | Недостаточно прав для операции |

---

## 3. Описание эндпоинтов

### 3.1 Создание заказа

**Метод:** `POST`  
**URL:** `/api/v1/orders`  
**Роли:** `customer`  
**Описание:** Создаёт новый заказ. Статус устанавливается автоматически как `new`. После создания в брокер сообщений публикуется событие `order.created`.

**Заголовки запроса:**
```
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

**Тело запроса:**
```json
{
  "franchiseId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "deliveryType": "delivery",
  "deliveryAddress": "ул. Пушкина, д. 10, кв. 5, Москва",
  "items": [
    {
      "menuItemId": "a1b2c3d4-1111-2222-3333-444455556666",
      "quantity": 2,
      "unitPrice": 350.00
    },
    {
      "menuItemId": "b2c3d4e5-1111-2222-3333-444455557777",
      "quantity": 1,
      "unitPrice": 120.00
    }
  ]
}
```

**Параметры тела запроса:**

| Поле | Тип | Обязательное | Описание |
|------|-----|:---:|---------|
| `franchiseId` | `string (UUID)` | ✓ | ID магазина, в котором размещается заказ |
| `deliveryType` | `enum: pickup \| delivery` | ✓ | Способ получения: самовывоз или доставка |
| `deliveryAddress` | `string` | Только при `delivery` | Адрес доставки, макс. 500 символов |
| `items` | `array` | ✓ | Список позиций заказа, минимум 1 элемент |
| `items[].menuItemId` | `string (UUID)` | ✓ | ID позиции меню |
| `items[].quantity` | `integer` | ✓ | Количество, от 1 до 99 |
| `items[].unitPrice` | `number` | ✓ | Цена за единицу на момент заказа (в рублях) |

**Успешный ответ `201 Created`:**
```json
{
  "success": true,
  "data": {
    "id": "9c1b2a3d-4444-5555-6666-777788889999",
    "customerId": "user-uuid-here",
    "franchiseId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "status": "new",
    "deliveryType": "delivery",
    "deliveryAddress": "ул. Пушкина, д. 10, кв. 5, Москва",
    "totalPrice": 820.00,
    "items": [
      {
        "id": "item-uuid-1",
        "menuItemId": "a1b2c3d4-1111-2222-3333-444455556666",
        "quantity": 2,
        "unitPrice": 350.00,
        "subtotal": 700.00
      },
      {
        "id": "item-uuid-2",
        "menuItemId": "b2c3d4e5-1111-2222-3333-444455557777",
        "quantity": 1,
        "unitPrice": 120.00,
        "subtotal": 120.00
      }
    ],
    "createdAt": "2025-03-18T10:30:00Z",
    "updatedAt": "2025-03-18T10:30:00Z"
  },
  "meta": {
    "timestamp": "2025-03-18T10:30:00Z"
  }
}
```

**Ответ при ошибке валидации `400 Bad Request`:**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Ошибка валидации входных данных",
    "details": [
      { "field": "items", "message": "Список позиций не может быть пустым" },
      { "field": "deliveryAddress", "message": "Адрес доставки обязателен при типе delivery" }
    ]
  },
  "meta": { "timestamp": "2025-03-18T10:30:00Z" }
}
```

---

### 3.2 Получение заказа по ID

**Метод:** `GET`  
**URL:** `/api/v1/orders/:orderId`  
**Роли:** `customer` (только свои заказы), `kitchen_staff`, `franchise_owner`, `courier`  
**Описание:** Возвращает полную информацию о заказе по его UUID. Покупатель может получить только свой заказ; сотрудники кухни и владельцы — только заказы своей франшизы.

**Параметры пути:**

| Параметр | Тип | Описание |
|----------|-----|---------|
| `orderId` | `string (UUID)` | Уникальный идентификатор заказа |

**Пример запроса:**
```
GET /api/v1/orders/9c1b2a3d-4444-5555-6666-777788889999
Authorization: Bearer <jwt_token>
```

**Успешный ответ `200 OK`:**
```json
{
  "success": true,
  "data": {
    "id": "9c1b2a3d-4444-5555-6666-777788889999",
    "customerId": "user-uuid-here",
    "franchiseId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "courierId": null,
    "status": "preparing",
    "deliveryType": "delivery",
    "deliveryAddress": "ул. Пушкина, д. 10, кв. 5, Москва",
    "totalPrice": 820.00,
    "items": [
      {
        "id": "item-uuid-1",
        "menuItemId": "a1b2c3d4-1111-2222-3333-444455556666",
        "menuItemName": "Классический сэндвич",
        "quantity": 2,
        "unitPrice": 350.00,
        "subtotal": 700.00
      }
    ],
    "createdAt": "2025-03-18T10:30:00Z",
    "updatedAt": "2025-03-18T10:45:00Z"
  },
  "meta": { "timestamp": "2025-03-18T10:50:00Z" }
}
```

**Ответ при отсутствии заказа `404 Not Found`:**
```json
{
  "success": false,
  "error": {
    "code": "ORDER_NOT_FOUND",
    "message": "Заказ с указанным ID не найден",
    "details": null
  },
  "meta": { "timestamp": "2025-03-18T10:50:00Z" }
}
```

---

### 3.3 Получение списка заказов покупателя

**Метод:** `GET`  
**URL:** `/api/v1/orders`  
**Роли:** `customer`  
**Описание:** Возвращает историю заказов текущего авторизованного покупателя с поддержкой пагинации и фильтрации по статусу. Покупатель видит только свои заказы (ID извлекается из JWT).

**Query-параметры:**

| Параметр | Тип | Обязательный | По умолчанию | Описание |
|----------|-----|:---:|:---:|---------|
| `page` | `integer` | — | `1` | Номер страницы, минимум 1 |
| `limit` | `integer` | — | `20` | Размер страницы, от 1 до 100 |
| `status` | `enum` | — | все | Фильтр по статусу: `new`, `accepted`, `preparing`, `ready`, `issued` |

**Пример запроса:**
```
GET /api/v1/orders?page=1&limit=5&status=preparing
Authorization: Bearer <jwt_token>
```

**Успешный ответ `200 OK`:**
```json
{
  "success": true,
  "data": [
    {
      "id": "9c1b2a3d-4444-5555-6666-777788889999",
      "franchiseId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "status": "preparing",
      "deliveryType": "delivery",
      "totalPrice": 820.00,
      "itemsCount": 2,
      "createdAt": "2025-03-18T10:30:00Z"
    }
  ],
  "meta": {
    "timestamp": "2025-03-18T10:50:00Z",
    "pagination": {
      "page": 1,
      "limit": 5,
      "total": 1,
      "totalPages": 1
    }
  }
}
```

---

### 3.4 Получение списка заказов франшизы

**Метод:** `GET`  
**URL:** `/api/v1/franchises/:franchiseId/orders`  
**Роли:** `kitchen_staff`, `franchise_owner`  
**Описание:** Возвращает список заказов конкретного магазина. Используется на панели кухни и в административной панели. Поддерживает фильтрацию по статусу и пагинацию.

**Параметры пути:**

| Параметр | Тип | Описание |
|----------|-----|---------|
| `franchiseId` | `string (UUID)` | ID магазина франшизы |

**Query-параметры:**

| Параметр | Тип | Обязательный | По умолчанию | Описание |
|----------|-----|:---:|:---:|---------|
| `page` | `integer` | — | `1` | Номер страницы |
| `limit` | `integer` | — | `20` | Размер страницы, от 1 до 100 |
| `status` | `enum` | — | все | Фильтр по статусу заказа |
| `deliveryType` | `enum` | — | все | Фильтр: `pickup` или `delivery` |

**Пример запроса:**
```
GET /api/v1/franchises/3fa85f64-5717-4562-b3fc-2c963f66afa6/orders?status=ready&limit=10
Authorization: Bearer <jwt_token>
```

**Успешный ответ `200 OK`:**
```json
{
  "success": true,
  "data": [
    {
      "id": "9c1b2a3d-4444-5555-6666-777788889999",
      "customerId": "user-uuid-here",
      "status": "ready",
      "deliveryType": "pickup",
      "totalPrice": 350.00,
      "itemsCount": 1,
      "createdAt": "2025-03-18T11:00:00Z",
      "updatedAt": "2025-03-18T11:20:00Z"
    }
  ],
  "meta": {
    "timestamp": "2025-03-18T11:25:00Z",
    "pagination": {
      "page": 1,
      "limit": 10,
      "total": 1,
      "totalPages": 1
    }
  }
}
```

---

### 3.5 Обновление статуса заказа

**Метод:** `PUT`  
**URL:** `/api/v1/orders/:orderId/status`  
**Роли:** `kitchen_staff`, `franchise_owner`  
**Описание:** Изменяет статус заказа согласно допустимым переходам жизненного цикла. При успешном обновлении публикуется событие `order.status_changed` в брокер сообщений, что инициирует отправку push-уведомления покупателю.

**Допустимые переходы статусов:**

```
new → accepted → preparing → ready → issued
```

Переход возможен только на следующий статус по цепочке. Обратные переходы запрещены.

**Параметры пути:**

| Параметр | Тип | Описание |
|----------|-----|---------|
| `orderId` | `string (UUID)` | ID заказа |

**Тело запроса:**
```json
{
  "status": "preparing"
}
```

| Поле | Тип | Обязательное | Описание |
|------|-----|:---:|---------|
| `status` | `enum` | ✓ | Новый статус: `accepted`, `preparing`, `ready`, `issued` |

**Успешный ответ `200 OK`:**
```json
{
  "success": true,
  "data": {
    "id": "9c1b2a3d-4444-5555-6666-777788889999",
    "previousStatus": "accepted",
    "status": "preparing",
    "updatedAt": "2025-03-18T11:10:00Z"
  },
  "meta": { "timestamp": "2025-03-18T11:10:00Z" }
}
```

**Ответ при недопустимом переходе `409 Conflict`:**
```json
{
  "success": false,
  "error": {
    "code": "INVALID_STATUS_TRANSITION",
    "message": "Недопустимый переход статуса: из 'ready' в 'accepted'",
    "details": {
      "currentStatus": "ready",
      "requestedStatus": "accepted",
      "allowedNextStatus": "issued"
    }
  },
  "meta": { "timestamp": "2025-03-18T11:10:00Z" }
}
```

---

### 3.6 Назначение курьера на заказ

**Метод:** `PUT`  
**URL:** `/api/v1/orders/:orderId/courier`  
**Роли:** `franchise_owner`  
**Описание:** Назначает курьера на заказ с типом `delivery`. Может быть вызван только для заказов со статусом `ready`. После назначения курьер получает задание в своём мобильном приложении.

**Параметры пути:**

| Параметр | Тип | Описание |
|----------|-----|---------|
| `orderId` | `string (UUID)` | ID заказа |

**Тело запроса:**
```json
{
  "courierId": "courier-uuid-here"
}
```

| Поле | Тип | Обязательное | Описание |
|------|-----|:---:|---------|
| `courierId` | `string (UUID)` | ✓ | ID пользователя с ролью `courier` |

**Успешный ответ `200 OK`:**
```json
{
  "success": true,
  "data": {
    "id": "9c1b2a3d-4444-5555-6666-777788889999",
    "courierId": "courier-uuid-here",
    "status": "ready",
    "updatedAt": "2025-03-18T11:30:00Z"
  },
  "meta": { "timestamp": "2025-03-18T11:30:00Z" }
}
```

**Ответ при ошибке `422 Unprocessable Entity`:**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Курьер может быть назначен только на заказ типа delivery со статусом ready",
    "details": {
      "orderId": "9c1b2a3d-4444-5555-6666-777788889999",
      "currentDeliveryType": "pickup",
      "currentStatus": "preparing"
    }
  },
  "meta": { "timestamp": "2025-03-18T11:30:00Z" }
}
```

---

### 3.7 Отмена заказа

**Метод:** `DELETE`  
**URL:** `/api/v1/orders/:orderId`  
**Роли:** `customer` (только свои заказы, только статус `new`), `franchise_owner`  
**Описание:** Отменяет заказ. Покупатель может отменить только заказ в статусе `new` (до принятия кухней). Владелец франшизы может отменить заказ в статусах `new` и `accepted`. Публикуется событие `order.cancelled`.

**Параметры пути:**

| Параметр | Тип | Описание |
|----------|-----|---------|
| `orderId` | `string (UUID)` | ID заказа |

**Пример запроса:**
```
DELETE /api/v1/orders/9c1b2a3d-4444-5555-6666-777788889999
Authorization: Bearer <jwt_token>
```

**Успешный ответ `204 No Content`:**
```
HTTP/1.1 204 No Content
```
*(тело ответа отсутствует)*

**Ответ при недопустимой отмене `409 Conflict`:**
```json
{
  "success": false,
  "error": {
    "code": "INVALID_STATUS_TRANSITION",
    "message": "Заказ в статусе 'preparing' не может быть отменён покупателем",
    "details": {
      "currentStatus": "preparing",
      "allowedStatuses": ["new"]
    }
  },
  "meta": { "timestamp": "2025-03-18T11:40:00Z" }
}
```

---

## 4. Реализация API

### Стек технологий

- **Runtime:** Node.js 20 LTS
- **Фреймворк:** Express 4 + TypeScript
- **Валидация:** Zod
- **БД:** PostgreSQL (через абстракцию репозитория)

---

### 4.1 Типы и интерфейсы

```typescript
// src/types/order.types.ts

export type OrderStatus = "new" | "accepted" | "preparing" | "ready" | "issued";
export type DeliveryType = "pickup" | "delivery";
export type UserRole = "customer" | "kitchen_staff" | "franchise_owner" | "courier";

export interface JwtPayload {
  sub: string;          // userId
  role: UserRole;
  franchiseId: string | null;
}

export interface OrderItem {
  id: string;
  menuItemId: string;
  quantity: number;
  unitPrice: number;
  subtotal: number;
}

export interface Order {
  id: string;
  customerId: string;
  franchiseId: string;
  courierId: string | null;
  status: OrderStatus;
  deliveryType: DeliveryType;
  deliveryAddress: string | null;
  totalPrice: number;
  items: OrderItem[];
  createdAt: Date;
  updatedAt: Date;
}

// Интерфейсы репозитория и публикатора (DIP)
export interface IOrderRepository {
  save(order: Omit<Order, "id" | "createdAt" | "updatedAt">): Promise<Order>;
  findById(id: string): Promise<Order | null>;
  findByCustomer(customerId: string, page: number, limit: number, status?: OrderStatus): Promise<{ data: Order[]; total: number }>;
  findByFranchise(franchiseId: string, page: number, limit: number, status?: OrderStatus, deliveryType?: DeliveryType): Promise<{ data: Order[]; total: number }>;
  updateStatus(id: string, status: OrderStatus): Promise<Order>;
  assignCourier(id: string, courierId: string): Promise<Order>;
  delete(id: string): Promise<void>;
}

export interface IEventPublisher {
  publish(event: string, payload: Record<string, unknown>): Promise<void>;
}
```

---

### 4.2 Утилиты: формирование ответов и пагинация

```typescript
// src/utils/response.ts

import { Response } from "express";

interface ApiMeta {
  timestamp: string;
  pagination?: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
  };
}

// DRY: единая функция формирования успешного ответа
export function sendSuccess<T>(
  res: Response,
  data: T,
  statusCode: number = 200,
  pagination?: { page: number; limit: number; total: number }
): void {
  const meta: ApiMeta = { timestamp: new Date().toISOString() };

  if (pagination) {
    meta.pagination = {
      ...pagination,
      totalPages: Math.ceil(pagination.total / pagination.limit),
    };
  }

  res.status(statusCode).json({ success: true, data, meta });
}

// DRY: единая функция формирования ответа с ошибкой
export function sendError(
  res: Response,
  statusCode: number,
  code: string,
  message: string,
  details: unknown = null
): void {
  res.status(statusCode).json({
    success: false,
    error: { code, message, details },
    meta: { timestamp: new Date().toISOString() },
  });
}
```

---

### 4.3 Middleware: аутентификация и авторизация

```typescript
// src/middleware/auth.ts

import { Request, Response, NextFunction } from "express";
import jwt from "jsonwebtoken";
import { JwtPayload, UserRole } from "../types/order.types";
import { sendError } from "../utils/response";

// Расширение типа Request для хранения данных пользователя
declare global {
  namespace Express {
    interface Request {
      user?: JwtPayload;
    }
  }
}

// SRP: middleware отвечает только за проверку токена
export function authenticate(req: Request, res: Response, next: NextFunction): void {
  const authHeader = req.headers.authorization;

  if (!authHeader?.startsWith("Bearer ")) {
    sendError(res, 401, "UNAUTHORIZED", "Требуется авторизация");
    return;
  }

  const token = authHeader.slice(7);

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET!) as JwtPayload;
    req.user = payload;
    next();
  } catch {
    sendError(res, 401, "UNAUTHORIZED", "Токен недействителен или истёк");
  }
}

// OCP: функция расширяема — достаточно передать новые роли
export function authorize(...roles: UserRole[]) {
  return (req: Request, res: Response, next: NextFunction): void => {
    if (!req.user || !roles.includes(req.user.role)) {
      sendError(res, 403, "ACCESS_DENIED", "Недостаточно прав для выполнения операции");
      return;
    }
    next();
  };
}
```

---

### 4.4 Валидация схем входных данных (Zod)

```typescript
// src/validators/order.validators.ts

import { z } from "zod";

// KISS: схемы валидации описывают структуру лаконично и читаемо
export const createOrderSchema = z.object({
  franchiseId: z.string().uuid("franchiseId должен быть валидным UUID"),
  deliveryType: z.enum(["pickup", "delivery"]),
  deliveryAddress: z.string().max(500).optional(),
  items: z
    .array(
      z.object({
        menuItemId: z.string().uuid(),
        quantity: z.number().int().min(1).max(99),
        unitPrice: z.number().positive(),
      })
    )
    .min(1, "Список позиций не может быть пустым"),
}).refine(
  (data) => data.deliveryType === "pickup" || !!data.deliveryAddress,
  { message: "Адрес доставки обязателен при типе delivery", path: ["deliveryAddress"] }
);

export const updateStatusSchema = z.object({
  status: z.enum(["accepted", "preparing", "ready", "issued"]),
});

export const assignCourierSchema = z.object({
  courierId: z.string().uuid("courierId должен быть валидным UUID"),
});

export const paginationSchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  status: z.enum(["new", "accepted", "preparing", "ready", "issued"]).optional(),
  deliveryType: z.enum(["pickup", "delivery"]).optional(),
});
```

---

### 4.5 Бизнес-логика: сервис заказов

```typescript
// src/services/OrderService.ts

import { v4 as uuidv4 } from "uuid";
import {
  Order, OrderStatus, IOrderRepository, IEventPublisher, DeliveryType
} from "../types/order.types";

// Допустимые переходы статусов (KISS)
const STATUS_TRANSITIONS: Partial<Record<OrderStatus, OrderStatus>> = {
  new:       "accepted",
  accepted:  "preparing",
  preparing: "ready",
  ready:     "issued",
};

// SRP: класс отвечает только за бизнес-логику заказов
// DIP: зависит от интерфейсов, а не от конкретных реализаций
export class OrderService {
  constructor(
    private readonly repo: IOrderRepository,
    private readonly publisher: IEventPublisher
  ) {}

  // POST /orders
  async createOrder(dto: {
    customerId: string;
    franchiseId: string;
    deliveryType: DeliveryType;
    deliveryAddress?: string;
    items: { menuItemId: string; quantity: number; unitPrice: number }[];
  }): Promise<Order> {
    const totalPrice = dto.items.reduce(
      (sum, item) => sum + item.unitPrice * item.quantity,
      0
    );

    const order = await this.repo.save({
      ...dto,
      courierId: null,
      status: "new",
      totalPrice,
      deliveryAddress: dto.deliveryAddress ?? null,
      items: dto.items.map((item) => ({
        id: uuidv4(),
        ...item,
        subtotal: item.unitPrice * item.quantity,
      })),
    });

    await this.publisher.publish("order.created", {
      orderId: order.id,
      customerId: order.customerId,
      franchiseId: order.franchiseId,
    });

    return order;
  }

  // GET /orders/:orderId
  async getOrderById(orderId: string, requesterId: string, requesterRole: string): Promise<Order> {
    const order = await this.repo.findById(orderId);

    if (!order) {
      throw { code: "ORDER_NOT_FOUND", status: 404, message: "Заказ не найден" };
    }

    // Покупатель видит только свои заказы
    if (requesterRole === "customer" && order.customerId !== requesterId) {
      throw { code: "ACCESS_DENIED", status: 403, message: "Нет доступа к этому заказу" };
    }

    return order;
  }

  // GET /orders (история покупателя)
  async getCustomerOrders(
    customerId: string,
    page: number,
    limit: number,
    status?: OrderStatus
  ): Promise<{ data: Order[]; total: number }> {
    return this.repo.findByCustomer(customerId, page, limit, status);
  }

  // GET /franchises/:franchiseId/orders
  async getFranchiseOrders(
    franchiseId: string,
    page: number,
    limit: number,
    status?: OrderStatus,
    deliveryType?: DeliveryType
  ): Promise<{ data: Order[]; total: number }> {
    return this.repo.findByFranchise(franchiseId, page, limit, status, deliveryType);
  }

  // PUT /orders/:orderId/status
  async updateStatus(orderId: string, newStatus: OrderStatus): Promise<Order> {
    const order = await this.repo.findById(orderId);

    if (!order) {
      throw { code: "ORDER_NOT_FOUND", status: 404, message: "Заказ не найден" };
    }

    const allowedNext = STATUS_TRANSITIONS[order.status];

    if (allowedNext !== newStatus) {
      throw {
        code: "INVALID_STATUS_TRANSITION",
        status: 409,
        message: `Недопустимый переход статуса: из '${order.status}' в '${newStatus}'`,
        details: { currentStatus: order.status, requestedStatus: newStatus, allowedNextStatus: allowedNext ?? null },
      };
    }

    const updated = await this.repo.updateStatus(orderId, newStatus);

    await this.publisher.publish("order.status_changed", {
      orderId: updated.id,
      customerId: updated.customerId,
      previousStatus: order.status,
      newStatus,
    });

    return updated;
  }

  // PUT /orders/:orderId/courier
  async assignCourier(orderId: string, courierId: string): Promise<Order> {
    const order = await this.repo.findById(orderId);

    if (!order) {
      throw { code: "ORDER_NOT_FOUND", status: 404, message: "Заказ не найден" };
    }

    if (order.deliveryType !== "delivery" || order.status !== "ready") {
      throw {
        code: "VALIDATION_ERROR",
        status: 422,
        message: "Курьер может быть назначен только на заказ типа delivery со статусом ready",
        details: { currentDeliveryType: order.deliveryType, currentStatus: order.status },
      };
    }

    return this.repo.assignCourier(orderId, courierId);
  }

  // DELETE /orders/:orderId
  async cancelOrder(orderId: string, requesterId: string, requesterRole: string): Promise<void> {
    const order = await this.repo.findById(orderId);

    if (!order) {
      throw { code: "ORDER_NOT_FOUND", status: 404, message: "Заказ не найден" };
    }

    const allowedStatuses: OrderStatus[] =
      requesterRole === "customer" ? ["new"] : ["new", "accepted"];

    if (!allowedStatuses.includes(order.status)) {
      throw {
        code: "INVALID_STATUS_TRANSITION",
        status: 409,
        message: `Заказ в статусе '${order.status}' не может быть отменён`,
        details: { currentStatus: order.status, allowedStatuses },
      };
    }

    if (requesterRole === "customer" && order.customerId !== requesterId) {
      throw { code: "ACCESS_DENIED", status: 403, message: "Нет доступа к этому заказу" };
    }

    await this.repo.delete(orderId);

    await this.publisher.publish("order.cancelled", {
      orderId,
      cancelledBy: requesterId,
    });
  }
}
```

---

### 4.6 Контроллер и маршруты

```typescript
// src/controllers/OrderController.ts

import { Request, Response } from "express";
import { OrderService } from "../services/OrderService";
import { sendSuccess, sendError } from "../utils/response";
import {
  createOrderSchema,
  updateStatusSchema,
  assignCourierSchema,
  paginationSchema,
} from "../validators/order.validators";
import { OrderStatus } from "../types/order.types";

// SRP: контроллер отвечает только за HTTP-слой (парсинг, вызов сервиса, ответ)
export class OrderController {
  constructor(private readonly orderService: OrderService) {}

  // POST /api/v1/orders
  createOrder = async (req: Request, res: Response): Promise<void> => {
    const parsed = createOrderSchema.safeParse(req.body);

    if (!parsed.success) {
      sendError(res, 400, "VALIDATION_ERROR", "Ошибка валидации входных данных",
        parsed.error.errors.map((e) => ({ field: e.path.join("."), message: e.message }))
      );
      return;
    }

    try {
      const order = await this.orderService.createOrder({
        customerId: req.user!.sub,
        ...parsed.data,
      });
      sendSuccess(res, order, 201);
    } catch (err: any) {
      sendError(res, err.status ?? 500, err.code ?? "INTERNAL_ERROR", err.message, err.details);
    }
  };

  // GET /api/v1/orders/:orderId
  getOrderById = async (req: Request, res: Response): Promise<void> => {
    try {
      const order = await this.orderService.getOrderById(
        req.params.orderId,
        req.user!.sub,
        req.user!.role
      );
      sendSuccess(res, order);
    } catch (err: any) {
      sendError(res, err.status ?? 500, err.code ?? "INTERNAL_ERROR", err.message, err.details);
    }
  };

  // GET /api/v1/orders
  getCustomerOrders = async (req: Request, res: Response): Promise<void> => {
    const parsed = paginationSchema.safeParse(req.query);

    if (!parsed.success) {
      sendError(res, 400, "VALIDATION_ERROR", "Некорректные параметры запроса");
      return;
    }

    try {
      const { page, limit, status } = parsed.data;
      const result = await this.orderService.getCustomerOrders(
        req.user!.sub, page, limit, status as OrderStatus
      );
      sendSuccess(res, result.data, 200, { page, limit, total: result.total });
    } catch (err: any) {
      sendError(res, err.status ?? 500, err.code ?? "INTERNAL_ERROR", err.message);
    }
  };

  // GET /api/v1/franchises/:franchiseId/orders
  getFranchiseOrders = async (req: Request, res: Response): Promise<void> => {
    const parsed = paginationSchema.safeParse(req.query);

    if (!parsed.success) {
      sendError(res, 400, "VALIDATION_ERROR", "Некорректные параметры запроса");
      return;
    }

    try {
      const { page, limit, status, deliveryType } = parsed.data;
      const result = await this.orderService.getFranchiseOrders(
        req.params.franchiseId, page, limit, status as OrderStatus, deliveryType
      );
      sendSuccess(res, result.data, 200, { page, limit, total: result.total });
    } catch (err: any) {
      sendError(res, err.status ?? 500, err.code ?? "INTERNAL_ERROR", err.message);
    }
  };

  // PUT /api/v1/orders/:orderId/status
  updateStatus = async (req: Request, res: Response): Promise<void> => {
    const parsed = updateStatusSchema.safeParse(req.body);

    if (!parsed.success) {
      sendError(res, 400, "VALIDATION_ERROR", "Некорректный статус",
        parsed.error.errors.map((e) => ({ field: e.path.join("."), message: e.message }))
      );
      return;
    }

    try {
      const order = await this.orderService.updateStatus(
        req.params.orderId,
        parsed.data.status as OrderStatus
      );
      sendSuccess(res, order);
    } catch (err: any) {
      sendError(res, err.status ?? 500, err.code ?? "INTERNAL_ERROR", err.message, err.details);
    }
  };

  // PUT /api/v1/orders/:orderId/courier
  assignCourier = async (req: Request, res: Response): Promise<void> => {
    const parsed = assignCourierSchema.safeParse(req.body);

    if (!parsed.success) {
      sendError(res, 400, "VALIDATION_ERROR", "Ошибка валидации",
        parsed.error.errors.map((e) => ({ field: e.path.join("."), message: e.message }))
      );
      return;
    }

    try {
      const order = await this.orderService.assignCourier(
        req.params.orderId,
        parsed.data.courierId
      );
      sendSuccess(res, order);
    } catch (err: any) {
      sendError(res, err.status ?? 500, err.code ?? "INTERNAL_ERROR", err.message, err.details);
    }
  };

  // DELETE /api/v1/orders/:orderId
  cancelOrder = async (req: Request, res: Response): Promise<void> => {
    try {
      await this.orderService.cancelOrder(
        req.params.orderId,
        req.user!.sub,
        req.user!.role
      );
      res.status(204).send();
    } catch (err: any) {
      sendError(res, err.status ?? 500, err.code ?? "INTERNAL_ERROR", err.message, err.details);
    }
  };
}
```

---

### 4.7 Регистрация маршрутов

```typescript
// src/routes/order.routes.ts

import { Router } from "express";
import { OrderController } from "../controllers/OrderController";
import { authenticate, authorize } from "../middleware/auth";

export function createOrderRouter(controller: OrderController): Router {
  const router = Router();

  // Все маршруты требуют аутентификации
  router.use(authenticate);

  // POST /api/v1/orders — создание заказа (только покупатель)
  router.post(
    "/orders",
    authorize("customer"),
    controller.createOrder
  );

  // GET /api/v1/orders — история заказов покупателя
  router.get(
    "/orders",
    authorize("customer"),
    controller.getCustomerOrders
  );

  // GET /api/v1/orders/:orderId — получение заказа по ID
  router.get(
    "/orders/:orderId",
    authorize("customer", "kitchen_staff", "franchise_owner", "courier"),
    controller.getOrderById
  );

  // PUT /api/v1/orders/:orderId/status — обновление статуса
  router.put(
    "/orders/:orderId/status",
    authorize("kitchen_staff", "franchise_owner"),
    controller.updateStatus
  );

  // PUT /api/v1/orders/:orderId/courier — назначение курьера
  router.put(
    "/orders/:orderId/courier",
    authorize("franchise_owner"),
    controller.assignCourier
  );

  // DELETE /api/v1/orders/:orderId — отмена заказа
  router.delete(
    "/orders/:orderId",
    authorize("customer", "franchise_owner"),
    controller.cancelOrder
  );

  // GET /api/v1/franchises/:franchiseId/orders — заказы франшизы
  router.get(
    "/franchises/:franchiseId/orders",
    authorize("kitchen_staff", "franchise_owner"),
    controller.getFranchiseOrders
  );

  return router;
}
```

---

### 4.8 Точка входа приложения

```typescript
// src/app.ts

import express from "express";
import { createOrderRouter } from "./routes/order.routes";
import { OrderController } from "./controllers/OrderController";
import { OrderService } from "./services/OrderService";

// Composition Root — сборка зависимостей (DIP)
// В реальном приложении здесь подключаются реальные реализации
import { PostgresOrderRepository } from "./repositories/PostgresOrderRepository";
import { RabbitMQEventPublisher } from "./publishers/RabbitMQEventPublisher";

const repo = new PostgresOrderRepository();
const publisher = new RabbitMQEventPublisher();
const orderService = new OrderService(repo, publisher);
const orderController = new OrderController(orderService);

const app = express();
app.use(express.json());

// Регистрация маршрутов с префиксом версии (Решение 2)
app.use("/api/v1", createOrderRouter(orderController));

export default app;
```

---

## Сводная таблица эндпоинтов

| Метод | URL | Роль | Описание |
|-------|-----|------|---------|
| `POST` | `/api/v1/orders` | customer | Создание нового заказа |
| `GET` | `/api/v1/orders` | customer | История заказов покупателя |
| `GET` | `/api/v1/orders/:orderId` | все | Получение заказа по ID |
| `GET` | `/api/v1/franchises/:franchiseId/orders` | kitchen_staff, franchise_owner | Заказы магазина |
| `PUT` | `/api/v1/orders/:orderId/status` | kitchen_staff, franchise_owner | Обновление статуса заказа |
| `PUT` | `/api/v1/orders/:orderId/courier` | franchise_owner | Назначение курьера |
| `DELETE` | `/api/v1/orders/:orderId` | customer, franchise_owner | Отмена заказа |
