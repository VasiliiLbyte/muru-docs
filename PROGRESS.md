# MURU — Прогресс и память проекта

Живой рабочий журнал. Обновляется в конце сессий. Версионируется git.
Последнее обновление: 2026-07-02 (сессия 2)

## Архитектура (3 компонента)
- **Telegram Mini App** — murushop.online (@murushop_bot), React+Vite / Express+TS+PostgreSQL, Beget VPS, PM2/nginx. Прод.
- **muru-storefront** — Next.js 16 App Router, headless-витрина, заменяет muru.ru (Bitrix). В активной разработке.
- **muru.ru** (Bitrix, Reg.ru, Aspro Premier) — выводится из эксплуатации.

Цель: единый бэкенд на оба канала (web + telegram). Strangler-паттерн — строим/валидируем единый бэкенд в muru-backend-local, запускаем витрину на нём, потом мигрируем Mini App.

## Канонический dev
- **muru-backend-local**, ветка `storefront-integration`, порт 4000 — канонический суперсет бэкенда.
- **MURU_miniAPP/backend** — прод-монорепо, может получать прямые хотфиксы. Форвард-портить в local немедленно.

## Сделано
- Миграция `014_web_identity.sql` (применена, идемпотентна): orders/payments → `telegram_user_id` nullable; `channel TEXT NOT NULL DEFAULT 'telegram'`, CHECK IN ('telegram','web').
- Удалён небезопасный `POST /orders/create` (принимал клиентские цены) — в muru-backend-local.
- **Шаг 3 (веб-канал оплаты, гостевой) — ВЫПОЛНЕН 2026-07-01**, `tsc --noEmit` чисто:
  - 3a: `telegramUserId` nullable в 4 типах, `OrderChannel`, `channel` через snapshot→order, фикс `Number(null)===0`, guard'ы промо, null-gate уведомления.
  - 3b: `POST /payments/web/create` без requireAuth, rateLimitByIp(10/min), Zod отклоняет promoCode, DRY `createPayment`, DEV_ORIGINS localhost:3000.
  - 3c: `getWebPaymentStatus` (`AND channel='web'`), `GET /payments/web/:paymentId/status`, rateLimitByIp(30/min).
- **Шаг 4 (веб-чекаут: оплата ЮКасса + СДЭК, гостевой) — ВЫПОЛНЕН по фронту 2026-07-01**, `tsc --noEmit` чисто на обоих репо:
  - Бэкенд уже был готов (проверено чтением всей цепочки оплаты): `computeTrustedPricing` channel-agnostic — при `delivery` пересчитывает СДЭК на сервере (`calculateTariff`, валидация тарифа против `env.cdek.*`, режет 0); `webSnapshotSchema` принимает `delivery`+cdek-поля (`.strict`); промо для гостей запрещены; `getWebPaymentStatus` есть. Изменений по бэку в этой сессии НЕ потребовалось.
  - 4a-be (ранее): публичный `POST /cdek/web/calculate` (IP-рейтлимит), общий `calculateCdekPrice()`.
  - 4b-be (ранее): email по цепочке snapshot→receipt (customer.email в чеке 54-ФЗ), НЕ пишется в заказ/CRM.
  - Фронт: `PvzMap` (Leaflet+OSM, divIcon без png, `next/dynamic ssr:false`), хелпер `cityNameForAddressSuggest`; CDEK-секция в `checkout-view.tsx` (город debounce → дверь/ПВЗ → карта/список → расчёт по смене города/корзины); submit шлёт `deliveryMode:'delivery'`+cdek-коды; `/checkout/return` (поллинг 3с/40 попыток, очистка корзины); мемоизация `visibleItems`.
  - Уборка: удалены заглушка `createOrder`/`OrderDraftSchema`/`CheckoutResponseSchema` и MSW-мок `*/orders`.

- **Шаг 4c-be (раздельные YooKassa Shop ID) — ВЫПОЛНЕН 2026-07-02**, tsc чисто, vitest yookassa 32/32:
  - `client.ts`: `channel: OrderChannel` — обязательный параметр `ykFetch`/`getYkPayment`; `credsFor(channel)`: web → `YOOKASSA_WEB_SHOP_ID/SECRET_KEY`, в production без web-кредов throw, в dev — fallback на основные с warn.
  - `fulfillPaidPayment`: лёгкий pre-SELECT channel по yookassa_payment_id ДО вызова YK API (заодно не дёргаем API для неизвестных id). `payment-expiry`: channel в SELECT и в cancel. `getPaymentStatusForUser`: channel из БД. `fulfillPaidIntent`/webhook/receipt — без изменений.
  - env: `YOOKASSA_WEB_SHOP_ID/SECRET_KEY` (optional, prod их пока не требует). Ops: у второго магазина настроить webhook на тот же эндпоинт.
- **in_stock-проверка в `computeTrustedPricing` — ВЫПОЛНЕНА 2026-07-02** (`db.in_stock < quantity` → PaymentPricingError, +2 теста). Пункт бэклога закрыт (это пре-чек, резервирования нет — race при параллельной оплате остаётся).
- **Форвард-порт удаления `POST /orders/create` в MURU_miniAPP — ВЫПОЛНЕН 2026-07-02**, tsc чист (be+fe). Удалены: роут+`createOrderHandler` (бэк), мёртвые `createOrder` в api.ts + `submitOrder` в CartContext (фронт; вызовов не было — проверено). `draftPayloadSchema` и сервисный `createOrder` НЕ тронуты (нужны draft/save и fulfill). **Деплой за Василием: бэк + пересборка фронта одним заходом.**
- **Конфиг для локального e2e добавлен 2026-07-02**: backend/.env — `CDEK_ENV=production` + прод-креды CDEK с VPS, DADATA_API_KEY, `YOOKASSA_WEB_RETURN_URL=http://localhost:3000/checkout/return`; storefront/.env.local — `NEXT_PUBLIC_API_BASE=http://localhost:4000/api` (без него дев-витрина сидит на MSW-моках, а не на бэке!).

## Следующее
- **Миграция гидрации корзины на реальный бэкенд (высокий приоритет, блокер cutover).** `src/lib/cart/hydrate.ts` → `getProductBySku()` → `GET /products/by-sku/:sku` — эндпоинт существует только в MSW-фикстурах, на бэке его нет (есть `/api/catalog/products/:sku`). В dev работает только благодаря browser-worker MSW. **В проде (MSW отсутствует) basket-view, mini-cart, favorite-view, checkout-view не смогут догрузить товары по SKU — корзина и чекаут не заработают.** Нужно: перевести `getProductBySku` на `fetchCatalogProductBySku` (уже есть в `catalog-backend.ts`, используется `getProduct`/`getProducts`), либо завести на бэке alias-роут, если у `fetchCatalogProductBySku` семантика slug≠sku разойдётся — проверить перед портированием.
- Полный e2e web-чекаута уже пройден руками Василием: город → ПВЗ/дверь → расчёт → оплата тестовой картой 5555 5555 5555 4477 → `/checkout/return` → заказ в БД, корзина очищена. Повторно гонять не нужно, если не менялась оплата/CDEK-цепочка.

## Блокеры
- Гидрация корзины по SKU висит на mock-only эндпоинте (см. Следующее) — блокирует прод-cutover, не блокирует текущий dev.

## Бэклог (не терять)
- Деплой на VPS (за Василием): форвард-порт orders/create (бэк+фронт Mini App), миграция 014 при переезде.
- Картинки плиток подкатегорий (пустые серые боксы) — фаза CMS; category-grid.tsx деградирует мягко.
- Схлопнуть две папки бэкенда в один git-репо с ветками (убрать merge-боль на cutover).
- `src/lib/content/collections.ts`: `productSlugs` у всех коллекций сейчас `[]` (раньше брались из mock-товаров, разорвано при выносе в статичный модуль) — заполнить реальными SKU после согласования наполнения лендингов с заказчиком.
- CDEK debug-урок для памяти: ранний симптом «расчёт цены не приходит» (сессия 2026-07-02) свёлся к пустым CDEK-кредам в `.env` на момент теста — сам расчётный код был исправен с самого начала. Перед глубоким дебагом смотреть `.env` в первую очередь.

## Крупное расширение скоупа
Заказчик прислал рецензию ТЗ + Арина 6 пунктов маркетинга (30.06.2026). Это **отдельный оплачиваемый этап, НЕ входит в 470k v1**. Детали и триаж — в `SPEC.md` → блок v0.2.

## Лог сессий
- **2026-07-01**: шаг 3 (3a/3b/3c) выполнен и проверен. Заведены muru-docs, PROGRESS.md, SPEC.md. Получены рецензия заказчика и 6 пунктов Арины → зафиксированы в SPEC v0.2 как pending.
- **2026-07-01 (сессия 2)**: шаг 4 закрыт по фронту. Бэкенд оказался уже полностью CDEK-ready — изменений не потребовалось. Фронт: Leaflet PvzMap, CDEK в `checkout-view`, `/checkout/return`, уборка заглушки. Оба репо tsc-чисты. Дальше — 4c-be (раздельные Shop ID) в новом чате.
- **2026-07-02**: 4c-be выполнен (channel — обязательный параметр YK-клиента, pre-SELECT в fulfill, 32/32 теста). Бонусом: in_stock-чек в pricing (+2 теста), форвард-порт удаления orders/create в прод-репо (бэк+фронт, tsc чист, деплой за Василием). Разобраны env-проблемы e2e: пустые CDEK_*/DADATA_* в backend/.env, отсутствовавшие YOOKASSA_WEB_RETURN_URL и NEXT_PUBLIC_API_BASE (витрина сидела на MSW). Ручной прогон: города и ПВЗ работают, расчёт стоимости доставки — НЕТ → блокер, дебаг в новом чате.
- **2026-07-02 (сессия 2)**: CDEK-блокер закрыт — код был рабочий с самого начала (curl подтвердил расчёт: door 615₽, pvz 420₽), симптом был из-за пустых кредов на момент прошлого теста. Василий вручную прогнал полный e2e (город→ПВЗ/дверь→расчёт→оплата 5555…4477→`/checkout/return`) — успешно. По пути найдена и закрыта новая проблема: главная (`/`) отдавала 500, т.к. `NEXT_PUBLIC_API_BASE` отключил MSW целиком, а контентные эндпоинты (`/collections`, `/lookbooks`, `/pages`, `/categories`, `/products*`) существуют только в фикстурах — на бэке их нет. Фикс в 2 шага: (1) гибридный `shouldServerMock()` в `client.ts` — контентные mock-only префиксы резолвятся из фикстур даже при заданном `API_BASE`, plus passthrough-хендлеры для `*/catalog/products*` в MSW, чтобы wildcard `*/products` их не перехватывал; (2) контент (`collections`/`lookbooks`/`pages`) вынесен из `src/mocks/fixtures` в `src/lib/content/` как статичный модуль без HTTP — `getCollections/getLookbooks/getPage` в `endpoints.ts` теперь возвращают его напрямую, `mocks/fixtures/*` стали реэкспортами для обратной совместимости с MSW-хендлерами/резолвером. Оба репо tsc-чисты, все страницы (`/`, `/catalog`, `/basket`, `/lookbooks/*`, `/landings/*`, `/company/*`) → 200. Важный найденный prod-блокер (не в этой сессии): `hydrate.ts` (корзина/мини-корзина/избранное/чекаут) всё ещё ходит на mock-only `/products/by-sku/:sku` — работает в dev только благодаря browser MSW; в проде сломается. Следующий шаг — в новом чате.
