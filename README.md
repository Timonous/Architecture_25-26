# **Либенсон А. М. РИС-22-1**

# **Проектирование архитектуры программных систем**

# ***Лабораторная работа №3***

## 

## **Тема:**

## Использование принципов проектирования на уровне методов и классов

## **Цель:**

## Получить опыт проектирования и реализации модулей с использованием принципов KISS, YAGNI, DRY, SOLID и др.

## **1. Диаграмма контейнеров**
![c4_containers](Lab3/ДиаграмаКонтейнеров.png)

## **2. Диаграмма компонентов контейнера "Сервис заказов"**
![c4_components](Lab3/ДиаграмаКомпонентов.png)

## **3. Диаграмма последовательностей**
```mermaid
sequenceDiagram
    participant Client as Мобильное приложение покупателя
    participant GW as API Gateway
    participant OC as Order Controller
    participant OS as Order Service
    participant OR as Order Repository
    participant OSM as Order Status Manager
    participant OCM as Order Cache Manager
    participant OEP as Order Event Publisher
    participant DB as БД заказов
    participant Cache as Кэш (Redis)
    participant MB as Брокер сообщений

    Client->>GW: POST /orders (состав заказа)
    GW->>OC: Маршрутизация запроса

    OC->>OS: Создать заказ

    OS->>OR: Сохранить черновик заказа
    OR->>DB: SQL INSERT (заказ со статусом "новый")
    DB-->>OR: Заказ сохранён, order_id
    OR-->>OS: order_id

    OS->>OSM: Установить начальный статус "принят"
    OSM->>OR: Обновить статус в БД
    OR->>DB: SQL UPDATE status = "принят"
    DB-->>OR: OK
    OSM->>OCM: Обновить статус в кэше
    OCM->>Cache: SET order:{id}:status = "принят"
    Cache-->>OCM: OK

    OS->>OEP: Опубликовать событие "заказ создан"
    OEP->>MB: Async — событие {order_id, status, user_id}

    OS-->>OC: Заказ успешно создан (order_id, статус)
    OC-->>GW: HTTP 201 Created
    GW-->>Client: Заказ принят, order_id, статус "принят"
```
Диаграмма описывает процесс взаимодействия компонентов Сервиса заказов:
1. **Приём запроса**
   Мобильное приложение отправляет `POST-запрос`. **API Gateway** маршрутизирует его в **Order Controller**, который передаёт управление в **Order Service**.
2. **Сохранение заказа**
   **Order Service** через **Order Repository** создаёт запись в БД заказов со статусом `новый`.
3. **Установка статуса**
   **Order Status Manager** переводит заказ в статус `принят`, обновляя одновременно:
   * **БД** через **Order Repository**;
   * **Кэш Redis** через **Order Cache Manager** (для быстрого чтения статуса клиентом).
4. **Публикация события**
   **Order Event Publisher** асинхронно отправляет в **Брокер сообщений** событие `заказ создан`. На него подписаны:
   * **Сервис уведомлений** (отправка push-уведомления);
   * **Сервис аналитики**.
   *Обработка происходит независимо и не блокирует основной поток.*
5. **Ответ клиенту**
   **Order Service** возвращает результат вверх по цепочке. Клиент получает подтверждение с `order_id` и текущим статусом.

## **4. Модель БД**
