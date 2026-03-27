# AvanPay Marketplace - Технический Документ Проектирования

> **Версия:** 2.1  
> **Статус:** Draft  
> **Дата:** 26-03-2026  
> **Автор:** AvanPay Architecture Team
>
> **Изменения v2.1:**
>
> - Добавлен раздел **Event-Driven Architecture** (3.4)
>   - Каталог событий с источниками и подписчиками
>   - Пример flow OrderPaid с кодом
>   - Сравнение реализаций Event Bus (Redis/NATS/RabbitMQ/Kafka)
>   - Эволюция Event Bus от MVP до Scale
> - Добавлен раздел **Real-Time решения** (3.5)
>   - Сравнение Centrifugo / Socket.io / GraphQL Subscriptions
>   - Пример интеграции Centrifugo (backend + frontend)
>   - Рекомендация: выбор остаётся за командой
>
> **Изменения v2.0:**
>
> - Архитектура изменена на модульный монолит
> - Добавлена система Feature Flags
> - Добавлена таблица user_connections для Unified Auth
> - Детализирована Design System
> - Добавлен раздел Telegram Mini App (TMA)

---

## 📋 Оглавление

1. [Введение и Цели](#1-введение-и-цели)
2. [Бизнес-Требования](#2-бизнес-требования)
3. [Архитектура Системы](#3-архитектура-системы)
4. [Модульная Структура](#4-модульная-структура)
5. [Доменная Модель](#5-доменная-модель)
6. [Навигация и UX](#6-навигация-и-ux)
7. [API Дизайн](#7-api-дизайн)
8. [База Данных](#8-база-данных)
9. [Интеграции](#9-интеграции)
10. [Безопасность](#10-безопасность)
11. [Масштабирование](#11-масштабирование)
12. [Feature Flags](#12-feature-flags)
13. [Telegram Mini App (TMA)](#13-telegram-mini-app-tma)
14. [Design System](#14-design-system)
15. [Дорожная Карта](#15-дорожная-карта)

---

## 1. Введение и Цели

### 1.1 Описание Проекта

AvanPay — маркетплейс для торговли игровыми товарами и услугами с системой гаранта. Платформа соединяет продавцов и покупателей, обеспечивая безопасность сделок через эскроу-механизм.

### 1.2 Ключевые Цели

| Цель                   | Метрика успеха                                                                  | Приоритет   |
| ---------------------- | ------------------------------------------------------------------------------- | ----------- |
| Безопасность сделок    | 99.5% успешных сделок без споров                                                | 🔴 Критично |
| Удобство использования | NPS > 50, конверсия > 5%                                                        | 🔴 Критично |
| Скорость транзакций    | Время ответа < 1 час                                                            | 🟡 Важно    |
| Масштабируемость       | 100K+++ MAU к 6 месяцам                                                         | 🟡 Важно    |
| Модульность            | Возможность добавления новые типы товаров/каталогов/структур без изменения ядра | 🔴 Критично |

### 1.3 Принципы Проектирования

```
┌────────────────────────────────────────────────────────────────┐
│                    DESIGN PRINCIPLES                           │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. MODULARITY FIRST                                           │
│     └── Каждый модуль независим, loosely coupled               │
│                                                                │
│  2. PLUGIN ARCHITECTURE                                        │
│     └── Новые типы товаров = плагины вкл/выкл не трогаем ядро  │
│                                                                │
│  3. EVENT-DRIVEN                                               │
│     └── Модули общаются через события, не прямые вызовы        │
│                                                                │
│  4. API-FIRST                                                  │
│     └── Весь функционал доступен через API                     │
│                                                                │
│  5. CONVENTION OVER CONFIGURATION                              │
│     └── Дефолтные настройки для 80% случаев                    │
│                                                                │
│  6. GRACEFUL DEGRADATION                                       │
│     └── Система работает даже при падении части модулей        │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## 2. Бизнес-Требования

### 2.1 Пользовательские Сценарии

#### 2.1.1 Покупатель (Buyer)

```
UC-B01: Поиск товара
  1. Ввести запрос в поиске / выбрать в каталоге
  2. Применить фильтры
  3. Выбрать товар
  4. Просмотреть детали
  5. Проверить репутацию продавца

UC-B02: Покупка товара
  1. Добавить в корзину / Купить сразу
  2. Подтвердить детали заказа
  3. Оплатить (карта/крипто/баланс)
  4. Получить товар
  5. Подтвердить получение
  6. Оставить отзыв

UC-B03: Разрешение спора
  1. Открыть спор
  2. Описать проблему
  3. Загрузить доказательства
  4. Дождаться решения модератора
```

#### 2.1.2 Продавец (Seller)

```
UC-S01: Создание объявления
  1. Выбрать игру / категорию / услугу
  2. Выбрать тип товара
  3. Заполнить атрибуты (динамическая форма)
  4. Загрузить медиа
  5. Установить цену
  6. Опубликовать

UC-S02: Обработка заказа
  1. Получить уведомление
  2. Связаться с покупателем
  3. Передать товар
  4. Отметить как доставлено
  5. Получить оплату

UC-S03: Управление продажами
  1. Просмотреть статистику
  2. Редактировать объявления
  3. Вывести средства
```

#### 2.1.3 Модератор (Moderator)

```
UC-M01: Разрешение спора
  1. Изучить детали сделки
  2. Проверить доказательства
  3. Принять решение
  4. Завершить спор
```

### 2.2 Требования к Расширяемости

| Область расширения            | Механизм                    | Пример                                        |
| ----------------------------- | --------------------------- | --------------------------------------------- |
| Новые типы товаров            | Product Type Plugin         | Добавление NFT как новый тип                  |
| Новые игры                    | Game Config                 | Добавление новой игры с кастомными атрибутами |
| Новые платежные методы        | Payment Gateway Plugin      | Интеграция нового платежного шлюза            |
| Новые типы уведомлений        | Notification Channel Plugin | Интеграция Discord уведомлений                |
| Новые категории пользователей | Role Plugin                 | Добавление VIP-пользователей                  |
| Новые API endpoints           | API Versioning              | Версионирование без breaking changes          |

---

## 3. Архитектура Системы

### 3.1 Архитектурная Стратегия: Модульный Монолит

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    АРХИТЕКТУРНАЯ СТРАТЕГИЯ                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ПОЧЕМУ МОДУЛЬНЫЙ МОНОЛИТ?                                              │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │  ✅ Быстрая разработка (1 кодbase, 1 deploy)                    │     │
│  │  ✅ Простая инфраструктура (1 контейнер)                        │     │
│  │  ✅ Лёгкий рефакторинг (без контрактов между сервисами)         │     │
│  │  ✅ Идеально для команды 1-5 человек                            │     │
│  │  ✅ Простой деплой и отладка                                    │     │
│  └────────────────────────────────────────────────────────────────┘     │
│                                                                          │
│  КОГДА ВЫНОСИТЬ В МИКРОСЕРВИС?                                          │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │  • Модуль > 10,000 строк кода                                   │     │
│  │  • Нагрузка > 1000 RPS только на этот модуль                    │     │
│  │  • Отдельная команда для этого модуля                           │     │
│  │  • Критическая изоляция (например, Payment Processing)          │     │
│  └────────────────────────────────────────────────────────────────┘     │
│                                                                          │
│  ПЕРВЫЕ КАНДИДАТЫ ДЛЯ ВЫНОСА (при росте нагрузки):                      │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │  1. Chat Service → Centrifugo или WebSocket (Go/Node.js)        │     │
│  │  2. Payment Service → Высоконагруженный процессинг              │     │
│  │  3. Search Service → Elasticsearch cluster                      │     │
│  └────────────────────────────────────────────────────────────────┘     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 High-Level Architecture (Модульный Монолит)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           CLIENT LAYER                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │              ЕДИНЫЙ NEXT.JS ПРОЕКТ (Shared Core)                 │   │
│  │                                                                   │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │   │
│  │  │   Web App    │  │   Telegram   │  │   Mobile     │            │   │
│  │  │   (Browser)  │  │   Mini App   │  │   (PWA/SSR)  │            │   │
│  │  │              │  │   (TMA)      │  │              │            │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘            │   │
│  │                                                                   │   │
│  │  Определение среды:                                               │   │
│  │  • Web: стандартный рендеринг, SEO, полный UI                     │   │
│  │  • TMA: window.Telegram.WebApp, Telegram CSS variables            │   │
│  │  • Mobile: адаптивный UI, touch-friendly                          │   │
│  │                                                                   │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           API GATEWAY                                    │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Rate Limiting │ Auth │ Routing │ Versioning │ Caching          │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    MODULAR MONOLITH (Single Deploy)                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                     NESTJS APPLICATION                            │  │
│  │                                                                    │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                │  │
│  │  │ AuthModule  │  │ UsersModule │  │CatalogModule│                │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                │  │
│  │                                                                    │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                │  │
│  │  │OrdersModule │  │PaymentsModule│ │ ChatModule  │                │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                │  │
│  │                                                                    │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                │  │
│  │  │DisputesModule│ │Notifications│  │SearchModule │                │  │
│  │  │             │  │   Module    │  │             │                │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                │  │
│  │                                                                    │  │
│  │  ┌─────────────────────────────────────────────────────────────┐  │  │
│  │  │                    EVENT BUS (Внутренний)                    │  │  │
│  │  │  OrderCreated │ PaymentReceived │ DisputeOpened │ ...        │  │  │
│  │  └─────────────────────────────────────────────────────────────┘  │  │
│  │                                                                    │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          DATA LAYER                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │
│  │  PostgreSQL  │  │    Redis     │  │Elasticsearch │                  │
│  │  (Main DB)   │  │   (Cache)    │  │  (Search)    │                  │
│  └──────────────┘  └──────────────┘  └──────────────┘                  │
│                                                                          │
│  ┌──────────────┐  ┌──────────────┐                                     │
│  │   S3/MinIO   │  │Message Queue │                                     │
│  │   (Files)    │  │  (NATS/RMQ)  │                                     │
│  └──────────────┘  └──────────────┘                                     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        EXTERNAL INTEGRATIONS                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │
│  │ Payment GWs  │  │ Telegram API │  │ Email Service│                  │
│  │(ЮKassa, etc.)│  │              │  │ (SendGrid)  │                  │
│  └──────────────┘  └──────────────┘  └──────────────┘                  │
│                                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │
│  │   Crypto GWs │  │ SMS Gateway  │  │Analytics(GA4)│                  │
│  │(Cryptomus)   │  │              │  │              │                  │
│  └──────────────┘  └──────────────┘  └──────────────┘                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Service Communication Patterns

```
┌─────────────────────────────────────────────────────────────┐
│                  COMMUNICATION PATTERNS                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  SYNC (HTTP/gRPC):                                           │
│  └── Для операций, требующих немедленного ответа            │
│      • Auth verification                                     │
│      • Payment processing                                    │
│      • Real-time availability check                         │
│                                                              │
│  ASYNC (Message Queue):                                      │
│  └── Для фоновых задач и событий                            │
│      • Order state changes                                   │
│      • Notification dispatch                                 │
│      • Analytics events                                      │
│      • Search index updates                                  │
│                                                              │
│  EVENT SOURCING (Critical paths):                           │
│  └── Для аудита и восстановления состояния                  │
│      • Payment transactions                                  │
│      • Dispute resolution                                    │
│      • User verification                                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3.4 Event-Driven Architecture

#### 3.4.1 Зачем Event-Driven?

```
┌─────────────────────────────────────────────────────────────────┐
│                    ЗАЧЕМ EVENT-DRIVEN?                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  БЕЗ СОБЫТИЙ (Tight Coupling):                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  OrderService.createOrder() {                              ││
│  │    // Создаём заказ                                         ││
│  │    await orderRepo.save(order);                            ││
│  │                                                             ││
│  │    // Прямые вызовы - ЖЁСТКАЯ СВЯЗЬ                        ││
│  │    await notificationService.send(user);  // Если упадёт?  ││
│  │    await analyticsService.track(order);   // Если таймаут? ││
│  │    await fraudService.check(user);        // Блокирует!    ││
│  │  }                                                          ││
│  │                                                             ││
│  │  Проблемы:                                                  ││
│  │  ❌ Заказ не создастся если NotificationService недоступен ││
│  │  ❌ Нельзя добавить новую фичу без изменения OrderService ││
│  │  ❌ Медленные сервисы замедляют всё                         ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  С СОБЫТИЯМИ (Loose Coupling):                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  OrderService.createOrder() {                              ││
│  │    await orderRepo.save(order);                            ││
│  │    await eventBus.publish('OrderCreated', { orderId });    ││
│  │    // Готово! Остальное происходит асинхронно              ││
│  │  }                                                          ││
│  │                                                             ││
│  │  // Подписчики (не связаны с OrderService):                ││
│  │  NotificationHandler.on('OrderCreated') → push + email      ││
│  │  AnalyticsHandler.on('OrderCreated') → track               ││
│  │  FraudHandler.on('OrderCreated') → async check             ││
│  │  LoyaltyHandler.on('OrderCreated') → add points (NEW!)     ││
│  │                                                             ││
│  │  Преимущества:                                              ││
│  │  ✅ Заказ создаётся даже если Notifications недоступны      ││
│  │  ✅ Новая фича = новый подписчик, без изменения ядра       ││
│  │  ✅ Каждый сервис работает независимо                      ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### 3.4.2 Event Catalog

| Событие               | Источник       | Payload                                             | Подписчики                                   |
| --------------------- | -------------- | --------------------------------------------------- | -------------------------------------------- |
| **ORDER EVENTS**      |
| `OrderCreated`        | OrdersModule   | `{ orderId, buyerId, sellerId, listingId, amount }` | Notifications, FraudCheck, Analytics         |
| `OrderPaid`           | PaymentsModule | `{ orderId, transactionId, amount, escrowId }`      | Notifications, Escrow, FraudCheck, Analytics |
| `OrderDelivered`      | OrdersModule   | `{ orderId, deliveryData }`                         | Notifications, Analytics                     |
| `OrderCompleted`      | OrdersModule   | `{ orderId, reviewId? }`                            | Notifications, SellerStats, Loyalty          |
| `OrderCancelled`      | OrdersModule   | `{ orderId, reason, refundAmount? }`                | Notifications, Escrow (refund), Analytics    |
| `DisputeOpened`       | DisputesModule | `{ disputeId, orderId, reason }`                    | Notifications, Moderation, Analytics         |
| `DisputeResolved`     | DisputesModule | `{ disputeId, orderId, resolution, winner }`        | Notifications, Escrow, Analytics             |
| **PAYMENT EVENTS**    |
| `PaymentInitiated`    | PaymentsModule | `{ paymentId, userId, amount, method }`             | Analytics                                    |
| `PaymentCompleted`    | PaymentsModule | `{ paymentId, transactionId }`                      | Notifications, Wallet, Analytics             |
| `PaymentFailed`       | PaymentsModule | `{ paymentId, reason }`                             | Notifications, Analytics                     |
| `WithdrawalRequested` | PaymentsModule | `{ withdrawalId, userId, amount }`                  | Notifications, Moderation (if flagged)       |
| `WithdrawalCompleted` | PaymentsModule | `{ withdrawalId }`                                  | Notifications                                |
| **USER EVENTS**       |
| `UserRegistered`      | AuthModule     | `{ userId, email?, telegramId? }`                   | Notifications (welcome), Analytics           |
| `UserVerified`        | UsersModule    | `{ userId, level }`                                 | Notifications, Analytics                     |
| `SellerLevelUp`       | UsersModule    | `{ userId, newLevel }`                              | Notifications, Analytics                     |
| **LISTING EVENTS**    |
| `ListingCreated`      | CatalogModule  | `{ listingId, sellerId, gameId }`                   | Search (index), Analytics                    |
| `ListingUpdated`      | CatalogModule  | `{ listingId, changes }`                            | Search (reindex)                             |
| `ListingSold`         | OrdersModule   | `{ listingId, orderId }`                            | SellerStats, Search (stock)                  |
| `ListingExpired`      | CatalogModule  | `{ listingId }`                                     | Notifications, Search (remove)               |
| **CHAT EVENTS**       |
| `MessageSent`         | ChatModule     | `{ messageId, conversationId, senderId }`           | Notifications (push to recipients)           |
| `ConversationCreated` | ChatModule     | `{ conversationId, participants }`                  | Notifications                                |
| **SYSTEM EVENTS**     |
| `FraudAlert`          | FraudModule    | `{ alertId, userId, type, risk }`                   | Notifications (admin), Moderation            |
| `SecurityEvent`       | AuthModule     | `{ userId, eventType, ip, device }`                 | Notifications (suspicious login)             |

#### 3.4.3 Event Flow Пример: OrderPaid

```
┌─────────────────────────────────────────────────────────────────┐
│              EVENT FLOW: OrderPaid (пример)                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ИСТОЧНИК: PaymentsModule                                       │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  async handlePaymentCallback(paymentData) {                ││
│  │    const transaction = await this.processPayment(payment); ││
│  │                                                             ││
│  │    // Публикуем событие                                     ││
│  │    await this.eventBus.publish('OrderPaid', {              ││
│  │      orderId: transaction.orderId,                          ││
│  │      transactionId: transaction.id,                         ││
│  │      amount: transaction.amount,                           ││
│  │      escrowId: escrow.id,                                  ││
│  │      timestamp: new Date()                                 ││
│  │    });                                                      ││
│  │  }                                                          ││
│  └─────────────────────────────────────────────────────────────┘│
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    EVENT BUS (Redis/NATS)                   ││
│  └─────────────────────────────────────────────────────────────┘│
│                              │                                   │
│            ┌─────────────────┼─────────────────┐                │
│            │                 │                 │                │
│            ▼                 ▼                 ▼                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │Notifications │  │   Escrow     │  │  FraudCheck  │          │
│  │   Handler    │  │   Handler    │  │   Handler    │          │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤          │
│  │ @Subscribe(  │  │ @Subscribe(  │  │ @Subscribe(  │          │
│  │ 'OrderPaid'  │  │ 'OrderPaid'  │  │ 'OrderPaid'  │          │
│  │ )            │  │ )            │  │ )            │          │
│  │              │  │              │  │              │          │
│  │ // Покупателю│  │ // Разморозка│  │ // Проверка  │          │
│  │ push: "Оплата│  │ // денег в   │  │ // на фрод   │          │
│  │ // прошла"   │  │ // эскроу    │  │ // (асинхр.) │          │
│  │              │  │              │  │              │          │
│  │ // Продавцу  │  │ await escrow │  │ if (risk>80) │          │
│  │ push: "Новый │  │  .hold(...); │  │  .flag();    │          │
│  │ // заказ!"   │  │              │  │              │          │
│  │              │  │ // Уведомить │  │ // Не блоки- │          │
│  │ // Email     │  │ // продавца  │  │ // рует      │          │
│  │ // (опц.)    │  │              │  │ // основной  │          │
│  └──────────────┘  └──────────────┘  │ // flow      │          │
│                                      └──────────────┘          │
│                                                                  │
│  Дополнительно (можно добавить без изменения PaymentsModule):   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Analytics   │  │   Loyalty    │  │  Audit Log   │          │
│  │   Handler    │  │   Handler    │  │   Handler    │          │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤          │
│  │ // Записать  │  │ // Начислить │  │ // Сохранить │          │
│  │ // в GA4/    │  │ // бонусные  │  │ // для аудита│          │
│  │ // Mixpanel  │  │ // баллы     │  │              │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### 3.4.4 Реализация Event Bus

```typescript
// apps/api/src/core/events/event-bus.interface.ts
export interface EventBus {
  publish<T>(eventName: string, payload: T): Promise<void>;
  subscribe<T>(eventName: string, handler: EventHandler<T>): void;
}

export interface EventHandler<T> {
  handle(payload: T): Promise<void>;
}

// apps/api/src/core/events/redis-event-bus.ts
import { Injectable, OnModuleInit } from "@nestjs/common";
import { Redis } from "ioredis";

@Injectable()
export class RedisEventBus implements EventBus, OnModuleInit {
  private publisher: Redis;
  private subscriber: Redis;
  private handlers: Map<string, EventHandler<unknown>[]> = new Map();

  constructor() {
    this.publisher = new Redis(process.env.REDIS_URL);
    this.subscriber = new Redis(process.env.REDIS_URL);
  }

  async onModuleInit() {
    this.subscriber.on("message", async (channel, message) => {
      const handlers = this.handlers.get(channel) || [];
      const payload = JSON.parse(message);

      // Обрабатываем всех подписчиков параллельно
      await Promise.allSettled(
        handlers.map((handler) => handler.handle(payload)),
      );
    });
  }

  async publish<T>(eventName: string, payload: T): Promise<void> {
    await this.publisher.publish(
      eventName,
      JSON.stringify({ ...payload, _timestamp: new Date().toISOString() }),
    );
  }

  subscribe<T>(eventName: string, handler: EventHandler<T>): void {
    const existing = this.handlers.get(eventName) || [];
    this.handlers.set(eventName, [
      ...existing,
      handler as EventHandler<unknown>,
    ]);
    this.subscriber.subscribe(eventName);
  }
}

// apps/api/src/modules/orders/events/handlers/notification.handler.ts
import { Injectable } from "@nestjs/common";
import { EventHandler } from "@/core/events";
import { NotificationService } from "@/modules/notifications";

interface OrderPaidPayload {
  orderId: string;
  transactionId: string;
  amount: number;
  escrowId: string;
}

@Injectable()
export class OrderPaidNotificationHandler implements EventHandler<OrderPaidPayload> {
  constructor(private notifications: NotificationService) {}

  async handle(payload: OrderPaidPayload): Promise<void> {
    // Получаем данные заказа (уже не блокирует создание заказа)
    const order = await this.orderRepo.findById(payload.orderId);

    // Уведомления отправляем параллельно
    await Promise.all([
      this.notifications.push(order.buyerId, {
        title: "Оплата прошла",
        body: `Заказ #${order.id} оплачен. Ожидайте доставку.`,
        data: { orderId: order.id },
      }),
      this.notifications.push(order.sellerId, {
        title: "Новый заказ!",
        body: `Заказ #${order.id} на сумму ${payload.amount}₽`,
        data: { orderId: order.id },
      }),
    ]);
  }
}

// Регистрация обработчика
// apps/api/src/modules/orders/orders.module.ts
@Module({
  providers: [
    OrderPaidNotificationHandler,
    OrderPaidEscrowHandler,
    OrderPaidFraudHandler,
    // Можно добавить новый обработчик БЕЗ изменения OrdersModule
    // OrderPaidLoyaltyHandler,  <-- новая фича!
  ],
})
export class OrdersModule {}
```

#### 3.4.5 Сравнение реализаций Event Bus

| Решение           | Плюсы                                                                      | Минусы                                               | Когда использовать                    |
| ----------------- | -------------------------------------------------------------------------- | ---------------------------------------------------- | ------------------------------------- |
| **Redis Pub/Sub** | ✅ Уже есть в стеке<br>✅ Просто внедрить<br>✅ Достаточно для MVP         | ❌ Нет персистентности<br>❌ Нет guaranteed delivery | MVP, до 10K событий/день              |
| **NATS**          | ✅ Быстрый<br>✅ Lightweight<br>✅ Persistence опционально                 | ❌ Отдельный сервис<br>❌ Нет встроенного UI         | Growth phase, нужен контроль доставки |
| **RabbitMQ**      | ✅ Guaranteed delivery<br>✅ Routing<br>✅ Management UI                   | ❌ Тяжелее NATS<br>❌ Сложнее настройка              | Enterprise, критичная доставка        |
| **Kafka**         | ✅ Event Sourcing<br>✅ Replay events<br>✅ Высокая пропускная способность | ❌ Overkill для старта<br>❌ Сложная инфраструктура  | Scale phase, нужен аудит событий      |

**Рекомендация**

- **MVP (месяцы 1-6):** Redis Pub/Sub — уже есть в стеке, минимум инфраструктуры
- **Growth (месяцы 7-12):** NATS или RabbitMQ — при росте нагрузки и потребности в guaranteed delivery
- **Scale (12+ месяцев):** Kafka — если нужен Event Sourcing для аудита финансовых операций

```
┌─────────────────────────────────────────────────────────────────┐
│                  EVENT BUS EVOLUTION                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  MVP (Месяцы 1-6)                                               │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Redis Pub/Sub                                              ││
│  │  ├── Уже используется для кэша и сессий                    ││
│  │  ├── Минимум дополнительной инфраструктуры                  ││
│  │  └── Достаточно для ~10K событий/день                       ││
│  └─────────────────────────────────────────────────────────────┘│
│                              │                                   │
│                              ▼                                   │
│  Growth (Месяцы 7-12)                                           │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  NATS JetStream или RabbitMQ                               ││
│  │  ├── Guaranteed delivery (важно для платежей)              ││
│  │  ├── Dead Letter Queue для failed handlers                 ││
│  │  ├── Monitoring и метрики                                   ││
│  │  └── Потребность: ~100K+ событий/день                       ││
│  └─────────────────────────────────────────────────────────────┘│
│                              │                                   │
│                              ▼                                   │
│  Scale (12+ месяцев)                                            │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Apache Kafka                                               ││
│  │  ├── Event Sourcing для финансовых операций                 ││
│  │  ├── Replay событий для отладки                             ││
│  │  ├── Аудит и комплаенс                                      ││
│  │  └── Потребность: ~1M+ событий/день                         ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.5 Real-Time решения (Чаты и Уведомления)

```
┌─────────────────────────────────────────────────────────────────┐
│                  REAL-TIME РЕШЕНИЯ                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ВАРИАНТ 1: Centrifugo (Рекомендация для P2P)                   │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Преимущества:                                              ││
│  │  ✅ Готовый WebSocket-сервер (не пишем сами)                ││
│  │  ✅ Масштабируемый (миллионы соединений)                     ││
│  │  ✅ Встроенная авторизация каналов                          ││
│  │  ✅ История сообщений                                       ││
│  │  ✅ Presence (кто онлайн)                                   ││
│  │  ✅ Admin UI для мониторинга                                ││
│  │  ✅ SDK для JS, Go, Python                                  ││
│  │                                                             ││
│  │  Когда использовать:                                        ││
│  │  • Чаты между покупателем и продавцом                       ││
│  │  • Real-time обновления статуса заказа                      ││
│  │  • Push-уведомления (вместо Polling)                        ││
│  │                                                             ││
│  │  Архитектура:                                               ││
│  │  ┌─────────┐    ┌─────────────┐    ┌─────────────┐         ││
│  │  │ Client  │◄──►│  Centrifugo │◄──►│   Redis     │         ││
│  │  │ (Web)   │    │  (WS Server)│    │  (Backend)  │         ││
│  │  └─────────┘    └─────────────┘    └─────────────┘         ││
│  │                       │                                    ││
│  │                       ▼                                    ││
│  │                 ┌─────────────┐                           ││
│  │                 │ PostgreSQL │ (история сообщений)       ││
│  │                 └─────────────┘                           ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ВАРИАНТ 2: Socket.io (Самописный сервер)                       │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Преимущества:                                              ││
│  │  ✅ Полный контроль над логикой                             ││
│  │  ✅ Интеграция в NestJS (@nestjs/platform-socket.io)        ││
│  │  ✅ Нет внешних зависимостей                                ││
│  │                                                             ││
│  │  Недостатки:                                                ││
│  │  ❌ Нужно писать логику самому                              ││
│  │  ❌ Сложнее масштабировать                                  ││
│  │  ❌ Нет встроенного UI для мониторинга                      ││
│  │                                                             ││
│  │  Когда использовать:                                        ││
│  │  • Нестандартные требования к протоколу                    ││
│  │  • Глубокая интеграция с NestJS                             ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ВАРИАНТ 3: GraphQL Subscriptions                               │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Преимущества:                                              ││
│  │  ✅ Единый API (REST + WS через GraphQL)                    ││
│  │  ✅ Type-safe подписки                                       ││
│  │  ✅ Фильтрация на уровне GraphQL                            ││
│  │                                                             ││
│  │  Недостатки:                                                ││
│  │  ❌ Overhead если нет GraphQL в проекте                     ││
│  │  ❌ Сложнее для простых use-cases                           ││
│  │                                                             ││
│  │  Когда использовать:                                        ││
│  │  • Уже используем GraphQL                                   ││
│  │  • Нужна сложная фильтрация real-time данных               ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  СРАВНЕНИЕ:                                                      │
│  ┌─────────────────┬────────────┬───────────┬──────────────┐   │
│  │                 │ Centrifugo │ Socket.io │ GraphQL Subs  │   │
│  ├─────────────────┼────────────┼───────────┼──────────────┤   │
│  │ Time to market  │ ⚡ Быстро  │ 🐢 Долго  │ 🐢 Долго     │   │
│  │ Scalability     │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐     │ ⭐⭐⭐        │   │
│  │ Поддержка       │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐⭐   │ ⭐⭐⭐⭐      │   │
│  │ Гибкость        │ ⭐⭐⭐      │ ⭐⭐⭐⭐⭐ │ ⭐⭐⭐⭐⭐    │   │
│  │ Monitoring UI   │ ✅ Да      │ ❌ Нет    │ ❌ Нет       │   │
│  │ История сообщ.  │ ✅ Да      │ ❌ Сами   │ ❌ Сами      │   │
│  │ Presence        │ ✅ Да      │ ✅ Да     │ ❌ Сами      │   │
│  └─────────────────┴────────────┴───────────┴──────────────┘   │
│                                                                  │
│  РЕКОМЕНДАЦИЯ ДЛЯ AVANPAY:                                       │
│  ├── MVP: Socket.io (интеграция с NestJS, минимум зависимостей) │
│  ├── Growth: Centrifugo (когда нагрузка растёт)                 │
│  └── Решение принимает команда на основе приоритетов            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### 3.5.1 Пример интеграции Centrifugo

```typescript
// apps/api/src/core/realtime/centrifugo.service.ts
import { Injectable } from "@nestjs/common";
import axios from "axios";

@Injectable()
export class CentrifugoService {
  private apiUrl = process.env.CENTRIFUGO_API_URL;
  private apiKey = process.env.CENTRIFUGO_API_KEY;

  async publish<T>(channel: string, data: T): Promise<void> {
    await axios.post(
      this.apiUrl,
      {
        method: "publish",
        params: {
          channel,
          data,
        },
      },
      {
        headers: { Authorization: `apikey ${this.apiKey}` },
      },
    );
  }

  async generateToken(userId: string, channels: string[]): Promise<string> {
    const response = await axios.post(
      this.apiUrl,
      {
        method: "generate_token",
        params: { user: userId, channels },
      },
      {
        headers: { Authorization: `apikey ${this.apiKey}` },
      },
    );
    return response.data.result.token;
  }
}

// Использование при отправке сообщения в чат
// apps/api/src/modules/chat/chat.service.ts
@Injectable()
export class ChatService {
  constructor(
    private centrifugo: CentrifugoService,
    private messagesRepo: MessagesRepository,
  ) {}

  async sendMessage(conversationId: string, senderId: string, content: string) {
    const message = await this.messagesRepo.create({
      conversationId,
      senderId,
      content,
    });

    // Публикуем в Centrifugo — все участники чата получат сообщение мгновенно
    await this.centrifugo.publish(`chat:${conversationId}`, {
      id: message.id,
      senderId: message.senderId,
      content: message.content,
      createdAt: message.createdAt,
    });

    return message;
  }
}

// Frontend: Подписка на канал чата
// apps/web/lib/use-chat.ts
import { Centrifuge } from "centrifuge";
import { useQueryClient } from "@tanstack/react-query";

export function useChat(conversationId: string) {
  const queryClient = useQueryClient();
  const [messages, setMessages] = useState<Message[]>([]);

  useEffect(() => {
    const centrifuge = new Centrifuge(
      "wss://centrifugo.avanpay.com/connection/websocket",
      {
        token: userToken, // Получен при логине
      },
    );

    const channel = centrifuge.newSubscription(`chat:${conversationId}`);

    channel.on("publication", (ctx) => {
      const message = ctx.data as Message;
      setMessages((prev) => [...prev, message]);

      // Инвалидируем кэш React Query
      queryClient.invalidateQueries(["messages", conversationId]);
    });

    channel.subscribe();
    centrifuge.connect();

    return () => {
      centrifuge.disconnect();
    };
  }, [conversationId]);

  return { messages };
}
```

---

## 4. Модульная Структура

### 4.1 Core Modules

```
avanpay/
├── apps/
│   ├── web/                          # Next.js Frontend
│   │   ├── app/
│   │   │   ├── (marketing)/          # Public pages
│   │   │   ├── (auth)/               # Auth pages
│   │   │   ├── (dashboard)/          # Protected dashboard
│   │   │   └── api/                  # API routes (BFF)
│   │   ├── components/
│   │   ├── lib/
│   │   └── styles/
│   │
│   ├── api/                          # Backend API
│   │   ├── src/
│   │   │   ├── modules/              # Feature modules
│   │   │   ├── core/                 # Core functionality
│   │   │   ├── shared/               # Shared utilities
│   │   │   └── infrastructure/      # Infrastructure code
│   │   └── tests/
│   │
│   └── worker/                       # Background workers
│       ├── src/
│       │   ├── consumers/            # Queue consumers
│       │   ├── schedulers/           # Cron jobs
│       │   └── processors/           # Task processors
│       └── tests/
│
├── packages/                         # Shared packages
│   ├── types/                        # TypeScript types
│   ├── utils/                        # Shared utilities
│   ├── ui/                           # Shared UI components
│   ├── config/                       # Configuration
│   ├── constants/                    # Constants & enums
│   └── validators/                   # Validation schemas
│
├── plugins/                          # Plugin system
│   ├── product-types/                # Product type plugins
│   │   ├── account/                  # Account type
│   │   ├── currency/                 # Currency type
│   │   ├── item/                     # Item type
│   │   ├── service/                  # Service type
│   │   ├── key/                      # Key type
│   │   └── topup/                    # Top-up type
│   │
│   ├── payment-gateways/             # Payment plugins
│   │   ├── yookassa/
│   │   ├── cryptomus/
│   │   ├── tinkoff/
│   │   └── stripe/
│   │
│   └── notifications/                # Notification plugins
│       ├── telegram/
│       ├── email/
│       ├── push/
│       └── discord/
│
├── infrastructure/                   # Infrastructure as Code
│   ├── docker/
│   ├── kubernetes/
│   └── terraform/
│
└── docs/                             # Documentation
    ├── api/
    ├── architecture/
    └── guides/
```

### 4.2 Module Interface Design

Каждый модуль должен следовать контракту:

```typescript
// packages/types/src/module.interface.ts

interface IModule {
  // Идентификация
  name: string;
  version: string;
  dependencies: string[];

  // Жизненный цикл
  onInit(): Promise<void>;
  onDestroy(): Promise<void>;

  // Конфигурация
  config: ModuleConfig;

  // События
  events: {
    published: EventDefinition[];
    subscribed: EventDefinition[];
  };

  // API
  routes?: RouteDefinition[];
  grpc?: GrpcServiceDefinition;

  // Миграции БД
  migrations?: MigrationDefinition[];

  // Health check
  healthCheck(): Promise<HealthStatus>;
}

// Пример модуля
interface ProductTypeModule extends IModule {
  // Тип товара
  type: ProductType;

  // Схема атрибутов
  attributeSchema: JSONSchema;

  // Валидаторы
  validators: AttributeValidator[];

  // UI компоненты (для фронтенда)
  uiComponents: {
    createForm: React.ComponentType;
    editForm: React.ComponentType;
    displayCard: React.ComponentType;
    detailView: React.ComponentType;
    filterPanel: React.ComponentType;
  };

  // Бизнес-логика
  handlers: {
    onCreate: (data: unknown) => Promise<Listing>;
    onUpdate: (listing: Listing, data: unknown) => Promise<Listing>;
    onPurchase: (order: Order) => Promise<void>;
    onDelivery: (order: Order) => Promise<void>;
    validateForSale: (listing: Listing) => Promise<ValidationResult>;
  };

  // Поиск
  search: {
    indexMapping: ElasticsearchMapping;
    searchHandler: (query: SearchQuery) => Promise<SearchResult>;
    suggestHandler: (prefix: string) => Promise<Suggestion[]>;
  };

  // Интеграции
  integrations?: {
    autoDelivery?: AutoDeliveryHandler;
    verification?: VerificationHandler;
  };
}
```

### 4.3 Plugin Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                     PLUGIN SYSTEM                              │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    CORE SYSTEM                          │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │   │
│  │  │ Plugin Host │  │ Event Bus   │  │ API Registry│      │   │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘      │   │
│  │         │                │                │             │   │
│  └─────────┼────────────────┼────────────────┼─────────────┘   │
│            │                │                │                 │
│            ▼                ▼                ▼                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    PLUGIN LAYER                         │   │
│  │                                                         │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐│   │
│  │  │ Product Type  │  │ Payment GW    │  │ Notification  ││   │
│  │  │   Plugins     │  │   Plugins     │  │   Plugins     ││   │
│  │  │               │  │               │  │               ││   │
│  │  │ • Account     │  │ • ЮKassa      │  │ • Telegram    ││   │
│  │  │ • Currency    │  │ • Cryptomus   │  │ • Email       ││   │
│  │  │ • Item        │  │ • Tinkoff     │  │ • Push        ││   │
│  │  │ • Service     │  │ • Stripe      │  │ • Discord     ││   │
│  │  │ • Key         │  │               │  │               ││   │
│  │  │ • Topup       │  │               │  │               ││   │
│  │  └───────────────┘  └───────────────┘  └───────────────┘│   │
│  │                                                         │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐│   │
│  │  │ Game Config   │  │ AI Provider   │  │ Analytics     ││   │
│  │  │   Plugins     │  │   Plugins     │  │   Plugins     ││   │
│  │  │               │  │               │  │               ││   │
│  │  │ • Genshin     │  │ • OpenAI      │  │ • GA4         ││   │
│  │  │ • Valorant    │  │ • Claude      │  │ • Mixpanel    ││   │
│  │  │ • WoW         │  │ • Local       │  │ • Custom      ││   │
│  │  │ • CS2         │  │               │  │               ││   │
│  │  └───────────────┘  └───────────────┘  └───────────────┘│   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## 5. Доменная Модель

### 5.1 Core Domains

```
┌─────────────────────────────────────────────────────────────────┐
│                     DOMAIN MODEL                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │                    IDENTITY DOMAIN                        │   │
│  │                                                            │   │
│  │  User ──────┬──────► Role                                  │   │
│  │   │         │                                              │   │
│  │   │         └──────► Permission                             │   │
│  │   │                                                        │   │
│  │   └────────────────► Session                              │   │
│  │                                                            │   │
│  │   └────────────────► Verification                         │   │
│  │                                                            │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │                    CATALOG DOMAIN                          │   │
│  │                                                            │   │
│  │  Game ──────────────► Category                            │   │
│  │   │                                                        │   │
│  │   └─────────────────► ProductType                          │   │
│  │                          │                                  │   │
│  │                          └──────► AttributeSchema           │   │
│  │                                                            │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │                    LISTING DOMAIN                          │   │
│  │                                                            │   │
│  │  Listing ────────────► ListingAttribute                   │   │
│  │   │                                                        │   │
│  │   ├───► Price                                              │   │
│  │   │                                                        │   │
│  │   ├───► Media                                              │   │
│  │   │                                                        │   │
│  │   ├───► DeliveryConfig                                     │   │
│  │   │                                                        │   │
│  │   └───► ListingStats                                       │   │
│  │                                                            │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │                    ORDER DOMAIN                            │   │
│  │                                                            │   │
│  │  Order ──────────────► OrderItem                          │   │
│  │   │                                                        │   │
│  │   ├───► OrderStatus (State Machine)                        │   │
│  │   │                                                        │   │
│  │   ├───► Delivery                                           │   │
│  │   │                                                        │   │
│  │   └───► Dispute                                             │   │
│  │                                                            │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │                    PAYMENT DOMAIN                          │   │
│  │                                                            │   │
│  │  Transaction ──────────► TransactionEvent                 │   │
│  │   │                                                        │   │
│  │   ├───► Wallet                                             │   │
│  │   │                                                        │   │
│  │   ├───► Withdrawal                                         │   │
│  │   │                                                        │   │
│  │   └───► Escrow                                             │   │
│  │                                                            │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │                    COMMUNICATION DOMAIN                    │   │
│  │                                                            │   │
│  │  Conversation ──────────► Message                         │   │
│  │                                                            │   │
│  │  Notification ──────────► NotificationLog                 │   │
│  │                                                            │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Order State Machine

```
┌─────────────────────────────────────────────────────────────────┐
│                    ORDER STATE MACHINE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                          ┌──────────┐                            │
│                          │  DRAFT   │                           │
│                          └────┬─────┘                            │
│                               │ create                           │
│                               ▼                                  │
│                         ┌──────────┐                             │
│               ┌─────────│  PENDING │─────────┐                  │
│               │         │  PAYMENT │         │                  │
│               │         └────┬─────┘         │                  │
│               │              │ paid          │ expired          │
│               │              ▼               │                  │
│               │        ┌──────────┐          │                  │
│               │        │  PAID    │          │                  │
│               │        │ (ESCROW) │          │                  │
│               │        └────┬─────┘          │                  │
│               │             │ deliver        │                  │
│               │             ▼                │                  │
│               │       ┌──────────┐           │                  │
│               │       │DELIVERING│           │                  │
│               │       └────┬─────┘           │                  │
│               │            │ confirm         │                  │
│               │            ▼                 │                  │
│               │       ┌──────────┐           │                  │
│               │       │COMPLETED │           │                  │
│               │       └──────────┘           │                  │
│               │                              │                  │
│               │ dispute                      │                  │
│               │                              │                  │
│               ▼                              ▼                  │
│         ┌──────────┐                  ┌──────────┐             │
│         │ DISPUTED │                  │ CANCELLED │             │
│         └────┬─────┘                  └───────────┘             │
│              │                                                   │
│              │ resolve                                           │
│              ▼                                                   │
│        ┌──────────┐                                             │
│        │ RESOLVED │                                             │
│        │ (REFUND/ │                                             │
│        │  RELEASE)│                                             │
│        └──────────┘                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 Data Models (Draft)

```sql
-- Core tables (simplified for planning)

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255),                    -- Может быть NULL если вход через Telegram
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255),            -- Может быть NULL если вход через Telegram
    role VARCHAR(20) NOT NULL DEFAULT 'user',
    verification_level VARCHAR(20) NOT NULL DEFAULT 'new',
    balance DECIMAL(15,2) DEFAULT 0,
    reputation_score DECIMAL(3,2) DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- User Connections (Unified Auth)
-- Связь один-ко-многим: один юзер может иметь email, telegram, google и т.д.
CREATE TABLE user_connections (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id) NOT NULL,
    provider VARCHAR(20) NOT NULL,         -- 'telegram', 'google', 'email', 'apple'
    provider_id VARCHAR(255) NOT NULL,     -- telegram_id, google_id, email
    provider_data JSONB,                   -- Дополнительные данные от провайдера
    is_primary BOOLEAN DEFAULT false,      -- Основной способ входа
    is_verified BOOLEAN DEFAULT false,     -- Подтверждён ли способ
    created_at TIMESTAMP DEFAULT NOW(),
    last_used_at TIMESTAMP,
    UNIQUE(provider, provider_id)          -- Один провайдер = одна связь
);

-- Индексы для user_connections
CREATE INDEX idx_user_connections_user ON user_connections(user_id);
CREATE INDEX idx_user_connections_provider ON user_connections(provider, provider_id);

-- Games catalog
CREATE TABLE games (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    icon_url VARCHAR(500),
    banner_url VARCHAR(500),
    is_active BOOLEAN DEFAULT true,
    popularity_rank INT,
    config JSONB, -- Game-specific configuration
    created_at TIMESTAMP DEFAULT NOW()
);

-- Product types
CREATE TABLE product_types (
    id UUID PRIMARY KEY,
    game_id UUID REFERENCES games(id),
    type VARCHAR(50) NOT NULL, -- 'account', 'currency', 'item', etc.
    name VARCHAR(255) NOT NULL,
    attribute_schema JSONB NOT NULL, -- JSON Schema for attributes
    ui_config JSONB, -- Frontend configuration
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Listings
CREATE TABLE listings (
    id UUID PRIMARY KEY,
    seller_id UUID REFERENCES users(id) NOT NULL,
    game_id UUID REFERENCES games(id) NOT NULL,
    product_type_id UUID REFERENCES product_types(id) NOT NULL,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    attributes JSONB NOT NULL, -- Dynamic attributes based on schema
    price DECIMAL(15,2) NOT NULL,
    currency VARCHAR(10) NOT NULL DEFAULT 'RUB',
    stock_type VARCHAR(20) NOT NULL, -- 'unlimited', 'limited', 'unique'
    stock_quantity INT,
    stock_reserved INT DEFAULT 0,
    delivery_type VARCHAR(20) NOT NULL, -- 'manual', 'automatic', 'semi'
    delivery_time_minutes INT DEFAULT 60,
    status VARCHAR(20) NOT NULL DEFAULT 'draft',
    stats JSONB DEFAULT '{}', -- views, favorites, sales count
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP
);

-- Orders
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    buyer_id UUID REFERENCES users(id) NOT NULL,
    seller_id UUID REFERENCES users(id) NOT NULL,
    listing_id UUID REFERENCES listings(id) NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'pending_payment',
    price DECIMAL(15,2) NOT NULL,
    fee_amount DECIMAL(15,2) NOT NULL,
    total_amount DECIMAL(15,2) NOT NULL,
    currency VARCHAR(10) NOT NULL DEFAULT 'RUB',
    escrow_id UUID,
    delivery_data JSONB,
    delivered_at TIMESTAMP,
    completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Transactions (Payments)
CREATE TABLE transactions (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id) NOT NULL,
    order_id UUID REFERENCES orders(id),
    type VARCHAR(30) NOT NULL, -- 'payment', 'withdrawal', 'refund', 'fee'
    amount DECIMAL(15,2) NOT NULL,
    currency VARCHAR(10) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    payment_method VARCHAR(50),
    payment_gateway VARCHAR(50),
    external_id VARCHAR(255),
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    processed_at TIMESTAMP
);

-- Escrow
CREATE TABLE escrow_accounts (
    id UUID PRIMARY KEY,
    order_id UUID REFERENCES orders(id) UNIQUE NOT NULL,
    amount DECIMAL(15,2) NOT NULL,
    currency VARCHAR(10) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'held', -- 'held', 'released', 'refunded'
    held_at TIMESTAMP DEFAULT NOW(),
    released_at TIMESTAMP,
    released_to UUID REFERENCES users(id)
);

-- Chat/Conversations
CREATE TABLE conversations (
    id UUID PRIMARY KEY,
    order_id UUID REFERENCES orders(id),
    type VARCHAR(20) NOT NULL, -- 'order', 'support', 'dispute'
    participants UUID[] NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE messages (
    id UUID PRIMARY KEY,
    conversation_id UUID REFERENCES conversations(id) NOT NULL,
    sender_id UUID REFERENCES users(id) NOT NULL,
    content TEXT NOT NULL,
    attachments JSONB,
    is_read BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Disputes
CREATE TABLE disputes (
    id UUID PRIMARY KEY,
    order_id UUID REFERENCES orders(id) UNIQUE NOT NULL,
    opened_by UUID REFERENCES users(id) NOT NULL,
    reason VARCHAR(50) NOT NULL,
    description TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'open',
    assigned_to UUID REFERENCES users(id), -- Moderator
    resolution VARCHAR(50),
    resolution_notes TEXT,
    evidence JSONB, -- Uploaded proofs
    created_at TIMESTAMP DEFAULT NOW(),
    resolved_at TIMESTAMP
);

-- Reviews
CREATE TABLE reviews (
    id UUID PRIMARY KEY,
    order_id UUID REFERENCES orders(id) UNIQUE NOT NULL,
    from_user_id UUID REFERENCES users(id) NOT NULL,
    to_user_id UUID REFERENCES users(id) NOT NULL,
    rating INT NOT NULL CHECK (rating >= 1 AND rating <= 5),
    comment TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

## 6. Навигация и UX

### 6.1 Information Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  INFORMATION ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PUBLIC AREA                                                     │
│  ├── 🏠 Главная (/)                                              │
│  │   ├── Banner                                            │
│  │   ├── Trending Games                                         │
│  │   ├── Featured Listings                                      │
│  │   ├── Recent Reviews                                         │
│  │   └── Trust Signals                                          │
│  │                                                               │
│  ├── 🔍 Поиск (/search)                                         │
│  │   ├── Search Bar (AI-powered)                                │
│  │   ├── Filters Panel                                          │
│  │   ├── Results Grid/List                                      │
│  │   └── Sort Options                                           │
│  │                                                               │
│  ├── 📂 Каталог (/catalog)                                      │
│  │   ├── По категориям                                          │
│  │   │   ├── Аккаунты (/catalog/accounts)                       │
│  │   │   ├── Валюта (/catalog/currency)                         │
│  │   │   ├── Предметы (/catalog/items)                          │
│  │   │   ├── Ключи (/catalog/keys)                              │
│  │   │   ├── Услуги (/catalog/services)                         │
│  │   │   └── Топ-ап (/catalog/topup)                            │
│  │   │                                                          │
│  │   └── По играм                                               │
│  │       ├── Популярные (/catalog/popular)                      │
│  │       ├── Новые (/catalog/new)                               │
│  │       └── Все игры (/catalog/games)                          │
│  │                                                               │
│  ├── 🎮 Страница игры (/game/{slug})                            │
│  │   ├── Game Info                                              │
│  │   ├── Product Types                                          │
│  │   ├── Filters                                                │
│  │   └── Listings                                               │
│  │                                                               │
│  ├── 📦 Карточка товара (/listing/{id})                         │
│  │   ├── Gallery                                                │
│  │   ├── Details & Attributes                                   │
│  │   ├── Seller Info                                            │
│  │   ├── Reviews                                                │
│  │   ├── Similar Listings                                       │
│  │   └── CTA (Buy/Contact)                                      │
│  │                                                               │
│  └── 👤 Профиль продавца (/seller/{id})                         │
│      ├── Seller Stats                                           │
│      ├── Reviews                                                │
│      ├── Active Listings                                        │
│      └── Contact                                                │
│                                                                  │
│  AUTHENTICATED AREA                                              │
│  ├── 🛒 Корзина (/cart)                                         │
│  ├── 💳 Чекаут (/checkout)                                      │
│  ├── 📋 Мои заказы (/orders)                                    │
│  │   └── Заказ (/order/{id})                                    │
│  │                                                               │
│  ├── 💰 Продавай (/sell)                                        │
│  │   ├── Dashboard (/sell/dashboard)                            │
│  │   ├── Объявления (/sell/listings)                           │
│  │   │   └── Создать (/sell/listings/create)                   │
│  │   ├── Заказы (/sell/orders)                                  │
│  │   ├── Статистика (/sell/analytics)                           │
│  │   ├── Финансы (/sell/finance)                               │
│  │   │   └── Вывод (/sell/finance/withdraw)                     │
│  │   └── Настройки магазина (/sell/settings)                   │
│  │                                                               │
│  └── 👤 Профиль (/profile)                                      │
│      ├── Настройки (/profile/settings)                          │
│      ├── Безопасность (/profile/security)                      │
│      ├── Уведомления (/profile/notifications)                  │
│      ├── Баланс (/profile/balance)                              │
│      └── Избранное (/profile/favorites)                         │
│                                                                  │
│  ADMIN AREA (/admin)                                            │
│  ├── Модерация объявлений                                       │
│  ├── Управление спорами                                         │
│  ├── Пользователи                                              │
│  ├── Игры и категории                                          │
│  ├── Финансы                                                  │
│  ├── Статистика                                               │
│  └── Настройки системы                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 Navigation Components

```
┌─────────────────────────────────────────────────────────────────┐
│                    NAVIGATION SYSTEM                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  GLOBAL HEADER                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 🏠 Logo │ 🔍 Search Bar │ 📂 Catalog │ 💰 Sell │ 👤 User │   │
│  │         │                │            │        │  Menu  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  BREADCRUMBS (Dynamic, based on current location)               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 🏠 Home > 📂 Catalog > 🎮 Genshin Impact > 📦 Listing     │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  CONTEXTUAL NAVIGATION (Varies by page type)                     │
│                                                                  │
│  [Catalog Page]                                                  │
│  ┌──────────────┐  ┌───────────────────────────────────────┐    │
│  │ SIDEBAR      │  │ MAIN CONTENT                          │    │
│  │ ┌──────────┐ │  │ ┌─────────┐ ┌─────────┐ ┌─────────┐  │    │
│  │ │Games     │ │  │ │Listing 1│ │Listing 2│ │Listing 3│  │    │
│  │ │ Genshin  │ │  │ └─────────┘ └─────────┘ └─────────┘  │    │
│  │ │ Valorant │ │  │ ┌─────────┐ ┌─────────┐ ┌─────────┐  │    │
│  │ │ CS2      │ │  │ │Listing 4│ │Listing 5│ │Listing 6│  │    │
│  │ │ WoW      │ │  │ └─────────┘ └─────────┘ └─────────┘  │    │
│  │ └──────────┘ │  │                                       │    │
│  │ ┌──────────┐ │  └───────────────────────────────────────┘    │
│  │ │Filters   │ │                                                │
│  │ │ Price    │ │                                                │
│  │ │ Region   │ │                                                │
│  │ │ Rating   │ │                                                │
│  │ └──────────┘ │                                                │
│  └──────────────┘                                                │
│                                                                  │
│  [Dashboard - Seller]                                           │
│  ┌──────────────┐  ┌───────────────────────────────────────┐    │
│  │ DASHBOARD    │  │ MAIN CONTENT                          │    │
│  │ NAV          │  │                                       │    │
│  │ ┌──────────┐ │  │ [Dynamic based on section]            │    │
│  │ │📊 Stats  │ │  │                                       │    │
│  │ │📦 Orders│ │  │                                       │    │
│  │ │📝 Listings   │  │                                       │    │
│  │ │💰 Finance│ │  │                                       │    │
│  │ │⚙️ Settings   │  │                                       │    │
│  │ └──────────┘ │  └───────────────────────────────────────┘    │
│  └──────────────┘                                                │
│                                                                  │
│  MOBILE NAVIGATION                                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ [Top Bar: Menu Icon | Logo | Search Icon | User Icon]    │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              [Content Area]                               │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ [🏠] [🔍] [🛒] [💰 Sell] [👤]                            │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 Search UX Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    SEARCH USER FLOW                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. USER OPENS SEARCH                                            │
│     │                                                            │
│     ├── Focus on search input                                   │
│     ├── Show recent searches                                    │
│     ├── Show trending searches                                  │
│     └── Show popular games                                      │
│                                                                  │
│  2. USER TYPES QUERY                                             │
│     │                                                            │
│     ├── Real-time suggestions (debounced)                       │
│     │   ├── Game suggestions                                    │
│     │   ├── Category suggestions                                │
│     │   ├── Listing title suggestions                           │
│     │   └── Recent similar searches                             │
│     │                                                            │
│     └── AI Query Understanding                                  │
│         ├── Intent detection (buy, browse, compare)            │
│         ├── Entity extraction (game, type, attributes)          │
│         └── Query enrichment                                    │
│                                                                  │
│  3. USER PRESSES ENTER/CLICKS SUGGESTION                         │
│     │                                                            │
│     ├── Navigate to search results page                         │
│     │                                                            │
│     └── Pre-fill filters based on AI understanding              │
│         ├── "Genshin аккаунт AR 55"                             │
│         │   └── game: genshin-impact, type: account, ar: 55+    │
│         │                                                        │
│         └── "Дешевый Valorant radiant"                          │
│             └── game: valorant, rank: radiant, sort: price_asc  │
│                                                                  │
│  4. SEARCH RESULTS PAGE                                          │
│     │                                                            │
│     ├── Applied filters (chips)                                 │
│     ├── Active filters sidebar                                  │
│     ├── Results grid/list toggle                                │
│     ├── Sort dropdown                                           │
│     ├── Pagination/Infinite scroll                              │
│     └── Save search button (notifications for new matches)      │
│                                                                  │
│  5. USER INTERACTS WITH RESULTS                                  │
│     │                                                            │
│     ├── Click listing → View listing page                        │
│     ├── Quick view (hover/press) → Preview modal                │
│     ├── Add to favorites                                        │
│     ├── Add to cart                                             │
│     └── Contact seller                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.4 Listing Creation Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                 LISTING CREATION FLOW                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  STEP 1: SELECT GAME                                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Search or select from list                              │   │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐           │   │
│  │  │Genshin│ │Valorant│ │ CS2 │ │ WoW │ │  ...│           │   │
│  │  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  STEP 2: SELECT PRODUCT TYPE                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Available types for selected game:                      │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │   │
│  │  │ 📱 Аккаунт│ │💎 Валюта │ │ ⚔️ Предметы│ │ 🎫 Ключи │    │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  STEP 3: FILL ATTRIBUTES (Dynamic Form)                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Based on game + type, show dynamic fields:               │   │
│  │                                                            │   │
│  │  [Genshin Impact - Account]                               │   │
│  │  ┌────────────────────────────────────────────────────┐   │   │
│  │  │ Сервер:        [▼ America          ]               │   │   │
│  │  │ AR:            [    60    ]                         │   │   │
│  │  │ Персонажи:     [+ Hu Tao, Raiden, Nahida]           │   │   │
│  │  │ Примогемы:     [  50000  ]                          │   │   │
│  │  │ Доступ к почте:[✓]                                  │   │   │
│  │  │ Платформа:     [✓] PC [✓] Mobile [ ] PS5           │   │   │
│  │  └────────────────────────────────────────────────────┘   │   │
│  │                                                            │   │
│  │  [Validation happens in real-time]                        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  STEP 4: MEDIA & DESCRIPTION                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Заголовок:                                               │   │
│  │  ┌────────────────────────────────────────────────────┐   │   │
│  │  │ AR 60 | Hu Tao C1 + Raiden + Nahida | 50K Primos  │   │   │
│  │  └────────────────────────────────────────────────────┘   │   │
│  │  [AI Suggestion: Make title more SEO-friendly]            │   │
│  │                                                            │   │
│  │  Описание:                                                │   │
│  │  ┌────────────────────────────────────────────────────┐   │   │
│  │  │                                                    │   │   │
│  │  │  [Markdown editor]                                 │   │   │
│  │  │                                                    │   │   │
│  │  └────────────────────────────────────────────────────┘   │   │
│  │                                                            │   │
│  │  Фотографии:                                              │   │
│  │  [+] [+] [+] [Drag to reorder]                            │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  STEP 5: PRICING & DELIVERY                                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Цена:            [   5000   ] RUB                        │   │
│  │  [Market suggestion: Similar listings: 4800-5500 RUB]    │   │
│  │                                                            │   │
│  │  Наличие:                                                 │   │
│  │  ○ Неограничено                                           │   │
│  │  ● Ограничено: [ 1 ] шт.                                  │   │
│  │  ○ Уникальный товар                                        │   │
│  │                                                            │   │
│  │  Доставка:                                                │   │
│  │  ● Ручная (связаться с покупателем)                       │   │
│  │  ○ Автоматическая                                         │   │
│  │                                                            │   │
│  │  Время доставки: [ 60 ] минут                             │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  STEP 6: REVIEW & PUBLISH                                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Preview of listing                                       │   │
│  │  Estimated fees: 500 RUB (10%)                            │   │
│  │  [Save Draft] [Publish]                                   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. API Дизайн

### 7.1 API Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     API ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PUBLIC API (REST + GraphQL)                                     │
│  └── Для клиентов (web, mobile, third-party)                    │
│                                                                  │
│  INTERNAL API (gRPC)                                             │
│  └── Для межсервисного общения                                  │
│                                                                  │
│  WEBHOOKS                                                        │
│  └── Для интеграций и уведомлений                               │
│                                                                  │
│  WEBSOCKETS                                                      │
│  └── Для real-time функционала (чат, уведомления)                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 RESTful API Structure

```
/api/v1/
├── /auth
│   ├── POST   /register              # Регистрация
│   ├── POST   /login                 # Авторизация
│   ├── POST   /logout                # Выход
│   ├── POST   /refresh               # Обновление токена
│   ├── POST   /forgot-password       # Сброс пароля
│   ├── POST   /reset-password        # Установка нового пароля
│   ├── POST   /verify-email          # Верификация email
│   └── POST   /2fa/...               # 2FA endpoints
│
├── /users
│   ├── GET    /me                    # Текущий пользователь
│   ├── PATCH  /me                    # Обновление профиля
│   ├── GET    /me/listings           # Мои объявления
│   ├── GET    /me/orders             # Мои заказы
│   ├── GET    /me/favorites          # Избранное
│   ├── GET    /me/transactions       # История транзакций
│   │
│   └── /{id}
│       ├── GET                       # Профиль пользователя (public)
│       ├── GET    /listings          # Объявления пользователя
│       └── GET    /reviews           # Отзывы пользователя
│
├── /games
│   ├── GET                           # Список игр
│   ├── GET    /popular               # Популярные игры
│   ├── GET    /{slug}                # Детали игры
│   └── GET    /{slug}/product-types  # Типы товаров для игры
│
├── /listings
│   ├── GET                           # Поиск/список объявлений
│   ├── POST                          # Создать объявление
│   │
│   └── /{id}
│       ├── GET                       # Детали объявления
│       ├── PATCH                     # Обновить
│       ├── DELETE                    # Удалить
│       ├── POST   /favorite          # Добавить в избранное
│       └── POST   /report            # Пожаловаться
│
├── /orders
│   ├── POST                          # Создать заказ
│   │
│   └── /{id}
│       ├── GET                       # Детали заказа
│       ├── POST   /cancel            # Отменить
│       ├── POST   /confirm           # Подтвердить получение
│       ├── POST   /dispute           # Открыть спор
│       └── POST   /review            # Оставить отзыв
│
├── /payments
│   ├── POST   /create                # Создать платеж
│   ├── POST   /callback              # Webhook от платежной системы
│   │
│   └── /withdrawals
│       ├── GET                       # История выводов
│       └── POST                      # Запросить вывод
│
├── /conversations
│   ├── GET                           # Список чатов
│   │
│   └── /{id}
│       ├── GET                       # История сообщений
│       └── POST   /messages          # Отправить сообщение
│
├── /notifications
│   ├── GET                           # Список уведомлений
│   ├── POST   /{id}/read             # Пометить прочитанным
│   └── POST   /read-all              # Пометить все прочитанными
│
└── /admin
    ├── /users/...
    ├── /listings/...
    ├── /orders/...
    ├── /disputes/...
    └── /games/...
```

### 7.3 API Response Format

```typescript
// Standard success response
interface ApiResponse<T> {
  success: true;
  data: T;
  meta?: {
    page?: number;
    limit?: number;
    total?: number;
    hasMore?: boolean;
  };
}

// Standard error response
interface ApiError {
  success: false;
  error: {
    code: string; // "VALIDATION_ERROR", "NOT_FOUND", etc.
    message: string; // Human-readable message
    details?: unknown; // Additional error details
  };
}

// Pagination
interface PaginatedResponse<T> extends ApiResponse<T[]> {
  meta: {
    page: number;
    limit: number;
    total: number;
    hasMore: boolean;
  };
}
```

### 7.4 GraphQL Schema (Optional - for complex queries)

```graphql
type Query {
  me: User
  user(id: ID!): User
  game(slug: String!): Game
  games(filter: GameFilter, sort: SortInput, page: PageInput): GameConnection!
  listing(id: ID!): Listing
  listings(
    filter: ListingFilter
    sort: SortInput
    page: PageInput
  ): ListingConnection!
  search(query: String!, filters: SearchFilters): SearchResult!
}

type Mutation {
  # Auth
  register(input: RegisterInput!): AuthPayload!
  login(input: LoginInput!): AuthPayload!

  # Listings
  createListing(input: CreateListingInput!): Listing!
  updateListing(id: ID!, input: UpdateListingInput!): Listing!

  # Orders
  createOrder(input: CreateOrderInput!): Order!
  confirmOrder(id: ID!): Order!
  cancelOrder(id: ID!): Order!

  # Payments
  createPayment(input: CreatePaymentInput!): Payment!
  requestWithdrawal(input: WithdrawalInput!): Withdrawal!
}

type Subscription {
  onMessage(conversationId: ID!): Message!
  onNotification: Notification!
  onOrderUpdate(orderId: ID!): Order!
}
```

---

## 8. База Данных

### 8.1 Database Design Principles

1. **Normalized Design** - Избегать дублирования данных
2. **JSONB for Flexibility** - Использовать JSONB для динамических атрибутов
3. **Partitioning** - Партиционирование для больших таблиц
4. **Read Replicas** - Репликация для чтения
5. **Connection Pooling** - PgBouncer для управления соединениями

### 8.2 Indexing Strategy

```sql
-- Users
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_reputation ON users(reputation_score DESC);

-- Listings
CREATE INDEX idx_listings_game ON listings(game_id);
CREATE INDEX idx_listings_seller ON listings(seller_id);
CREATE INDEX idx_listings_status ON listings(status);
CREATE INDEX idx_listings_price ON listings(price);
CREATE INDEX idx_listings_created ON listings(created_at DESC);

-- Full-text search on title
CREATE INDEX idx_listings_title_search ON listings
  USING gin(to_tsvector('russian', title));

-- JSONB attributes index (GIN for flexible queries)
CREATE INDEX idx_listings_attributes ON listings
  USING gin(attributes);

-- Orders
CREATE INDEX idx_orders_buyer ON orders(buyer_id);
CREATE INDEX idx_orders_seller ON orders(seller_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created ON orders(created_at DESC);

-- Transactions
CREATE INDEX idx_transactions_user ON transactions(user_id);
CREATE INDEX idx_transactions_status ON transactions(status);
CREATE INDEX idx_transactions_created ON transactions(created_at DESC);
```

### 8.3 Data Migration Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                  MIGRATION STRATEGY                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Version Control                                              │
│     └── Все миграции в Git репозитории                          │
│                                                                  │
│  2. Naming Convention                                            │
│     └── YYYYMMDDHHMM_description.sql                            │
│         Example: 202603211500_create_users_table.sql            │
│                                                                  │
│  3. Reversible Migrations                                        │
│     └── Каждая миграция должна иметь UP и DOWN                  │
│                                                                  │
│  4. Testing                                                      │
│     └── Тестировать на копии продакшн данных                    │
│                                                                  │
│  5. Zero-Downtime Deployments                                   │
│     └── Обратная совместимость при изменении схемы              │
│                                                                  │
│  6. Rollback Plan                                                │
│     └── План отката на случай проблем                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. Интеграции

### 9.1 Payment Gateway Integration

```
┌─────────────────────────────────────────────────────────────────┐
│                  PAYMENT INTEGRATION                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  PAYMENT FLOW                              │   │
│  │                                                            │   │
│  │  1. User initiates payment                                │   │
│  │     └── POST /api/v1/payments/create                      │   │
│  │                                                            │   │
│  │  2. Backend creates payment in gateway                    │   │
│  │     └── Gateway returns payment URL/QR                    │   │
│  │                                                            │   │
│  │  3. User redirects to payment page                        │   │
│  │     └── User completes payment                            │   │
│  │                                                            │   │
│  │  4. Gateway sends webhook                                 │   │
│  │     └── POST /api/v1/payments/callback                    │   │
│  │                                                            │   │
│  │  5. Backend verifies & processes                          │   │
│  │     └── Update transaction status                         │   │
│  │     └── Credit user balance / Create escrow               │   │
│  │     └── Send notifications                                │   │
│  │                                                            │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  SUPPORTED GATEWAYS:                                             │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   ЮKassa     │  │  Tinkoff     │  │  Cryptomus   │          │
│  │              │  │              │  │              │          │
│  │ Cards        │  │ Cards        │  │ Crypto       │          │
│  │ SBP          │  │ SBP          │  │ USDT/BTC/ETH │          │
│  │ YooMoney     │  │ Tinkoff Pay  │  │              │          │
│  │ QIWI         │  │              │  │              │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  WITHDRAWAL METHODS:                                             │
│  ├── Bank Cards (RU)                                            │
│  ├── SBP                                                        │
│  ├── ЮMoney                                                     │
│  ├── QIWI                                                       │
│  └── Crypto (USDT TRC20/ERC20, BTC, ETH)                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 Telegram Integration

```
┌─────────────────────────────────────────────────────────────────┐
│                  TELEGRAM INTEGRATION                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. NOTIFICATION BOT                                             │
│     ├── Уведомления о заказах                                   │
│     ├── Уведомления о сообщениях                                │
│     ├── Уведомления о спорах                                    │
│     └── Быстрые действия (кнопки в сообщениях)                  │
│                                                                  │
│  2. MINI APP (Web App)                                           │
│     ├── Полноценный интерфейс внутри Telegram                   │
│     ├── Авторизация через Telegram                              │
│     ├── Быстрый доступ к профилю                                │
│     └── Push-уведомления                                        │
│                                                                  │
│  3. AUTH WIDGET                                                  │
│     ├── Login via Telegram на сайте                            │
│     └── Автоматическая верификация                             │
│                                                                  │
│  NOTIFICATION MESSAGE EXAMPLE:                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 🛒 Новый заказ #12345                                    │   │
│  │                                                            │   │
│  │ Товар: Genshin Impact AR 60                               │   │
│  │ Цена: 5,000 ₽                                             │   │
│  │                                                            │   │
│  │ [📋 Детали] [💬 Написать] [✅ Отметить доставленным]       │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 9.3 Game API Integrations (Future)

```
┌─────────────────────────────────────────────────────────────────┐
│                    GAME API INTEGRATIONS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Для автоматической верификации аккаунтов:                      │
│                                                                  │
│  1. RIOT GAMES (Valorant, LoL)                                  │
│     └── Riot ID verification                                    │
│                                                                  │
│  2. VALVE (Steam, CS2)                                          │
│     └── Steam API for inventory/rank verification               │
│                                                                  │
│  3. BLIZZARD (WoW, Overwatch)                                   │
│     └── Battle.net API                                          │
│                                                                  │
│  4. MOBILE GAMES                                                │
│     └── Account screenshot verification (OCR + AI)              │
│                                                                  │
│  VERIFICATION FLOW:                                              │
│  1. Seller provides account credentials/link                    │
│  2. System makes API call to game                              │
│  3. Verify attributes match listing                            │
│  4. Store verification proof                                   │
│  5. Display "Verified" badge on listing                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 10. Безопасность

### 10.1 Security Layers

```
┌─────────────────────────────────────────────────────────────────┐
│                    SECURITY ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  LAYER 1: NETWORK SECURITY                                       │
│  ├── DDoS Protection (Cloudflare)                               │
│  ├── WAF (Web Application Firewall)                             │
│  ├── Rate Limiting                                              │
│  └── IP Reputation Checks                                       │
│                                                                  │
│  LAYER 2: AUTHENTICATION                                         │
│  ├── JWT Tokens (access + refresh)                              │
│  ├── 2FA (TOTP, SMS, Email)                                     │
│  ├── Device Fingerprinting                                      │
│  ├── Session Management                                         │
│  └── OAuth (Google, Telegram)                                   │
│                                                                  │
│  LAYER 3: AUTHORIZATION                                          │
│  ├── Role-Based Access Control (RBAC)                           │
│  ├── Resource-Level Permissions                                 │
│  └── API Scopes                                                 │
│                                                                  │
│  LAYER 4: DATA SECURITY                                          │
│  ├── Encryption at Rest (AES-256)                               │
│  ├── Encryption in Transit (TLS 1.3)                            │
│  ├── PII Protection                                             │
│  └── Secure Backup                                              │
│                                                                  │
│  LAYER 5: APPLICATION SECURITY                                   │
│  ├── Input Validation                                           │
│  ├── SQL Injection Prevention                                   │
│  ├── XSS Prevention                                              │
│  ├── CSRF Protection                                            │
│  ├── CORS Configuration                                         │
│  └── Content Security Policy                                    │
│                                                                  │
│  LAYER 6: MONITORING                                              │
│  ├── Security Event Logging                                     │
│  ├── Anomaly Detection                                          │
│  ├── Fraud Detection (ML)                                       │
│  └── Incident Response                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 Fraud Prevention

```typescript
interface FraudPrevention {
  // User behavior analysis
  behaviorAnalysis: {
    velocityChecks: {
      maxRegistrationsPerIP: 5; // per hour
      maxLoginAttempts: 10; // per hour
      maxOrdersPerUser: 20; // per day
      maxListingCreations: 50; // per day
    };

    patternDetection: {
      multipleAccountsSameDevice: boolean;
      suspiciousLoginPatterns: boolean;
      unusualTransactionPatterns: boolean;
    };
  };

  // Transaction monitoring
  transactionMonitoring: {
    highValueThreshold: 50000; // RUB
    velocityChecks: {
      maxWithdrawalsPerDay: 3;
      maxWithdrawalAmount: 100000; // RUB
    };
    manualReviewThresholds: {
      firstWithdrawal: true;
      highValueTransaction: 30000; // RUB
      newBuyerHighValue: 10000; // RUB
    };
  };

  // Device fingerprinting
  deviceFingerprint: {
    collectCanvas: true;
    collectWebGL: true;
    collectAudio: true;
    collectFonts: true;
    collectScreen: true;
    collectTimezone: true;
  };

  // ML-based detection
  mlDetection: {
    chargebackPrediction: true;
    fraudScore: true; // 0-100 score
    riskLevel: "low" | "medium" | "high";
  };
}
```

---

## 11. Масштабирование

### 11.1 Scalability Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                  SCALABILITY ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  HORIZONTAL SCALING                                              │
│  ├── Stateless services (easy to scale)                         │
│  ├── Load balancer (sticky sessions optional)                   │
│  ├── Auto-scaling based on metrics                              │
│  └── Container orchestration (Kubernetes)                       │
│                                                                  │
│  DATABASE SCALING                                                │
│  ├── Read replicas                                              │
│  ├── Connection pooling (PgBouncer)                             │
│  ├── Partitioning (by time/game)                                │
│  └── Sharding (future, if needed)                               │
│                                                                  │
│  CACHING STRATEGY                                                │
│  ├── Application cache (Redis)                                  │
│  │   ├── User sessions                                         │
│  │   ├── Rate limiting counters                                 │
│  │   └── Frequently accessed data                               │
│  │                                                              │
│  ├── CDN (Cloudflare)                                           │
│  │   ├── Static assets                                         │
│  │   ├── Images                                                │
│  │   └── API responses (cacheable)                             │
│  │                                                              │
│  └── Database query cache                                       │
│      ├── Materialized views                                     │
│      └── Pre-computed aggregations                              │
│                                                                  │
│  ASYNC PROCESSING                                                │
│  ├── Message queue (NATS/RabbitMQ)                              │
│  │   ├── Notifications                                         │
│  │   ├── Search index updates                                  │
│  │   └── Analytics                                             │
│  │                                                              │
│  └── Background workers                                        │
│      ├── Image processing                                       │
│      ├── Report generation                                      │
│      └── Cleanup tasks                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 11.2 Performance Targets

| Metric                         | Target    | Measurement       |
| ------------------------------ | --------- | ----------------- |
| API Response Time (p95)        | < 200ms   | Prometheus        |
| API Response Time (p99)        | < 500ms   | Prometheus        |
| Time to First Byte (TTFB)      | < 100ms   | Lighthouse        |
| Largest Contentful Paint (LCP) | < 2.5s    | Lighthouse        |
| Database Query Time            | < 50ms    | APM               |
| Search Query Time              | < 100ms   | Elasticsearch     |
| WebSocket Latency              | < 50ms    | Custom monitoring |
| Concurrent Users               | 10,000    | Load testing      |
| Requests per Second            | 1,000 RPS | Load testing      |

---

## 12. Feature Flags

### 12.1 Назначение и Преимущества

```
┌─────────────────────────────────────────────────────────────────┐
│                    FEATURE FLAGS SYSTEM                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ЗАЧЕМ НУЖНЫ?                                                    │
│  ├── Gradual Rollout — выкатка на 5% → 25% → 50% → 100%         │
│  ├── A/B Testing — тестирование разных вариантов                │
│  ├── Quick Rollback — отключение фичи без редеплоя              │
│  ├── Canary Releases — тестирование на части пользователей      │
│  └── Kill Switch — экстренное отключение проблемной фичи        │
│                                                                  │
│  ПРИМЕРЫ ИСПОЛЬЗОВАНИЯ:                                          │
│  ├── 'key_marketplace' — биржа ключей (5% пользователей)        │
│  ├── 'new_checkout_flow' — новый чекаут (A/B тест)              │
│  ├── 'auto_delivery_v2' — новая авто-выдача (canary)            │
│  └── 'maintenance_mode' — режим обслуживания (kill switch)      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 12.2 Архитектура Feature Flags

```typescript
interface FeatureFlag {
  key: string; // Уникальный идентификатор
  name: string; // Читаемое название
  description: string; // Описание фичи
  enabled: boolean; // Глобально включена?
  rolloutPercentage: number; // 0-100, процент пользователей
  whitelistUserIds?: string[]; // Белый список пользователей
  blacklistUserIds?: string[]; // Чёрный список
  variants?: {
    // Для A/B тестирования
    name: string;
    percentage: number;
    config: Record<string, unknown>;
  }[];
  createdAt: Date;
  updatedAt: Date;
  createdBy: string;
}

interface FeatureFlagService {
  isEnabled(key: string, userId?: string): Promise<boolean>;
  getVariant(key: string, userId: string): Promise<string | null>;
  getVariantConfig(
    key: string,
    variant: string,
  ): Promise<Record<string, unknown>>;
}
```

### 12.3 Реализация

```typescript
// Пример использования в коде
class OrderService {
  constructor(private featureFlags: FeatureFlagService) {}

  async createOrder(dto: CreateOrderDto) {
    // Проверяем включена ли новая система авто-выдачи
    if (await this.featureFlags.isEnabled('auto_delivery_v2', dto.userId)) {
      return this.createOrderV2(dto);
    }
    return this.createOrderV1(dto);
  }
}

// Пример использования на фронтенде
function CheckoutPage() {
  const isNewCheckout = useFeatureFlag('new_checkout_flow');

  if (isNewCheckout) {
    return <CheckoutFlowV2 />;
  }
  return <CheckoutFlowV1 />;
}
```

### 12.4 Хранение

```
┌─────────────────────────────────────────────────────────────────┐
│                    STORAGE OPTIONS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ВАРИАНТ 1: Redis (Рекомендуется для старта)                    │
│  ├── Быстрое чтение                                              │
│  ├── Кэширование на уровне приложения                           │
│  └── Минимальные задержки                                        │
│                                                                  │
│  ВАРИАНТ 2: PostgreSQL + Redis Cache                            │
│  ├── Персистентное хранение                                      │
│  ├── История изменений                                           │
│  └── Кэширование для производительности                         │
│                                                                  │
│  ВАРИАНТ 3: Внешние сервисы (при масштабировании)               │
│  ├── LaunchDarkly                                               │
│  ├── Unleash (Open Source)                                      │
│  └── Flagsmith                                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 12.5 SQL Schema для Feature Flags

```sql
CREATE TABLE feature_flags (
    id UUID PRIMARY KEY,
    key VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    enabled BOOLEAN DEFAULT true,
    rollout_percentage INT DEFAULT 100 CHECK (rollout_percentage >= 0 AND rollout_percentage <= 100),
    whitelist_user_ids UUID[],
    blacklist_user_ids UUID[],
    variants JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    created_by UUID REFERENCES users(id)
);

CREATE TABLE feature_flag_logs (
    id UUID PRIMARY KEY,
    flag_id UUID REFERENCES feature_flags(id),
    action VARCHAR(50) NOT NULL,  -- 'created', 'updated', 'enabled', 'disabled'
    old_value JSONB,
    new_value JSONB,
    changed_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW()
);
```

### 12.6 Примеры Feature Flags для AvanPay

| Флаг                  | Описание                        | Rollout     |
| --------------------- | ------------------------------- | ----------- |
| `key_marketplace`     | Биржа игровых ключей            | 5% → 100%   |
| `auto_delivery_v2`    | Новая система авто-выдачи       | Canary      |
| `new_checkout_flow`   | Новый процесс оформления заказа | A/B 50/50   |
| `telegram_mini_app`   | Telegram Mini App               | 10% → 100%  |
| `crypto_payments`     | Криптовалютные платежи          | 100%        |
| `seller_analytics_v2` | Новая аналитика для продавцов   | 25% → 100%  |
| `maintenance_mode`    | Режим обслуживания              | Kill Switch |
| `read_only_mode`      | Только чтение (без покупок)     | Kill Switch |

---

## 13. Telegram Mini App (TMA)

### 13.1 Стратегия: Shared Core, Different Shells

```
┌─────────────────────────────────────────────────────────────────┐
│              ЕДИНЫЙ NEXT.JS ПРОЕКТ                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ПРИНЦИП:                                                        │
│  ├── Один кодbase для Web и TMA                                 │
│  ├── Единый стейт-менеджмент (корзина, чаты, фильтры)           │
│  ├── Общие компоненты (карточка товара = один компонент)        │
│  └── Мгновенные обновления (поправили баг — исправилось везде) │
│                                                                  │
│  ОПРЕДЕЛЕНИЕ СРЕДЫ:                                              │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  // lib/platform.ts                                         ││
│  │  export function detectPlatform() {                         ││
│  │    if (typeof window !== 'undefined') {                     ││
│  │      if (window.Telegram?.WebApp) {                         ││
│  │        return 'tma';  // Telegram Mini App                  ││
│  │      }                                                      ││
│  │      if (/Mobile/.test(navigator.userAgent)) {              ││
│  │        return 'mobile';                                     ││
│  │      }                                                      ││
│  │      return 'web';                                          ││
│  │    }                                                        ││
│  │    return 'ssr';                                            ││
│  │  }                                                          ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 13.2 Различия Web vs TMA

```
┌─────────────────────────────────────────────────────────────────┐
│                    WEB vs TMA COMPARISON                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────┬─────────────────────────────────┐  │
│  │         WEB             │           TMA                   │  │
│  ├─────────────────────────┼─────────────────────────────────┤  │
│  │ Полный Header/Footer    │ Скрыты, используются TG-элементы│  │
│  │ SEO-оптимизация         │ Не нужна (закрытая экосистема)  │  │
│  │ Стандартные цвета       │ Telegram CSS переменные         │  │
│  │ Обычные кнопки          │ MainButton, BackButton (TG API) │  │
│  │ Кастомная навигация     │ Закрытие через tg.close()       │  │
│  │ Email/Password логин    │ Авто-логин через initData       │  │
│  │ Browser notifications   │ Telegram notifications          │  │
│  │ Haptic feedback нет     │ tg.HapticFeedback.*             │  │
│  └─────────────────────────┴─────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 13.3 Telegram CSS Variables

```css
/* Автоматически применяются в TMA */
:root {
  --tg-theme-bg-color: #ffffff;
  --tg-theme-text-color: #000000;
  --tg-theme-hint-color: #999999;
  --tg-theme-link-color: #2481cc;
  --tg-theme-button-color: #2481cc;
  --tg-theme-button-text-color: #ffffff;
  --tg-theme-secondary-bg-color: #f0f0f0;
}

/* Использование в компонентах */
.button-primary {
  background-color: var(--tg-theme-button-color, #3b82f6);
  color: var(--tg-theme-button-text-color, #ffffff);
}
```

### 13.4 TMA-Specific Components

```typescript
// components/tma/TmaProvider.tsx
'use client';

import { useEffect, useState } from 'react';
import { detectPlatform } from '@/lib/platform';

export function TmaProvider({ children }: { children: React.ReactNode }) {
  const [isTma, setIsTma] = useState(false);

  useEffect(() => {
    const platform = detectPlatform();
    setIsTma(platform === 'tma');

    if (platform === 'tma') {
      const tg = window.Telegram.WebApp;

      // Инициализация TMA
      tg.ready();
      tg.expand();

      // Применяем тему Telegram
      document.documentElement.style.setProperty(
        '--tg-theme-bg-color',
        tg.themeParams.bg_color || '#ffffff'
      );
      // ... остальные переменные

      // Настраиваем главную кнопку
      tg.MainButton.setParams({
        text: 'Купить',
        color: tg.themeParams.button_color,
      });
    }
  }, []);

  return (
    <div className={isTma ? 'tma-mode' : 'web-mode'}>
      {children}
    </div>
  );
}
```

### 13.5 Unified Auth Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    UNIFIED AUTH FLOW                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  СЦЕНАРИЙ 1: Вход через TMA                                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  1. Пользователь открывает TMA                              ││
│  │  2. Получаем window.Telegram.WebApp.initData                ││
│  │  3. Бэкенд проверяет подпись (hash)                         ││
│  │  4. Находим/создаем user по telegram_id                     ││
│  │  5. Выдаём JWT токен                                        ││
│  │  6. Пользователь авторизован                                ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  СЦЕНАРИЙ 2: Вход через Web (позже привязка TMA)                │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  1. Вход через Email/Password или Google                    ││
│  │  2. В настройках: "Привязать Telegram"                      ││
│  │  3. Открывается Telegram Bot с deep link                    ││
│  │  4. Бот отправляет callback на бэкенд                       ││
│  │  5. Создаётся запись в user_connections                     ││
│  │  6. Теперь можно входить через TMA                          ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  СЦЕНАРИЙ 3: QR-код для входа (Web)                             │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  1. На странице входа: "Войти через Telegram"               ││
│  │  2. Показываем QR-код с уникальным токеном                  ││
│  │  3. Пользователь сканирует ботом                            ││
│  │  4. Бот подтверждает авторизацию                            ││
│  │  5. Web-страница автоматически логинится                    ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 13.6 Тестирование TMA

```
┌─────────────────────────────────────────────────────────────────┐
│                    TMA TESTING STRATEGY                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. ЛОКАЛЬНАЯ РАЗРАБОТКА                                         │
│  ├── ngrok / cloudflared для проброса localhost                 │
│  ├── Настройка Bot Father: /setmenubutton → TMA URL             │
│  └── Тестирование в Telegram Desktop / Mobile                   │
│                                                                  │
│  2. ТЕСТИРОВАНИЕ БЕЗ TELEGRAM                                    │
│  ├── Mock-режим для разработки                                  │
│  │   ┌────────────────────────────────────────────────────────┐│
│  │   │ // lib/tg-mock.ts                                     ││
│  │   │ window.Telegram = {                                   ││
│  │   │   WebApp: {                                           ││
│  │   │     initData: 'user_id=123&...',                      ││
│  │   │     themeParams: { bg_color: '#ffffff', ... },        ││
│  │   │     MainButton: { show: () => {}, hide: () => {} },   ││
│  │   │     HapticFeedback: { impactOccurred: () => {} },     ││
│  │   │     ready: () => {},                                  ││
│  │   │     expand: () => {},                                 ││
│  │   │   }                                                   ││
│  │   │ };                                                    ││
│  │   └────────────────────────────────────────────────────────┘│
│  └── Позволяет разрабатывать без установки Telegram             │
│                                                                  │
│  3. ТЕСТИРОВАНИЕ НА STAGING                                      │
│  ├── Отдельный Telegram Bot для staging                         │
│  ├── Feature Flag для ограничения доступа                       │
│  └── Тестирование на реальных устройствах                       │
│                                                                  │
│  4. АВТОМАТИЧЕСКИЕ ТЕСТЫ                                         │
│  ├── Playwright / Cypress с mock-объектом Telegram              │
│  ├── Unit-тесты для TMA-компонентов                             │
│  └── Интеграционные тесты для auth flow                         │
│                                                                  │
│  5. ПРОДАКШН РОЛЛАУТ                                             │
│  ├── Feature Flag: telegram_mini_app                            │
│  ├── Rollout: 5% → 25% → 50% → 100%                             │
│  └── Мониторинг ошибок и метрик                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 13.7 Bot Commands для AvanPay

```
/start - Открыть Mini App
/help - Справка по использованию
/orders - Мои заказы
/sales - Мои продажи (для продавцов)
/balance - Баланс и вывод
/support - Связаться с поддержкой
/link - Привязать Web-аккаунт
```

---

## 14. Design System

### 14.1 Принципы

```
┌─────────────────────────────────────────────────────────────────┐
│                    DESIGN SYSTEM PRINCIPLES                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. ATOMIC DESIGN                                                │
│     └── Atoms → Molecules → Organisms → Templates → Pages       │
│                                                                  │
│  2. CONSISTENCY                                                  │
│     └── Единый визуальный язык на всех платформах               │
│                                                                  │
│  3. REUSABILITY                                                  │
│     └── Один компонент — множество использований                │
│                                                                  │
│  4. ACCESSIBILITY                                                │
│     └── WCAG 2.1 AA совместимость                               │
│                                                                  │
│  5. THEMING                                                      │
│     └── Light/Dark mode + Telegram theme support                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 14.2 Структура UI Kit

```
packages/ui/
├── src/
│   ├── atoms/                    # Базовые компоненты
│   │   ├── Button/
│   │   │   ├── Button.tsx
│   │   │   ├── Button.styles.ts
│   │   │   ├── Button.types.ts
│   │   │   ├── Button.test.tsx
│   │   │   └── index.ts
│   │   ├── Input/
│   │   ├── Badge/
│   │   ├── Avatar/
│   │   ├── Icon/
│   │   ├── Spinner/
│   │   ├── Tooltip/
│   │   └── Typography/
│   │
│   ├── molecules/               # Комбинации атомов
│   │   ├── SearchBar/
│   │   ├── UserCard/
│   │   ├── ListingCard/
│   │   ├── PriceTag/
│   │   ├── Rating/
│   │   └── FilterChip/
│   │
│   ├── organisms/               # Сложные компоненты
│   │   ├── Header/
│   │   ├── Footer/
│   │   ├── Sidebar/
│   │   ├── ListingGrid/
│   │   ├── OrderCard/
│   │   ├── ChatWindow/
│   │   └── CheckoutForm/
│   │
│   ├── templates/               # Шаблоны страниц
│   │   ├── CatalogTemplate/
│   │   ├── ListingDetailTemplate/
│   │   ├── DashboardTemplate/
│   │   └── CheckoutTemplate/
│   │
│   ├── foundations/             # Базовые стили
│   │   ├── colors.ts
│   │   ├── typography.ts
│   │   ├── spacing.ts
│   │   ├── shadows.ts
│   │   └── breakpoints.ts
│   │
│   ├── hooks/                   # Общие хуки
│   │   ├── useMediaQuery.ts
│   │   ├── useDebounce.ts
│   │   └── useLocalStorage.ts
│   │
│   └── index.ts                 # Экспорт всего
│
├── stories/                     # Storybook stories
│   ├── atoms/
│   ├── molecules/
│   └── organisms/
│
└── package.json
```

### 14.3 Цветовая палитра

```typescript
// packages/ui/src/foundations/colors.ts
export const colors = {
  // Primary
  primary: {
    50: "#eff6ff",
    100: "#dbeafe",
    200: "#bfdbfe",
    300: "#93c5fd",
    400: "#60a5fa",
    500: "#3b82f6", // Основной
    600: "#2563eb",
    700: "#1d4ed8",
    800: "#1e40af",
    900: "#1e3a8a",
  },

  // Success
  success: {
    50: "#f0fdf4",
    500: "#22c55e",
    700: "#15803d",
  },

  // Warning
  warning: {
    50: "#fffbeb",
    500: "#f59e0b",
    700: "#b45309",
  },

  // Error
  error: {
    50: "#fef2f2",
    500: "#ef4444",
    700: "#b91c1c",
  },

  // Neutral
  neutral: {
    0: "#ffffff",
    50: "#f9fafb",
    100: "#f3f4f6",
    200: "#e5e7eb",
    300: "#d1d5db",
    400: "#9ca3af",
    500: "#6b7280",
    600: "#4b5563",
    700: "#374151",
    800: "#1f2937",
    900: "#111827",
  },
};

// Telegram theme integration
export const telegramColors = {
  bg: "var(--tg-theme-bg-color, #ffffff)",
  text: "var(--tg-theme-text-color, #000000)",
  hint: "var(--tg-theme-hint-color, #999999)",
  link: "var(--tg-theme-link-color, #2481cc)",
  button: "var(--tg-theme-button-color, #3b82f6)",
  buttonText: "var(--tg-theme-button-text-color, #ffffff)",
  secondaryBg: "var(--tg-theme-secondary-bg-color, #f0f0f0)",
};
```

### 14.4 Основные компоненты

```
┌─────────────────────────────────────────────────────────────────┐
│                    CORE COMPONENTS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  BUTTON                                                          │
│  ├── Variants: primary, secondary, outline, ghost, danger       │
│  ├── Sizes: sm, md, lg                                          │
│  ├── States: loading, disabled                                  │
│  └── Telegram: использует tg.MainButton в TMA                   │
│                                                                  │
│  INPUT                                                           │
│  ├── Types: text, email, password, search, number               │
│  ├── States: error, success, disabled                           │
│  ├── Addons: icon, suffix, prefix                               │
│  └── Validation: inline error messages                          │
│                                                                  │
│  CARD                                                            │
│  ├── Variants: default, outlined, elevated                      │
│  ├── Parts: header, body, footer                                │
│  └── Interactive: hoverable, clickable                          │
│                                                                  │
│  LISTING CARD                                                    │
│  ├── Image gallery                                               │
│  ├── Price tag                                                   │
│  ├── Seller info                                                 │
│  ├── Rating                                                      │
│  └── Quick actions (favorite, cart)                              │
│                                                                  │
│  USER CARD                                                       │
│  ├── Avatar                                                      │
│  ├── Username + verification badge                              │
│  ├── Reputation score                                            │
│  └── Stats (sales, rating)                                       │
│                                                                  │
│  SEARCH BAR                                                      │
│  ├── Input with icon                                             │
│  ├── Suggestions dropdown                                        │
│  ├── Recent searches                                             │
│  └── Clear button                                                │
│                                                                  │
│  CHAT COMPONENTS                                                 │
│  ├── MessageBubble                                               │
│  ├── ChatInput                                                   │
│  ├── AttachmentPreview                                           │
│  └── TypingIndicator                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 14.5 Пример компонента Button

```typescript
// packages/ui/src/atoms/Button/Button.tsx
import { forwardRef } from 'react';
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '@/utils/cn';
import { Spinner } from '../Spinner';

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-lg font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        primary: 'bg-[var(--tg-theme-button-color,#3b82f6)] text-[var(--tg-theme-button-text-color,#fff)] hover:opacity-90',
        secondary: 'bg-[var(--tg-theme-secondary-bg-color,#f3f4f6)] text-[var(--tg-theme-text-color,#1f2937)] hover:bg-neutral-200',
        outline: 'border border-neutral-300 bg-transparent hover:bg-neutral-100',
        ghost: 'bg-transparent hover:bg-neutral-100',
        danger: 'bg-error-500 text-white hover:bg-error-600',
      },
      size: {
        sm: 'h-8 px-3 text-sm',
        md: 'h-10 px-4 text-base',
        lg: 'h-12 px-6 text-lg',
      },
    },
    defaultVariants: {
      variant: 'primary',
      size: 'md',
    },
  }
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  loading?: boolean;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
}

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, loading, leftIcon, rightIcon, children, disabled, ...props }, ref) => {
    return (
      <button
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        disabled={disabled || loading}
        {...props}
      >
        {loading && <Spinner size="sm" className="mr-2" />}
        {!loading && leftIcon && <span className="mr-2">{leftIcon}</span>}
        {children}
        {!loading && rightIcon && <span className="ml-2">{rightIcon}</span>}
      </button>
    );
  }
);

Button.displayName = 'Button';
```

### 14.6 Storybook интеграция

```typescript
// packages/ui/stories/atoms/Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from '../../src/atoms/Button';

const meta: Meta<typeof Button> = {
  title: 'Atoms/Button',
  component: Button,
  argTypes: {
    variant: {
      control: 'select',
      options: ['primary', 'secondary', 'outline', 'ghost', 'danger'],
    },
    size: {
      control: 'select',
      options: ['sm', 'md', 'lg'],
    },
  },
};

export default meta;
type Story = StoryObj<typeof Button>;

export const Primary: Story = {
  args: {
    children: 'Button',
    variant: 'primary',
  },
};

export const AllVariants: Story = {
  render: () => (
    <div className="flex gap-4">
      <Button variant="primary">Primary</Button>
      <Button variant="secondary">Secondary</Button>
      <Button variant="outline">Outline</Button>
      <Button variant="ghost">Ghost</Button>
      <Button variant="danger">Danger</Button>
    </div>
  ),
};
```

### 14.7 Преимущества Design System

```
┌─────────────────────────────────────────────────────────────────┐
│                    DESIGN SYSTEM BENEFITS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  СКОРОСТЬ РАЗРАБОТКИ                                             │
│  ├── Новая фича собирается из готовых блоков                    │
│  ├── "Аренда аккаунтов" = существующие компоненты               │
│  └── Экономия времени: 40-60% на разработку UI                  │
│                                                                  │
│  КОНСИСТЕНТНОСТЬ                                                 │
│  ├── Единый визуальный стиль                                    │
│  ├── Предсказуемое поведение                                    │
│  └── Улучшенный UX                                              │
│                                                                  │
│  ПОДДЕРЖКА                                                       │
│  ├── Изменение в одном месте                                    │
│  ├── Простой рефакторинг                                        │
│  └── Документация в Storybook                                   │
│                                                                  │
│  ТЕСТИРОВАНИЕ                                                    │
│  ├── Unit-тесты для компонентов                                 │
│  ├── Visual regression тесты                                    │
│  └── Accessibility тесты                                        │
│                                                                  │
│  КОМАНДА                                                         │
│  ├── Общий язык для дизайнеров и разработчиков                  │
│  ├── Быстрый онбординг новых членов                             │
│  └── Параллельная разработка                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 15. Дорожная Карта

### 15.1 Phase 1: MVP (Месяцы 1-3)

**Цель:** Рабочий прототип с базовым функционалом

| Спринт | Функционал                              | Приоритет |
| ------ | --------------------------------------- | --------- |
| 1-2    | Auth (register, login, 2FA)             | 🔴        |
| 1-2    | Database schema & migrations            | 🔴        |
| 3-4    | Game catalog (50 games)                 | 🔴        |
| 3-4    | Product types (Account, Currency, Item) | 🔴        |
| 5-6    | Listing CRUD                            | 🔴        |
| 5-6    | Search (basic)                          | 🔴        |
| 7-8    | Order creation & flow                   | 🔴        |
| 7-8    | Payment (Сбп)                           | 🔴        |
| 9-10   | Escrow system                           | 🔴        |
| 9-10   | Chat in orders                          | 🔴        |
| 11-12  | Telegram notifications                  | 🟡        |
| 11-12  | Admin panel (basic)                     | 🔴        |

**Стек MVP:**

- Frontend: Next.js 14, React 18, TailwindCSS, shadcn/ui
- Backend: Node.js + NestJS или Go
- Database: PostgreSQL 15, Redis 7
- Search: Meilisearch или Elasticsearch
- Storage: MinIO (S3-compatible)
- Payments: Провайдер (????)

### 15.2 Phase 2: Growth (Месяцы 4-6)

**Цель:** Улучшение UX и расширение функционала

| Функционал          | Описание                          | Приоритет |
| ------------------- | --------------------------------- | --------- |
| Telegram Mini App   | Полноценное приложение в Telegram | 🔴        |
| AI Search           | Поиск по естественному языку      | 🟡        |
| Auto-delivery       | Для ключей и топ-апов             | 🟡        |
| Crypto payments     | Cryptomus интеграция              | 🔴        |
| Reviews & Ratings   | Система отзывов                   | 🔴        |
| Favorites & Alerts  | Избранное и уведомления о ценах   | 🟡        |
| Advanced filters    | Умные фильтры по атрибутам        | 🟡        |
| Seller dashboard    | Статистика и аналитика            | 🟡        |
| Dispute system      | Полноценная система споров        | 🔴        |
| Verification levels | Верификация продавцов             | 🔴        |

### 15.3 Phase 3: Scale (Месяцы 7-12)

**Цель:** Масштабирование и новые рынки

| Функционал         | Описание                        | Приоритет |
| ------------------ | ------------------------------- | --------- |
| Mobile App         | React Native приложение         | 🟡        |
| Multi-language     | EN, RU, CN, ES, PT              | 🟡        |
| API for sellers    | Публичное API для автоматизации | 🟡        |
| Vendor accounts    | Аккаунты для магазинов          | 🟡        |
| More games         | 200+ игр                        | 🟡        |
| Advanced analytics | BI для продавцов                | 🟡        |
| Affiliate program  | Реферальная система             | 🟢        |
| White-label        | Решение для других площадок     | 🟢        |

### 15.3.1 Иерархия Категорий

```
  🎮 Игры (Games)
  ├── 📱 Мобильные игры (Mobile)
  │   ├── Gacha (Genshin, Honkai, HSR, WuWa)
  │   ├── MOBA (MLBB, Wild Rift, Brawl Stars)
  │   ├── Battle Royale (PUBG Mobile, Free Fire, COD Mobile)
  │   ├── Strategy (Clash of Clans, Clash Royale)
  │   └── Puzzle/Other
  │
  ├── 💻 PC игры
  │   ├── MMO (WoW, FFXIV, ESO, Lost Ark)
  │   ├── MOBA (LoL, Dota 2)
  │   ├── Shooter (CS2, Valorant, Apex, Warzone)
  │   ├── Battle Royale (Fortnite, PUBG)
  │   ├── RPG (Diablo 4, PoE, Path of Exile 2)
  │   ├── Sandbox (Minecraft, Roblox, Rust)
  │   └── Other
  │
  ├── 🎮 Консоли
  │   ├── PlayStation (PS4/PS5)
  │   ├── Xbox (One/Series X|S)
  │   └── Nintendo Switch
  │
  └── 🌐 Сервисы и ПО
      ├── Стриминг (Netflix, Spotify, Crunchyroll)
      ├── AI (ChatGPT, Claude, Midjourney)
      ├── VPN и Proxy
      ├── Adobe, Microsoft
      └── Другие подписки
```

### 15.3.2 Типы Объявлений (Product Types)

    | Категория | Подкатегории | Авто-доставка | Примеры |
    |-----------|-------------|---------------|---------|
    | **Аккаунты** | Starter, High-level, Rare, Personal | Нет | Genshin starter, Valorant radiant |
    | **Валюта** | Gold, Coins, Gems, Credits | Частично | WoW gold, FC coins, V-Bucks |
    | **Предметы** | Weapons, Skins, Materials, Mounts | Частично | CS2 skins, LoL skins |
    | **Ключи** | Game keys, DLC, Gift cards | Да | Steam keys, PSN cards |
    | **Услуги** | Boosting, Coaching, Powerleveling | Нет | Rank boost, coaching |
    | **Топ-ап** | In-game currency recharge | Да | Genshin top-up, ML diamonds |
    | **NFT/Крипто** | Game NFTs, Tokens | Да | Axie, Gods Unchained |

### 15.4 Technical Debt & Maintenance

Регулярно выделять время на:

- Рефакторинг кода
- Оптимизация запросов
- Обновление зависимостей
- Улучшение тестового покрытия
- Документация
- Monitoring & Alerting

---

## Приложения

### A. Глоссарий

| Термин | Определение |
| ------ | ----------- |
|        |

### B. Ссылки на Документацию

### C. Контактная Информация

- Product Owner: []
- Tech Lead: [@savanswaty]

---

**История изменений:**

| Версия | Дата       | Автор        | Изменения                                                          |
| ------ | ---------- | ------------ | ------------------------------------------------------------------ |
| 1.0    | 2026-03-21 | AvanPay Team | Initial version                                                    |
| 2.0    | 2026-03-26 | AvanPay Team | Модульный монолит, Feature Flags, TMA, Design System, Unified Auth |
| 2.1    | 2026-03-27 | AvanPay Team | Event-Driven Architecture, Event Catalog, Real-Time решения        |

---

_Этот документ будет дорабатываться по мере развития проекта._
