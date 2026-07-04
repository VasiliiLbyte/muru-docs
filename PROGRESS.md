# MURU — Прогресс и память проекта

Живой рабочий журнал. Обновляется в конце сессий. Версионируется git.
Последнее обновление: 2026-07-04 (web catalog F/G/H + channel=web live на staging)

## Архитектура (3 компонента)
- **Telegram Mini App** — murushop.online (@murushop_bot), React+Vite / Express+TS+PostgreSQL, Beget VPS, PM2/nginx. Прод.
- **muru-storefront** — Next.js 16 App Router, headless-витрина, заменяет muru.ru (Bitrix). В активной разработке.
- **muru.ru** (Bitrix, Reg.ru, Aspro Premier) — выводится из эксплуатации.

Цель: единый бэкенд на оба канала (web + telegram). Strangler-паттерн — строим/валидируем единый бэкенд в muru-backend-local, запускаем витрину на нём, потом мигрируем Mini App.

## Канонический dev
- **muru-backend-local**, ветка `storefront-integration`, порт 4000 — канонический суперсет бэкенда.
- **MURU_miniAPP/backend** — прод-монорепо, может получать прямые хотфиксы. Форвард-портить в local немедленно.

## Сделано
- Миграция `014_web_identity.sql` (идемпотентна): orders/payments → `telegram_user_id` nullable; `channel TEXT NOT NULL DEFAULT 'telegram'`, CHECK IN ('telegram','web'). **Local DB + VPS prod-БД** (2026-07-02/03, коммит `0fcdf41`, `psql` на VPS — 4×ALTER + DO, без ошибок).
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
- **Форвард-порт удаления `POST /orders/create` в MURU_miniAPP — ВЫПОЛНЕН и ЗАДЕПЛОЕН 2026-07-02** (`9e8e8e0`, VPS `deploy.sh`, подтверждено Василием).
- **Конфиг для локального e2e добавлен 2026-07-02**: backend/.env — `CDEK_ENV=production` + прод-креды CDEK с VPS, DADATA_API_KEY, `YOOKASSA_WEB_RETURN_URL=http://localhost:3000/checkout/return`; storefront/.env.local — `NEXT_PUBLIC_API_BASE=http://localhost:4000/api` (без него дев-витрина сидит на MSW-моках, а не на бэке!).
- **Гидрация корзины на каталог-бэкенд — ВЫПОЛНЕНА 2026-07-02** (промпт `2026-07-02-01`, `muru-storefront`), verify: diff 2 файла, `tsc`+`build` чисто, curl `GET /api/catalog/products/:sku` → 200:
  - `catalog-backend.ts`: `CATALOG_API_BASE` fallback на `NEXT_PUBLIC_API_BASE` (в prod хватит одной env); `isCatalogBackendEnabled()` → `Boolean(CATALOG_API_BASE)`; `fetchCatalogProductBySku` — `trim().toUpperCase()` перед запросом.
  - `getProductBySku` → `fetchCatalogProductBySku` при включённом каталог-бэке (ветка уже была в `endpoints.ts`; `hydrate.ts` без изменений).
  - `.env.example`: добавлен `NEXT_PUBLIC_CATALOG_API_BASE` с комментарием про fallback.
  - Smoke без MSW (Василий): `/basket/`, mini-cart, `/checkout/` — гидрация OK (названия/цены).
- **`AddToCartButton` → корзина — ВЫПОЛНЕНО 2026-07-02** (промпт `2026-07-02-02`, `muru-storefront`), verify: 1 файл, `tsc` чисто, ручной тест Василия — каталог → badge, `/basket/`, mini-cart OK:
  - `add-to-cart-button.tsx`: `useCartStore.addItem(sku)` после `preventDefault/stopPropagation`; заглушка убрана.

- **Forward-port миграции 014 в MURU_miniAPP — ВЫПОЛНЕН и ПРИМЕНЁН на VPS 2026-07-03** (промпт `2026-07-03-01`, коммит `0fcdf41`): `backend/src/db/migrations/014_web_identity.sql` идентичен `muru-backend-local`; README обновлён; VPS `git pull` + `psql -f` — успешно.
- **Storefront: cutover корзины на GitHub 2026-07-03** — `muru-storefront` / `main` коммит `3a03a61` (`catalog-backend.ts`, `.env.example`, `add-to-cart-button.tsx`).
- **Web checkout API на VPS — ВЫПОЛНЕН и ЗАДЕПЛОЕН 2026-07-03** (`DEP-003`, промпт `2026-07-03-02-fp`, коммит `52b23fd`): forward-port web API в `MURU_miniAPP`; VPS `git pull` + `deploy.sh`; curl prod: `/api/cdek/web/calculate` и `/api/payments/web/create` → 400 (роуты живы, не 404); smoke mini app Василия — OK.

- **Staging `web.murushop.ru` — LIVE 2026-07-03:** deploy `8f5bb15`→`6a5564a`, PM2+nginx+certbot, `curl /catalog/` → 200; fix `force-dynamic` (`2026-07-03-04`). Mini app smoke OK.
- **Каталог на staging: ProductGrid в топ-категории — ВЫПОЛНЕН 2026-07-03** (промпт `2026-07-03-05`, коммит `cf2084d`): `hasSubcategories` → `variant=listing` при плоском дереве бэка; verify curl/HTML: `/catalog/флористика/` — карточки MU0069…, без «Скоро здесь»; PDP 200. Задеплоено на VPS (Василий).
- **E2e staging `web.murushop.ru` — ВЫПОЛНЕН 2026-07-03** (ручной прогон Василия): каталог → корзина → `/checkout/` → CDEK (расчёт OK) → ЮKassa (тестовая карта) → `/checkout/return/` «Заказ оплачен». Env: `YOOKASSA_WEB_RETURN_URL`, `YOOKASSA_WEB_SHOP_ID`, `YOOKASSA_WEB_SECRET_KEY` в `/var/www/muru/backend/.env` (SSH; файловый менеджер Beget не сохранял — права панели).
- **Web catalog F/G/H — бэкенд + sync на VPS 2026-07-04** (`2026-07-03-06-be/fp`, hotfix subcategory columns, sync ~260). Mini app OK.
- **Storefront channel=web — LIVE на staging 2026-07-04** (`2026-07-03-06-fe`, `b419a3f` на VPS): cross-категории G/H на `web.murushop.ru` — verify Василия OK.

## Следующее
- **Контент/UX staging (по приоритету бизнеса):** картинки плиток категорий, URL PDP (`/catalog/…/…/SKU/`), `collections.ts`.
- **Go-live `muru.ru`:** DNS, prod ЮKassa web, 301 с Bitrix — когда заказчик готов.

## Блокеры
- Нет активных блокеров. Pending deploy: DEP-001..005 закрыты.

## Ожидает деплоя (Pending deploy)

Живой статус: что **закоммичено и verified**, но ещё **не на VPS** (или не на prod-БД). Детали команд — [`DEPLOY.md`](DEPLOY.md). Протокол синхронизации репо — [`FORWARD_PORT.md`](FORWARD_PORT.md).

| ID | Изменение | Репо / ветка | Код | VPS | Действие | Заметки |
|---|---|---|---|---|---|---|
| DEP-001 | Удаление небезопасного `POST /orders/create` (бэк + фронт mini app) | `MURU_miniAPP` / `master` | verified | **deployed** (Василий, 2026-07-02) | — | Коммит `9e8e8e0` на GitHub `master`; `git pull` + `deploy.sh` на VPS |
| DEP-002 | Миграция `014_web_identity.sql` (web-канал, nullable telegram_user_id) | `MURU_miniAPP` / `master` | verified | **deployed** (Василий, 2026-07-03) | — | Коммит `0fcdf41`; VPS `psql -f` — 4×ALTER + DO OK |
| DEP-003 | Web checkout API (`/payments/web/*`, `/cdek/web/calculate`, channel-aware YK) | `MURU_miniAPP` / `master` | verified | **deployed** (Василий, 2026-07-03) | — | Коммит `52b23fd`; `deploy.sh`; smoke mini app OK |
| DEP-004 | Web catalog F/G/H + hotfix subcategory | `MURU_miniAPP` / `master` | verified | **deployed** (Василий, 2026-07-04) | — | `c2df422` + hotfix; migration 015; sync ~260 |
| DEP-005 | Storefront `channel=web` cross-listing | `muru-storefront` / `main` | verified | **deployed** (Василий, 2026-07-04) | — | `b419a3f` на `web.murushop.ru` |

**Как обновлять:** оркестратор добавляет строку при verify prod-затрагивающей задачи; после деплоя Василий сообщает → колонка VPS = `deployed`, строка переносится в «Сделано» или помечается ✅.

## Бэклог (не терять)
- Деплой на VPS — `DEP-001`..`003` ✅. Следующий ops: storefront staging на `web.murushop.ru`.
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
- **2026-07-02 (сессия 3)**: Документация процесса: `FORWARD_PORT.md`, `55-executor.mdc` в backend-local и storefront, секция Pending deploy (`DEP-001`..`003`) в PROGRESS. GitHub remotes выровнены. Пуш muru-docs (API_CONTRACT, DEPLOY). Готовность к оркестратору в workspace muru-docs.
- **2026-07-02 (сессия 4)**: Промпт `2026-07-02-01` — гидрация корзины на каталог-бэкенд; `2026-07-02-02` — `AddToCartButton` → `useCartStore`. Verify + ручной smoke Василия: каталог/корзина/чекаут без MSW OK. `API_CONTRACT.md` §2 обновлён.
- **2026-07-03**: Cutover staging `web.murushop.ru` LIVE (`8f5bb15`→`6a5564a`, `force-dynamic`, PM2 env). DEP-001..003 ✅. Каталог 200; товары не видны — UX hub vs flat tree (не sync).
- **2026-07-03 (сессия 2)**: Промпт `2026-07-03-05` — `variant=listing` при flat tree; `cf2084d` на VPS. Verify: флористика 25 карточек. Следующий шаг — e2e оплаты на staging.
- **2026-07-03 (сессия 3)**: E2e staging закрыт — CDEK + ЮKassa web env через SSH; «Заказ оплачен» на `web.murushop.ru`. Beget file manager — не использовать для `/var/www/.../.env`.
- **2026-07-04**: Web catalog F/G/H (`2026-07-03-06-be/fp/fe`). Hotfix `products.subcategory` на VPS. Sync 260. Mini app + web cross-listing OK. DEP-004/005 ✅. Фаза staging v1 закрыта.
