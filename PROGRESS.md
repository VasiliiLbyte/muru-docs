# MURU — Прогресс и память проекта

Живой рабочий журнал. Обновляется в конце сессий. Версионируется git.
Последнее обновление: 2026-07-14 (invoice telegram.me hotfix merged, DEP-017 pending)

## Архитектура (3 компонента)
- **Telegram Mini App** — murushop.online (@murushop_bot), React+Vite / Express+TS+PostgreSQL, Beget VPS, PM2/nginx. Прод.
- **muru-storefront** — Next.js 16 App Router, headless-витрина, заменяет muru.ru (Bitrix). В активной разработке.
- **muru.ru** (Bitrix, Reg.ru, Aspro Premier) — выводится из эксплуатации.

Цель: единый бэкенд на оба канала (web + telegram). Strangler-паттерн — строим/валидируем единый бэкенд в muru-backend-local, запускаем витрину на нём, потом мигрируем Mini App.

## Канонический репозиторий (после cutover 2026-07-06)
- **muru-backend-local**, ветка `master` — единственный канон. Прод `/var/www/muru` деплоится напрямую отсюда (`origin` на VPS переключён на `github.com/VasiliiLbyte/muru-backend-local`).
- **MURU_miniAPP** — заморожен (см. лог U4-post). Новые изменения — только в каноне, форвард-порт больше не нужен (`FORWARD_PORT.md` deprecated).
- Локальная разработка — по-прежнему в `muru-backend-local`, порт 4000; для рискованных изменений — staging на VPS (`api-staging.murushop.ru`, порт 4001, см. `DEPLOY.md`).

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
- **Аудит 2026-07-04** (read-only): ядро v1 подтверждено (miniAPP vitest 221/221); CRM v0.1 ≈ 15–20%; закрыто: обратный FP-gap CORS (`84cab5e` в local), rate limits, `schema.sql` parity, prod-only `node_modules` на каноне, билд-зависимость `catalog-menu` от живого API; trust proxy уже был (ошибка аудита).
- **Hardening DEP-006** (`2c97f0e`, VPS 2026-07-04): `restock-notify` 5/min IP, `pickup-points` 30/min IP, `schema.sql` web-identity; `deploy.sh`, без psql; verify: 6-й POST restock → 429.
- **Storefront build resilience DEP-007** (`05f08bf`, VPS 2026-07-04): `catalog-menu` try/catch + `.env.production*` в gitignore; build 28/28; `/catalog/` + `/legal/offer/` 200.

- **Унификация бэкендов — фаза стартована 2026-07-06.** Решение: до CRM-фазы схлопнуть `MURU_miniAPP` и `muru-backend-local` в единый репозиторий (канон — `muru-backend-local`). План: U1 аудит дрейфа → U2 выравнивание+merge в master → U3 staging на VPS → U4 cutover прода (смена git remote в `/var/www/muru`).
  - **U1+U2 ВЫПОЛНЕНЫ и ВЕРИФИЦИРОВАНЫ 2026-07-06** (промпт `2026-07-05-01`, репо `muru-backend-local`). Исполнитель совместил read-only аудит с фиксом и merge (процессное отклонение от промпта — ожидался сначала отчёт-таблица, но по факту работа корректна, приняли по verify):
    - Pre-U2 фикс (`6c8ba48`): `frontend/src/cart/CartContext.tsx` (удалены `createOrder`/`submitOrder`/`LEGAL_VERSION`), `frontend/src/lib/api.ts` (удалён `createOrder`) — форвард-порт miniAPP `9e8e8e0`; README выровнен (миграции 014/015, payment-flow).
    - U2 merge: `storefront-integration → master` fast-forward `56a32c2 → 6c8ba48`, 6 коммитов (`aab9902` web checkout API, `3cc8c89` executor rule, `c5aeda5` web catalog F/G/H, `1103af0` catalog hotfix mirror, `84cab5e` CORS+rate limits, `6c8ba48` frontend cleanup+README).
    - Verify исполнителя: `tsc --noEmit` чисто, vitest 221/221, `git status` чисто, миграции 014/015 применены (local DB), smoke `:4000` (health/catalog/cdek/pickup-points/payments 200|400-ожидаемо).
    - **Верификация оркестратора (независимая, выборочная):** `frontend/src` local-master vs miniAPP-master — идентичны (только `.DS_Store`); `backend/src` — только 2 известных отличия с аудита 2026-07-04 (`index.ts` DEV_ORIGINS dev-only, `types/order.ts` пустая строка), новых нет; вся инфраструктура (`deploy.sh`, `deploy/`, `ecosystem.config.js`, `backend/package.json`, `frontend/package.json`, `.env.example`, `vite.config.ts`, `tsconfig*`, `index.html`, `README.md`) — идентична байт-в-байт; FF-merge подтверждён (`merge-base --is-ancestor 56a32c2 6c8ba48` = true). `git cherry` изначально показал 5 коммитов с несовпадающим patch-id (`9e8e8e0`,`0fcdf41`,`52b23fd`,`c2df422`,`2c97f0e`) как потенциальный red flag «хотфиксы прода не в каноне» — перепроверено напрямую по содержимому файлов, реальных расхождений нет (ложное срабатывание: форвард-порты делались вручную, а не `cherry-pick`, отсюда разный patch-id при идентичном результате).
    - **Итог:** `muru-backend-local/master` — унифицированный канон, дрейфа не осталось. `git push origin master` **не выполнен** — следующий шаг.
  - **U3-a ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-06** (промпт `2026-07-06-01`, коммит `6e7ab0f`): добавлены `ecosystem.staging.config.js` (PM2-процесс `muru-backend-staging`, отдельный от прод `muru-backend`), `deploy-staging.sh` (backend-only, с prod-safety guard на имя процесса), `.env.staging.example`, `deploy/nginx-api-staging.conf` (без `/yookassa-webhook` — вебхук остаётся на проде). Существующие прод-файлы не тронуты, verify оркестратора — все 4 файла перечитаны, содержимое соответствует промпту.
  - **U3-b/U3-c (staging на VPS) ВЫПОЛНЕНЫ и ВЕРИФИЦИРОВАНЫ 2026-07-06.** Staging живой: `https://api-staging.murushop.ru` (PM2 `muru-backend-staging`, порт 4001, БД `muru_staging` — снапшот прод-БД `muru_db`, 260 products). Полный смоук (внутренний `:4001` + внешний HTTPS, verified оркестратором независимо): `/api/health` 200, `/api/catalog/tree` 200, `/api/catalog/products?channel=web` 200 с web-полями, rate-limit `pickup-points` 30/мин → 429 на 30-м запросе (DEP-006 жив), `/api/payments/web/create` без body → 400, `/api/profile` без токена → 401, миграции 014/015 (`channel`, `web_subcategory_name`) унаследованы из дампа. Прод (`muru-backend`) все время инцидента — `103` рестарта без единого изменения, health 200 непрерывно.
    - **Известный некритичный gap:** `/api/cdek/health`, `/api/cdek/pickup-points` → 500 на staging (пустые тестовые `CDEK_CLIENT_ID/SECRET` в `.env.staging` — код не делает startup-guard для CDEK, падает в рантайме при реальном запросе к CDEK API; не регрессия от мержа, чисто конфигурационный пробел staging-окружения, rate-limit при этом отработал корректно независимо от 500).
    - **Инциденты по пути (важно для будущих staging-сетапов и памяти проекта):**
      1. GitHub-репозиторий `muru-backend-local` имеет **дефолтную ветку `main`** (устаревшая, `aab9902`), а не `master` (канон, `6e7ab0f`+) — `git clone` без `-b master` берёт `main` и молча даёт старый код без свежих файлов. **Рекомендация (не блокирует U4, но стоит сделать):** сменить дефолтную ветку на GitHub на `master` через Settings → Branches, во избежание повтора в будущем.
      2. `pg_dump | psql` в свежесозданную `muru_staging` падал с `permission denied for schema public` — на PG15+ владелец БД (`-O muru_user` в `createdb`) не наследует владение автосозданной схемой `public`. Фикс: `ALTER SCHEMA public OWNER TO muru_user;` перед restore.
      3. `TELEGRAM_BOT_TOKEN` в staging `.env` оказался непустым (нарушение жёсткого инварианта) — проверено через `pm2 logs muru-backend | grep 409/conflict` и сравнение хешей токенов, прод-бот не пострадал (в логах только старый несвязанный `fetch failed` до старта staging). Очищено, процесс перезапущен.
      4. `env.ts` при `NODE_ENV=production` жёстко требует непустые `YOOKASSA_SHOP_ID/SECRET_KEY/RETURN_URL` (`process.exit(1)` без видимого лога из-за гонки flush/exit) — `.env.staging.example` изначально оставлял их пустыми (ошибка оркестратора в промпте U3-a). Фикс: плейсхолдер-значения для staging.
      5. `deploy/nginx-api-staging.conf` ссылался на ещё не выпущенный сертификат → `nginx -t`/`reload` ломались для **всего** nginx (включая прод-конфиги в том же `sites-enabled/`) до получения сертификата. Фикс: временный HTTP-only блок → `certbot --nginx` → автодопись SSL-директив. Прод не переставал отвечать (`nginx` не перезапускался, только неудачный `reload`, старый воркер продолжал работать).
  - **U4 (cutover прода) ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-06.** `/var/www/muru`: `git remote set-url origin https://github.com/VasiliiLbyte/muru-backend-local.git` → `git reset --hard origin/master` (`6e7ab0f`) → `bash deploy.sh` (backend `tsc` + frontend `vite build` чисто, `pm2 reload muru-backend` успешно, restart-counter `103→104`, ровно один reload). Смоук (verified независимо оркестратором): `murushop.ru/api/health` 200, `/` 200, `/catalog` 200; `murushop.online/yookassa-webhook` → 301 на `murushop.ru/yookassa-webhook` (ожидаемо, легаси-домен, см. `deploy/nginx-murushop.online.conf`); `murushop.ru/yookassa-webhook` → 404 при прямом curl — **не баг**, это `yookassaIpGuard` (`YOOKASSA_VERIFY_IP=true` подтверждён на проде) сознательно отдаёт пустой 404 не-ЮKassa IP. Ручной смоук Василия: Mini App из Telegram доходит до экрана оплаты ЮKassa, `web.murushop.ru` — каталог и корзина работают.
    - **Точки отката (на случай регрессии в будущем):** старый `origin` = `https://github.com/VasiliiLbyte/MURU_miniAPP.git`, SHA прод-HEAD до cutover = `2c97f0ecd40fd3f66d0fb09463cbd8903d5c1df9`. Откат: `git remote set-url origin <старый URL>` + `git reset --hard 2c97f0e` + `bash deploy.sh`.
    - **Инциденты U4:** нет (единственная находка — ложная тревога с `yookassa-webhook` 404, разобрана и объяснена выше, не потребовала изменений кода).
  - **U4-post (заморозка `MURU_miniAPP`) ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-06** (промпт `2026-07-06-02`, коммит `7877be1` на `MURU_miniAPP/master`, push OK): баннер «⚠️ Репозиторий заморожен (2026-07-06). Канон и прод-деплой: `VasiliiLbyte/muru-backend-local`» первыми строками README, `git show 7877be1 --stat` → только README.md +2 строки (verified оркестратором). GitHub archive — опционально, за Василием. Финальное обновление `DEPLOY.md` (карта окружений, §2a staging, §7 webhook), `FORWARD_PORT.md` (deprecated), `ORCHESTRATOR.md` (карта репо, `MURU_miniAPP` заморожен) — выполнено.
  - **ИТОГ ФАЗЫ: унификация бэкендов U1-U4 полностью завершена и верифицирована за одну сессию 2026-07-06.** Единый канон `muru-backend-local/master` = прод (`/var/www/muru`, деплой напрямую). `MURU_miniAPP` заморожен. Форвард-порт больше не нужен. Точка отката на прод: `origin` = `MURU_miniAPP.git`, SHA `2c97f0e`.

## Следующее
- **DEP-017 deploy на VPS:** `master` @ `a98222e` (`git pull` + `deploy.sh`) — cutover prep + invoice `telegram.me` hotfix. Миграций нет.
- **Smoke после deploy:** mini app → checkout → «Перейти к оплате» → invoice открывается.
- **Cutover окно:** после smoke — `CUTOVER.md`.

## Cutover prep (код, merged 2026-07-14)

**master @ `a98222e`** (FF `1e439bf→da280b2→a98222e`): `[env] loaded`, `MINIAPP_MAINTENANCE`, sync 423 crm, invoice URL accepts `telegram.me/$`. Ранбук — `CUTOVER.md`. **Не на VPS** (DEP-017).

## Фаза: Админка A (фундамент — auth, роли, каркас)

**Старт:** 2026-07-06. **Цель:** фундамент CRM v0.1 — модуль ролей/доступа (ТЗ §3) + каркас веб-панели, на который лягут следующие фазы (товары, контент).

### Архитектурные решения (приняты, не пересматривать)
1. **Размещение:** новая папка `admin/` в корне канона (рядом с `backend/`, `frontend/`). Vite + React + TS strict, тот же стек что frontend mini app. Без SSR.
2. **Раздача:** SPA → статика, nginx отдаёт на `https://murushop.ru/admin/` (тот же origin что API → нет CORS/cross-domain cookie). Vite `base: '/admin/'`, роутер с basename.
3. **Auth:** email+пароль, bcrypt cost 12. Отдельный `ADMIN_JWT_SECRET` (НЕ `JWT_SECRET` TG-канала), payload `{adminId, role}`, TTL 12h. Доставка — httpOnly cookie `admin_token`, Secure, SameSite=Strict. Без refresh-токенов в v1.
4. **Роли:** enum `('owner','manager')`. owner = всё; manager = всё кроме управления пользователями и настроек. Кладовщик — позже (при складе).
5. **Изоляция:** TG-admin роуты `/api/admin/*` НЕ трогаются (mini app-админка работает параллельно). Новые CRM-роуты — префикс `/api/crm/*` + auth-роуты `/api/admin-auth/*`, middleware `requireCrmAuth`.
6. **Workflow (обкатываем):** вся работа в ветке `admin-phase-a`. Staging деплоится из неё. Миграция 016 — сначала на `muru_staging`, смоук, только после verify — merge в `master` + прод. Прод-деплой = **DEP-009**.

### Инварианты фазы
- Миграция 016 НИКОГДА не применяется к прод-БД раньше staging-обкатки.
- Пароли/секреты не в git; `.env.example` — только имена переменных.
- TG-admin (`/api/admin/*`) не регрессирует — любой промпт, задевший `admin.ts`/`admin.middleware.ts`, стоп.
- Каждый executor-отчёт верифицируется чтением фактического кода до перехода к следующему шагу.

### Состав (порядок обязателен)
- **A1** — backend: миграция `016_admin_users.sql` + auth-ядро (сервис, роуты `/api/admin-auth`, `requireCrmAuth`, `ADMIN_JWT_SECRET` в env, vitest). Модель сильнее Auto.
- **A2** — seed-скрипт первого владельца (`seed-admin.ts`, env-переменные процесса, идемпотентно).
- **A3** — SPA-каркас `admin/` (login, защищённый layout, дашборд-плейсхолдер, API-клиент).
- **A4** — интеграция деплоя (`deploy.sh`, `deploy-staging.sh`, nginx `location /admin/`).
- **A5** — раноблок staging (миграция 016 на `muru_staging`, seed, смоук) → STOP-гейт.
- **A6** — прод (DEP-009): merge → master, миграция 016 на прод-БД, deploy, nginx, seed владельца.

### Статус
- **A1 ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-06** (executor Codex 5.3, ветка `admin-phase-a`, коммит `8aa5f14`): миграция `016_admin_users.sql` (идемпотентна) + синхронный блок в `schema.sql`; auth-ядро CRM полностью отдельно от TG-канала (`admin-auth.service.ts` — тайминг-безопасный `login()` через `DUMMY_HASH` для несуществующих/неактивных админов, `signAdminJwt`/`verifyAdminJwt` со строгой проверкой формы payload; `require-crm-auth.middleware.ts`; `admin-auth.controller.ts` — cookie `admin_token` httpOnly/secure(prod)/sameSite=strict/path=/api/maxAge=12h; роуты `/api/admin-auth/{login,logout,me}`, `login` под `rateLimitByIp 5/min`). `env.ts`: `ADMIN_JWT_SECRET` + production-guard (>=32 симв., `process.exit(1)` по образцу YooKassa-guard). `index.ts`: `cookie-parser` после `express.json()`+webhook-mount, до прочих `/api/*`. Зависимости: `bcryptjs@^3.0.3`, `cookie-parser@^1.4.7` (+типы) — **нативный `bcrypt` осознанно не использован** (pure-JS, без node-gyp на VPS).
  - **Верификация оркестратора (независимая, полная):** `git diff --name-only master admin-phase-a` — TG-admin (`routes/admin/**`, `admin.middleware.ts`, `auth.middleware.ts`, `jwt.service.ts`) НЕ затронут; прочитан весь новый/изменённый код построчно (сервис, middleware, контроллер, роуты, `index.ts`/`env.ts` diff, `.env.example`/`.env.staging.example`); `DUMMY_HASH` — валидный bcrypt-хэш (проверено `bcrypt.compare` без исключения); самостоятельный прогон `npx tsc --noEmit` — чисто, `npm run test` — **43 файла / 229 тестов зелёные** (221 было + 8 новых: 5 admin-auth.service, 3 require-crm-auth.middleware), совпадает с отчётом исполнителя.
  - **Сигнатуры для A3/A5:** роуты `POST /api/admin-auth/login|logout`, `GET /api/admin-auth/me`; JWT `{adminId, role}` TTL 12h; cookie `admin_token`, опции см. выше.
  - Миграция 016 НЕ применялась ни к одной БД (code-only, по инварианту фазы) — применение только на A5 (staging).
- **A2 ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-06** (ветка `admin-phase-a`, коммит `a1fbf5c`): `backend/src/scripts/seed-admin.ts` — идемпотентный upsert владельца (`ON CONFLICT (email) DO UPDATE`), `ADMIN_SEED_EMAIL`/`ADMIN_SEED_PASSWORD` только из `process.env` (не из `.env`-файла), bcryptjs cost 12, пароль/хэш не логируются, `pool.end()` перед выходом (в т.ч. в error-ветке). `package.json`: добавлен `"seed:admin": "tsx src/scripts/seed-admin.ts"`, прочие scripts не тронуты. Верификация оркестратора: `git diff --name-only 8aa5f14 admin-phase-a` — только 2 файла; содержимое перечитано целиком; самостоятельный `tsc --noEmit` — чисто. Скрипт против БД не запускался (по инварианту — только на A5). Команда запуска: `ADMIN_SEED_EMAIL=... ADMIN_SEED_PASSWORD=... npm run seed:admin` (из `backend/`).
- **A3 ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-06** (ветка `admin-phase-a`, коммит `48f62ee`): SPA-каркас `admin/` (Vite+React 19+TS, `base:'/admin/'`, dev-proxy `/api`→`:4000` для same-origin cookie без CORS, без `VITE_API_BASE_URL` — архитектурно не нужен). `lib/api.ts` (envelope-клиент, 401→редирект `/admin/login` с исключением `/me`), `lib/auth.ts`, `context/AuthContext.tsx` (гидрация сессии), `components/ProtectedLayout.tsx` (редирект неавторизованных, сайдбар: «Дашборд» кликабелен, остальные 4 пункта — `<span>`, не кликабельны), `pages/LoginPage.tsx`/`DashboardPage.tsx`, `App.tsx` (роуты + wildcard→`/`). Отклонение (обосновано): `ApiError` без TS parameter properties — конфликт с `erasableSyntaxOnly` (TS1294), заменено на явные поля в конструкторе.
  - Верификация оркестратора: `git diff --name-only a1fbf5c admin-phase-a` — только `admin/**`; весь код прочитан (api.ts, auth.ts, AuthContext, ProtectedLayout, App, main.tsx, LoginPage, vite.config.ts, package.json); самостоятельный `npm install`+`npx tsc -b`+`npm run build` — идентичные размеры `dist/` (236.83 kB JS/1.50 kB CSS), 0 ошибок/warning; `.gitignore` подтверждён (`node_modules`/`dist` не в git, `git status` чист после локальной сборки).
- **A4 ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-06** (ветка `admin-phase-a`, коммит `154955d`): `deploy.sh` — шаг `[3/4]` сборка `admin/` (тот же паттерн `rm -rf node_modules && npm install`, что у frontend), pm2-restart сдвинут на `[4/4]`. `deploy-staging.sh` — шаг `[2/3]` сборка `admin/` (только build-check, без раздачи — staging остаётся API-only по домену), guard на имя процесса не тронут. `deploy/nginx-murushop.ru.conf` — новый `location /admin/` (`alias`, не `root`; SPA-fallback на `/admin/index.html`), вставлен между `/yookassa-webhook` и `location /`, остальное не тронуто. **Решение оркестратора по staging:** `nginx-api-staging.conf` НЕ меняется (домен named/scoped как API-only) — `admin/` на staging только собирается, полноценная раздача и визуальный смоук логина — только на проде (A6).
  - Верификация: `git diff --name-only 48f62ee admin-phase-a` — ровно 3 файла; полный diff перечитан построчно; самостоятельный `bash -n` на обоих скриптах — чисто.
- **A5 ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-06** (staging `api-staging.murushop.ru`): миграция 016 применена (`\d admin_users` подтвердил структуру), `deploy-staging.sh` (backend+admin build+pm2) прошёл чисто, seed тестового владельца успешен. **По пути найден и закрыт реальный баг** (не гипотетический — проявился на практике): `npm ci --omit=dev` в `deploy-staging.sh`/`deploy.sh` вычищает devDependencies, включая `tsx`, на котором был завязан `seed:admin` → падал `tsx: not found`. Фикс (коммит `17876da`, ветка `admin-phase-a`): `seed:admin` теперь запускает уже скомпилированный `dist/scripts/seed-admin.js` (компилируется автоматически при обычном `npm run build`, `tsconfig.json` → `include: ["src/**/*"]`) вместо `tsx src/scripts/...ts`. Это было бы воспроизвелось и на проде в A6 — поймано вовремя.
  - Полный смоук после фикса (скрипт-файл, не построчный paste — во избежание склейки команд при вставке, ловили это несколько раз ранее в сессии): 4×невалидный логин → 401, 5-й (валидный) → 200, 6-й подряд → **429** (rate-limit 5/мин жив), `/me` с cookie → 200 `{"email":"owner-staging@example.com","role":"owner"}`, `/me` без cookie → 401, `/logout` → 200, `/me` после logout → 401. **Все 7 пунктов совпали с ожиданием.** Прод (`muru-backend`) — счётчик рестартов не менялся (`104`) весь шаг A5, health 200.
  - Fast-forward подтверждён локально: `master` (`6e7ab0f`) — прямой предок `admin-phase-a` (5 коммитов: `8aa5f14`→`a1fbf5c`→`48f62ee`→`154955d`→`17876da`), merge в A6 будет чистым FF без конфликтов.
- **A6 (прод, DEP-009) ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-06.** `master` (`17876da`) — fast-forward merge `admin-phase-a` (5 коммитов A1→A2→A3→A4→A2-fix), push OK. На `/var/www/muru`: миграция 016 применена на прод-БД (`muru_db`) ДО деплоя кода; `ADMIN_JWT_SECRET` (независимый, `openssl rand -hex 32`) вписан в прод `.env`; `bash deploy.sh` (backend+frontend+admin build, `pm2 reload muru-backend`); `sudo bash deploy/sync-nginx-murushop.sh` (подхватил новый `location /admin/`); seed боевого owner (`admin@muru.ru`). Полный смоук: `login` 200, `/me` 200 `{"email":"admin@muru.ru","role":"owner"}`, `logout` 200; TG-admin `/api/admin/me` 401 (роут жив, не регрессировал); ручной браузерный вход на `https://murushop.ru/admin/` — дашборд, сайдбар (Дашборд активен, 4 пункта приглушены), логаут — все по скрину Василия; Mini App (Telegram) без регрессий.
  - **Инцидент (пойман и объяснён, без потери данных):** счётчик рестартов `muru-backend` скакнул `104→120` (+16 вместо обычного +1 на reload) — реконструкция: первый `deploy.sh` был запущен до того, как `ADMIN_JWT_SECRET` реально попал в `.env` (secret вписывался через GUI файл-менеджер VPS, не через видимый в терминале `nano`) → production-guard из A1 (`process.exit(1)` при коротком/пустом `ADMIN_JWT_SECRET`) вызвал crash-loop → после вписывания секрета и повторного `deploy.sh` процесс стабилизировался. Прод не терял доступности по mini app/API в моменты проверки (PM2 crash-loop на cluster-процессе не даёт полного даунтайма благодаря graceful restart).
  - Точка отката (если понадобится): `git revert`/`reset` на `6e7ab0f` в `master` + `bash deploy.sh` — миграция 016 аддитивна, старый код её игнорирует, откат кода безопасен без отката БД.
- **ИТОГ ФАЗЫ: «Админка A: фундамент» (A1-A6) полностью завершена и верифицирована 2026-07-06.** Фундамент CRM v0.1 готов: `admin_users` (owner/manager), auth-ядро (`/api/admin-auth/*`, httpOnly-cookie, bcrypt, отдельный JWT), SPA-каркас `admin/` на `https://murushop.ru/admin/`, деплой интегрирован. TG-admin (`/api/admin/*`) не регрессировал ни на одном шаге.

## Фаза: Админка B (контент-модуль — CMS, ТЗ §10)

**Старт:** 2026-07-08. **Цель:** страницы, коллекции, лукбуки, баннеры управляются из CRM-админки и питают витрину. Блог и подарочные карты — **вне скоупа**.

**Репозитории:** `muru-backend-local` (ветка `admin-phase-b` от `master`) + `muru-storefront` (`main`). Порядок: backend → admin → storefront. Прод = **DEP-010**.

### Архитектурные решения (приняты, не пересматривать)
1. **Изображения:** локально на VPS — `uploads/` в корне репо (`/var/www/muru/uploads/`), в `.gitignore`, раздача nginx `location /uploads/`, immutable cache. S3 — не сейчас.
2. **Редактор:** TipTap (WYSIWYG) в B3 — body как HTML.
3. **Безопасность HTML:** `sanitize-html` на бэке при записи (whitelist: p, h2-h4, ul/ol/li, strong/em, a[href], img[src|alt], br, blockquote).
4. **API:** CRM CRUD — `/api/crm/content/*` + `requireCrmAuth`; публичное чтение — `GET /api/content/*` (только видимое).
5. **Контракт публичного API** = Zod-схемы витрины (`collection.ts`, `lookbook.ts`, `static-page.ts`, `common.ts` ImageSchema). `productSlugs` = SKU-массив; витрина резолвит каталогом.
6. **Витрина:** контент с `src/lib/content/*` → fetch `/api/content/*` с graceful fallback (образец `catalog-menu`, DEP-007).

### Инварианты фазы
- Миграция `017_content.sql` — только после staging-обкатки (B5), не на прод раньше.
- Не трогать: TG-admin (`/api/admin/*`), payments/yookassa, CDEK, catalog-sync.
- `schema.sql` синхронен с 017; `uploads/` не в git.
- Verify чтением кода на каждом шаге.

### Состав (порядок обязателен)
| Шаг | Что | Модель |
|-----|-----|--------|
| **B1** | Миграция 017 + CRUD + публичное чтение + vitest | Сильнее Auto |
| **B2** | Upload изображений (`POST /api/crm/content/upload`, sharp, multer) | Сильнее Auto |
| **B3** | Admin UI «Контент» (TipTap, табы, upload-компонент) | Сильнее при буксе |
| **B4** | Storefront → API + fallback | Auto, жёсткий промпт |
| **B5** | Staging STOP-гейт | — |
| **B6** | Прод DEP-010 + nginx `/uploads/` | — |

### Статус
- **B1 ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-08** (ветка `admin-phase-b`, коммит `d9218ac`): миграция `017_content.sql` (6 таблиц, идемпотентна) + синхронный блок в `schema.sql`; `content-sanitize.service.ts` (whitelist, script/onerror вырезаются); `content.service.ts` + `content.mapper.ts` + Zod `content.schemas.ts`; CRM `/api/crm/content/*` под `requireCrmAuth()` (pages/collections/lookbooks/banners, PUT `/:id/products`, PUT `/:id/images`); публичные `/api/content/*` (только visible/active+расписание); slug → 409, unknown SKU → 404; mount в `index.ts` **до** `/api/admin` (TG-admin не затронут).
  - **Верификация оркестратора:** `git diff master` — forbidden paths (admin/payments/cdek/catalog) пусто; миграция ≡ `schema.sql` блок 017; публичный DTO совпадает с витриной (`body`, `heroImage`, `cover`, `productSlugs`, `seo`, `updatedAt`, `sortOrder`); самостоятельный `tsc --noEmit` чисто; `npm run test` — **47 файлов / 245 тестов** (+16, 229→245).
  - **Зафиксировано для B3:** PUT products/images — массив в body напрямую; `startsAt`/`endsAt` — ISO-8601 datetime.
- **B2 ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-08** (ветка `admin-phase-b`, коммит `880aebc`): `POST /api/crm/content/upload` под `requireCrmAuth()`; multer memory 10MB, поле `file`; sharp rotate+resize max 2000px+webp q85; `env.uploadsDir` (`UPLOADS_DIR`, prod default `/var/www/muru/uploads`); `uploads/` в `.gitignore`; ответ `{ url: "/uploads/<uuid>.webp", width, height }`; ошибки 400/401/413.
  - **Верификация оркестратора:** `tsc` чисто; vitest **49 / 253** (+8); route tests: 401 без cookie, 200 с cookie, 400 без file/не-image, 413 >10MB; `sharp` уже был в deps (image proxy).
  - **Заметка для B5:** staging `.env.staging.example` указывает тот же `UPLOADS_DIR` что прод — при обкатке развести (напр. `uploads-staging`).
- **B3 ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-08** (ветка `admin-phase-b`, коммит `de7d638`, push `origin/admin-phase-b`): раздел «Контент» в `admin/` — 4 таба (страницы/коллекции/лукбуки/баннеры), списки+CRUD-формы, TipTap (`bodyHtml`, `description`), `ImageUploadField`+upload API, `SkuListEditor`+PUT products, лукбук-галерея с reorder, баннеры datetime-local→ISO, auto-slug, confirm delete, redirect после create. Сайдбар: только «Контент» стал ссылкой; `backend/` не тронут.
  - **Верификация оркестратора:** 19 новых файлов в `admin/src/**` + 5 изменённых; `tsc -b` + `build` чисто (JS 695 kB / CSS 4.1 kB, chunk warning от TipTap — ок); `content-api.ts` покрывает все CRM эндпоинты + FormData upload.
  - **Ручной smoke не прогонялся** (ни исполнителем, ни оркестратором) — рекомендуется перед B5: login → page CRUD + upload + invalid SKU.
  - **Риски зафиксированы:** preview `/uploads/*` локально 404 до B6 nginx; timezone offset у `datetime-local`; bundle TipTap ~695 kB — приемлемо для admin-only.
- **B4 ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-08** (`muru-storefront/main`, коммит `06db73d`, deployed на `web.murushop.ru`): добавлены `banner.ts`/`content-backend.ts`/`home-banners.ts`; `endpoints.ts` переведён на `/api/content/*` с try/catch fallback на `src/lib/content/*`; главная (`app/page.tsx`) рендерит баннеры из `getHomeBanners()` с fallback 4 статических блоков; из MSW удалены контентные хендлеры/резолверы (чтобы не перехватывать `/api/content/*`).
  - **Верификация оркестратора:** самостоятельный `npx tsc --noEmit` — чисто; `next build` — успешно, 28/28 routes; при `NEXT_PUBLIC_API_BASE=http://localhost:4000/api` и отсутствующей миграции 017 бэкенд отдаёт 500 (`relation content_* does not exist`), но билды/страницы не падают благодаря fallback (ожидаемое поведение до B5).
  - **Риск/заметка:** без B5 контент из CRM в витрине не появится (только статика fallback), это нормально.
- **B5 (staging STOP-гейт) ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-08** (`/var/www/muru-staging`, ветка `admin-phase-b` @ `de7d638`, БД `muru_staging`): миграция 017 применена (4 таблицы content_*); `UPLOADS_DIR=/var/www/muru/uploads-staging`; `deploy-staging.sh` OK; полный smoke CRUD/public/upload/auth + storefront build против `https://api-staging.murushop.ru/api` (28/28). Прод (`muru-backend`) не тронут.
  - **Верификация оркестратора (независимая curl):** `/api/health` 200; `/api/content/pages/b5-page-test` 200 `body:"<p>B5 body</p>"`; `/api/content/collections/b5-collection-test` 200 `productSlugs:["MU0167"]`; `/api/content/lookbooks/b5-lookbook-test` 200 с images; `/api/content/banners` 200 active banner `subtitle:"From API"`; CRM без cookie → 401; TG `/api/admin/me` → 401 (не 404).
  - **Заметки:** `.env.staging.example` в репо всё ещё `/var/www/muru/uploads` — на VPS переопределено вручную на `uploads-staging` (ок). B4 в `muru-storefront` — коммит/push ещё pending.
- **B6 (DEP-010) ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-08:** backend+admin+storefront на проде. Backend: `/var/www/muru` @ `da99aeb`; `backend/.env` из корневого `.env` + `ADMIN_JWT_SECRET`; миграция 017; nginx `/uploads/`; smoke **10/10**. Storefront: `b6-storefront-deploy.sh`, `05f08bf→06db73d`, build 28/28, PM2 restart; curl сразу после restart дал 502 (гонка), через ~мин — **200** на `/`, `/landings/`, `/company/` (verified оркестратором).
  - **Инцидент B6 (закрыт):** `backend/.env` отсутствовал (только корневой `.env`) → guard ADMIN_JWT_SECRET; фикс: `cp .env backend/.env` + новый secret.
  - **Build-логи витрины:** `[content] ... fetch failed, using static fallback` для company/offer/collections — **норма** (в прод-БД пока нет этих slug; fallback работает).
- **ИТОГ ФАЗЫ: «Админка B: контент-модуль» (B1–B6) полностью завершена и верифицирована 2026-07-08.** CMS live: `murushop.ru/admin/content`, API `/api/content/*`, upload `/uploads/`, витрина `web.murushop.ru` на B4.
- **B7a ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-08** (`muru-storefront/main`, коммит `4a0b834`): `ASSETS_BASE` + `resolveAssetUrl()` в `content-backend.ts` — `/uploads/*` → абсолютный URL на домен API (`murushop.ru`); резолв heroImage/cover/images/banner.image + `body` (`src="/uploads/`); `next.config.ts` remotePatterns `murushop.ru`; fallback-статика не тронута. Аудит Fable 5 (HIGH cross-domain) закрыт.
  - **Верификация оркестратора:** `tsc --noEmit` чисто; локальный `next build` без доступа к API падает на sitemap (pre-existing, не регрессия B7a); prod `web.murushop.ru` 200.
- **B7b ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-08** (`muru-backend-local`, коммит `0d91307` на `admin-phase-c`): `vitest.setup.ts` (`NODE_ENV=test`) + `setupFiles` в `vitest.config.ts` — route-тесты content больше не зависят от `NODE_ENV=production` в локальном `.env`.
  - **Верификация оркестратора:** `npm run test` — **49 файлов / 253 теста** зелёные; `tsc` чисто.
- **Известное поведение (не баг B7):** `is_visible=false` в CMS → API 404 → витрина показывает статический fallback с тем же slug (страница не «исчезает»). Позже: fallback только на 5xx/сеть.

## Фаза: Админка C (CRM-модуль «Заказы», ТЗ §7.1–7.2)

**Старт:** 2026-07-13. **Цель:** заказчик управляет заказами обоих каналов (telegram + web) из CRM `https://murushop.ru/admin/`: список с фильтрами, карточка, смена статуса, комментарий менеджера, отмена с возвратом остатков. TG-админка mini app (`/api/admin/*`) — без изменений.

**Репозиторий:** `muru-backend-local`, ветка `admin-phase-c` от `master` (`0d91307`). Порядок: C1 backend → C2 admin UI → C3 staging STOP-гейт → C4 prod (DEP-012). **Миграций БД нет.**

### Архитектурные решения (приняты, не пересматривать)
1. **Изоляция:** новый модуль `crm-orders.service.ts` / `crm-orders.controller.ts` / `crm-orders.routes.ts` → `/api/crm/orders/*` + `requireCrmAuth()`. `admin-orders.service.ts` — образец, **не изменять**. Из `admin-orders.helpers.ts` — только импорт без правок (`normalizeAdminOrdersPage`, `normalizeAdminOrdersPageSize`, `shouldNotifyConfirmed`).
2. **Статусы:** только `constants/order-statuses.ts`. «Оплачен» в UI — бейдж по `payment_status`/`paid_at`, не статус.
3. **Контакты channel-aware:** telegram → `user_profiles`; web → `cdek_recipient_name`/`cdek_recipient_phone`. `telegramUserId: number | null` (без `Number(null)→0`).
4. **Уведомления:** PATCH статуса telegram-заказа → `notifyClientStatusChange` при `shouldNotifyConfirmed` (как TG-админка). Web — без уведомлений.
5. **`manager_telegram_id` из CRM не пишется.** Аудит смены статуса — вне скоупа v1.
6. **Вне скоупа:** ручное создание, возвраты как процесс, экспорт.

### Инварианты фазы
- Миграций нет — при необходимости → СТОП, эскалация.
- Запретные зоны: `routes/admin.ts`, `/api/admin/*`, `admin-orders.service.ts`, payments/yookassa, cdek, sync, telegram-bot, `order-statuses.ts`, `frontend/**`.
- Разрешено: `index.ts` (mount), `admin/**` (только C2), новые файлы.
- Staging: `TELEGRAM_BOT_TOKEN` пуст — ошибка отправки уведомления на staging ожидаема; проверка уведомлений — тестами C1 + прод-смоук C4.

### Состав (порядок обязателен)
| Шаг | Что | Модель |
|-----|-----|--------|
| **C1** | Backend API + vitest | Сильнее Auto |
| **C2** | Admin UI «Заказы» | Auto / сильнее при буксе |
| **C3** | Staging STOP-гейт | — (Василий) |
| **C4** | FF-merge → master, DEP-012, ранбук прода | — |

### Статус
- **C1 ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-13** (ветка `admin-phase-c`, коммит `d9c0b2a`): `/api/crm/orders/*`, vitest 51/265 (+12).
- **C2 ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-13** (коммит `f256a39`): admin UI `/orders`, `orders-api.ts`, бейджи, PATCH/cancel UI.
- **C3 ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-13** (staging `api-staging.murushop.ru` @ `f256a39`): deploy OK; login; list web+telegram; channel filter; cards #25/#27; PATCH; cancel #26 restock MU0103 1→2; repeat cancel 409; health/catalog/banners 200; TG-admin 401. Прод не тронут.
- **C4 (DEP-012) ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-13:** prod `/var/www/muru` @ `f256a39` (FF `0d91307→f256a39`), `deploy.sh` OK, PM2 reload. Re-seed `admin@muru.ru` (пароль сброшен на VPS). Смоук: login 200, CRM list web#27+telegram#25 `total:21`, health 200, crm без cookie 401; TG-admin `/api/admin/orders` → 401 без JWT mini app (как A6 — роут жив, не регрессия).
- **ИТОГ ФАЗЫ: «Админка C: CRM Заказы» (C1–C4) полностью завершена и верифицирована 2026-07-13.** Live: `https://murushop.ru/admin/orders`, API `/api/crm/orders/*`. TG-admin без изменений. Миграций не было. Откат: `git reset --hard 0d91307` + `deploy.sh`.

## Фаза: Админка D (CRM-модуль «Каталог» + подготовка cutover с Google Sheets, ТЗ §5.1–5.5)

**Старт:** 2026-07-13. **Цель:** заказчик управляет каталогом из CRM (`https://murushop.ru/admin/`): список, карточка, создание, архивирование, фото, категории, характеристики, импорт/экспорт XLSX. Источник каталога переключается `CATALOG_SOURCE`: `'sheets'` (sync жив, CRM read-only) | `'crm'` (sync мёртв, CRM пишет). На проде фаза деплоится с default `'sheets'`. Полный `'crm'` — только staging (D5). Прод-cutover на `'crm'` — вне фазы.

**Репозиторий:** `muru-backend-local`, ветка `admin-phase-d` от `master` (`f256a39`). Storefront и `frontend/` (mini app) **не трогаем**. Прод = **DEP-013**.

### Архитектурные решения (приняты, не пересматривать)
1. **`CATALOG_SOURCE`** (`'sheets'|'crm'`, default `'sheets'`) в `env.ts`. Guards на уровне **сервисов** (не роутов): при `'sheets'` мутации `/api/crm/catalog/*` → **423 LOCKED** (читать можно); при `'crm'` — `syncCatalogFromGoogle` отказывает (объект, не throw), scheduler noop, `ENABLE_SHEETS_STOCK_WRITE`-ветка noop. Upsert-ядро sync — не менять поведение sync-пути; CRM-импорт (D3) переиспользует его.
2. **Миграция 018:** `products.is_archived BOOLEAN NOT NULL DEFAULT FALSE` (+ индекс); таблица `characteristics` (справочник ключей). Публичный `/api/catalog/*` фильтрует `is_archived=FALSE`. Sync-upsert `is_archived` не трогает.
3. **Характеристики:** значения в `products.specs` JSONB; `characteristics` — только словарь ключей (D2).
4. **Категории:** CRUD `categories` + массовый rename подкатегории на `products` (D2). Таблицу `subcategories` не вводить.
5. **Фото:** `POST /api/crm/catalog/upload-image` → image-proxy cache, URL `/img/:id/...` (D2).
6. **Импорт/экспорт XLSX** (D3): только при `CATALOG_SOURCE=crm`. CommerceML — вне скоупа.
7. **Габариты:** `admin-product-dims.validation` (D1 update в карточке).
8. **Остатки:** прямое `in_stock` в карточке (журнал §6 — позже).

### Инварианты фазы
- Запретные зоны: `routes/admin.ts`, `/api/admin/*`, `admin-orders.service.ts`, `crm-orders.*`, payments/yookassa (кроме noop-guard stock-write), `cdek`, `telegram-bot`, `frontend/**`, google-sync-upsert-ядро (расширять можно, менять sync-поведение — нельзя).
- Миграция 018: staging (D5) → прод (D6). Базовая линия тестов: **51 файл / 265 тестов**.

### Состав
| Шаг | Что | Статус |
|-----|-----|--------|
| **D1** | Миграция 018 + `CATALOG_SOURCE` + guards + CRUD products + `is_archived` в публичном каталоге + vitest | **ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-13** (`admin-phase-d`, `6e0d3fe`) |
| **D2** | Категории + характеристики + upload-image + vitest | **ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-13** (`admin-phase-d`, `0d29b46`) |
| **D3** | import/export XLSX + vitest | **ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-13** (`admin-phase-d`, `484878d`) |
| **D4** | Admin UI «Каталог» | **ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-13** (`admin-phase-d`, `e5cc51c`) |
| **D5** | Staging STOP-гейт (`CATALOG_SOURCE=crm`) | **ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-13** (`admin-phase-d` @ `e5cc51c`, staging) |
| **D6** | Прод DEP-013 (`CATALOG_SOURCE` default sheets, CRM read-only) | **ВЫПОЛНЕН и ВЕРИФИЦИРОВАН 2026-07-13** (`master` @ `e5cc51c`) |
| **D7** | Категории: web-дерево + фикс delete (cross-placements) | **DEPLOYED** (`master` @ `8fa0f38`, DEP-014) |
| **D7.1** | Hotfix: `COUNT(DISTINCT p.id)` в directProductCount | **DEPLOYED** (`master` @ `2d20658`, DEP-015) |
| **D8** | Migration 019 subcategory + nginx `/img/` | **DEPLOYED** (`master` @ `0877d6d`, DEP-016) |
| **D8.1** | TD-005: mock env in sync-scheduler.test.ts | **MERGED** (`master` @ `1e439bf`, push OK; deploy не нужен) |

- **ИТОГ ФАЗЫ: «Админка D: CRM Каталог» (D1–D8.1) полностью завершена и верифицирована 2026-07-14.** Live: `https://murushop.ru/admin/catalog`, API `/api/crm/catalog/*`, nginx `/img/`, migration 018/019 на прод-БД. Прод в режиме `CATALOG_SOURCE=sheets` (CRM read-only, sync из Google Sheets). Полный cutover на `crm` — операционный шаг вне фазы, ранбук `CUTOVER.md`. Откат кода: `git reset --hard f256a39` + `deploy.sh` (миграции 018/019 аддитивны).

### Env-заметка (D1)
Сейчас в коде `CATALOG_SOURCE` = `'xlsx'|'sheets'|'crm'` в Zod; верхний уровень владения — `'sheets'|'crm'`. Legacy `xlsx` → `catalogSource='sheets'` + `googleCatalogReadMode='xlsx'`. Экспорт: `catalogSource`, `googleCatalogReadMode`, `isCatalogCrmMode`. **D1 verified:** guards, CRUD, публичный фильтр `is_archived`, 54/275 тестов.

### Статус D1 (детали verify)
- **Миграция 018** + `schema.sql` блок (`is_archived`, `characteristics`) — идемпотентно, к БД не применялась.
- **Guards:** `assertCatalogCrmWritable` → 423 LOCKED; `syncCatalogFromGoogle` early return; scheduler noop; stock-write skip при `crm`.
- **API:** `/api/crm/catalog/meta|products*` — 9 эндпоинтов, `requireCrmAuth`, list с пагинацией + `catalogSource`/`readOnly` в ответе.
- **Публичный каталог:** `is_archived=FALSE` в tree/products/by-sku/subcategories.
- **Запретные пути:** не затронуты (`admin.ts`, `crm-orders.*`, `google-sync-upsert-sql.ts`, `admin/**`).
- **Тесты (оркестратор):** `tsc --noEmit` чисто; vitest **54 файла / 275 тестов** (+3/+10 к базе 51/265).
- **Замечание:** D1 коммит `6e0d3fe`. D2 verified, коммит pending.

### Статус D2 (детали verify)
- **Категории:** list+productCount, CRUD, delete 409 при активных товарах, rename-subcategory (web+legacy subcategory, транзакция).
- **Характеристики:** list/create/patch, unique → 409.
- **Upload:** `POST /upload-image` → `crm_*` ID, `saveCrmCachedOriginal`, Drive-bypass в `downloadOriginal` (маркер `.crm-upload` + prefix).
- **Роутинг:** `POST /categories/rename-subcategory` зарегистрирован до `/:id` — OK.
- **Запретные пути:** не затронуты.
- **Тесты (оркестратор):** `tsc` чисто; vitest **55 файлов / 284 теста** — все зелёные (+1 файл, +9 тестов к D1).

### Статус D3 (детали verify)
- **Export:** `GET /export?format=xlsx|csv`, все товары incl. archived, `Content-Disposition` attachment, CSV BOM.
- **Import:** `parseXlsxBufferToCatalog` → `importCatalogProductsFromRows` → `upsertProductsBatched`; **без purge**; dry-run classify create/update.
- **google-sync:** экспорт `normalizeProductFromSheetRow` / `importCatalogProductsFromRows`; `CATALOG_SPEC_MAPPING` вынесен в `crm-catalog-sheet-map.ts`; `syncCatalogFromGoogle` + purge не затронуты; `google-sync-upsert-sql.ts` — 0 diff.
- **Guards:** import 423 при sheets; export 200 в обоих режимах.
- **Тесты:** **58 файлов / 294 теста** (+3/+10 к D2). Uncommitted — коммит перед D4.

## Блокеры
- Нет активных блокеров.

## Ожидает деплоя (Pending deploy)

Живой статус: что **закоммичено и verified**, но ещё **не на VPS** (или не на prod-БД). Детали команд — [`DEPLOY.md`](DEPLOY.md). Протокол синхронизации репо — [`FORWARD_PORT.md`](FORWARD_PORT.md).

| ID | Изменение | Репо / ветка | Код | VPS | Действие | Заметки |
|---|---|---|---|---|---|---|
| DEP-001 | Удаление небезопасного `POST /orders/create` (бэк + фронт mini app) | `MURU_miniAPP` / `master` | verified | **deployed** (Василий, 2026-07-02) | — | Коммит `9e8e8e0` на GitHub `master`; `git pull` + `deploy.sh` на VPS |
| DEP-002 | Миграция `014_web_identity.sql` (web-канал, nullable telegram_user_id) | `MURU_miniAPP` / `master` | verified | **deployed** (Василий, 2026-07-03) | — | Коммит `0fcdf41`; VPS `psql -f` — 4×ALTER + DO OK |
| DEP-003 | Web checkout API (`/payments/web/*`, `/cdek/web/calculate`, channel-aware YK) | `MURU_miniAPP` / `master` | verified | **deployed** (Василий, 2026-07-03) | — | Коммит `52b23fd`; `deploy.sh`; smoke mini app OK |
| DEP-004 | Web catalog F/G/H + hotfix subcategory | `MURU_miniAPP` / `master` | verified | **deployed** (Василий, 2026-07-04) | — | `c2df422` + hotfix; migration 015; sync ~260 |
| DEP-005 | Storefront `channel=web` cross-listing | `muru-storefront` / `main` | verified | **deployed** (Василий, 2026-07-04) | — | `b419a3f` на `web.murushop.ru` |
| DEP-006 | Rate limits + `schema.sql` parity | `MURU_miniAPP` / `master` | verified | **deployed** (Василий, 2026-07-04) | — | `2c97f0e`; restock 6-й → 429 |
| DEP-007 | `catalog-menu` fallback + gitignore | `muru-storefront` / `main` | verified | **deployed** (Василий, 2026-07-04) | — | `05f08bf`; build 28/28 |
| DEP-008 | Cutover прода `/var/www/muru`: `origin` → канон `VasiliiLbyte/muru-backend-local`, `reset --hard origin/master` (`6e7ab0f`), `deploy.sh` | `muru-backend-local` / `master` | verified | **deployed** (Василий, 2026-07-06) | — | Откат: `git remote set-url origin https://github.com/VasiliiLbyte/MURU_miniAPP.git` + `git reset --hard 2c97f0e` + `bash deploy.sh`. Смоук OK (health/catalog/Mini App checkout/web.murushop.ru) |
| DEP-009 | Админка A: фундамент — миграция `016_admin_users`, `/api/admin-auth/*`, SPA `admin/` на `/admin/` | `muru-backend-local` / `master` | verified | **deployed** (Василий, 2026-07-06) | — | `17876da` (FF-merge `admin-phase-a`); миграция на прод-БД до деплоя кода; seed owner `admin@muru.ru`; смоук login/me/logout 200, TG-admin не регрессировал, браузерный вход OK |
| DEP-010 | Админка B: контент-модуль — 017, CRM/public API, upload, admin CMS, storefront B4 | `muru-backend-local` `da99aeb` + `muru-storefront` `06db73d` | verified | **deployed** (Василий, 2026-07-08) | — | B6 smoke 10/10; nginx `/uploads/`; `web.murushop.ru` 200 |
| DEP-011 | B7a: CMS `/uploads/` cross-domain на витрине | `muru-storefront` / `main` (`4a0b834`) | verified | **deployed** (Василий, 2026-07-13) | — | curl HTML: `https://murushop.ru/uploads/*.webp` на главной; баннеры из CMS видны |
| DEP-012 | Админка C: CRM «Заказы» — `/api/crm/orders/*`, admin `/orders` | `muru-backend-local` / `master` (`f256a39`) | verified | **deployed** (Василий, 2026-07-13) | — | FF merge; deploy.sh; login+list smoke OK; TG-admin 401 без JWT — ожидаемо |
| DEP-013 | Админка D: CRM «Каталог» — 018, `/api/crm/catalog/*`, admin `/catalog`, `CATALOG_SOURCE` | `muru-backend-local` / `master` (`e5cc51c`) | verified | **deployed** (Василий, 2026-07-13) | — | FF `f256a39→e5cc51c`; миграция 018 на `muru_db`; schema drift `subcategory`/`subcategory_slug` допатчены; `CATALOG_SOURCE=xlsx` (legacy sheets); deploy.sh; API smoke: meta readOnly, 423 LOCKED; re-seed `admin@muru.ru` |
| DEP-014 | D7: CRM categories web tree + safe delete | `muru-backend-local` / `master` (`8fa0f38`) | verified | **deployed** (Василий, 2026-07-13) | FF `e5cc51c→8fa0f38`; `deploy.sh`; без миграции | браузерный смоук categories — по желанию |
| DEP-015 | D7.1: DISTINCT directProductCount hotfix | `muru-backend-local` / `master` (`2d20658`) | verified | **deployed** (Василий, 2026-07-13) | FF `8fa0f38→2d20658`; smoke cat#821: 49/1/2 | — |
| DEP-016 | D8: migration 019 + nginx `/img/` | `muru-backend-local` / `master` (`0877d6d`) | verified | **deployed** (Василий, 2026-07-13) | psql 019 (NOTICE skip OK); `sync-nginx-murushop.sh`; `/img/` → 200 | TD-001, TD-004 closed |
| DEP-017 | Cutover prep + invoice hotfix: env log, `MINIAPP_MAINTENANCE`, sync 423 crm, `telegram.me` invoice URL | `muru-backend-local` / `master` (`a98222e`) | verified | pending | `git pull` + `deploy.sh`; без миграции | Было `da280b2` (guard только t.me ломал оплату); cutover — `CUTOVER.md` после smoke |

**Как обновлять:** оркестратор добавляет строку при verify prod-затрагивающей задачи; после деплоя Василий сообщает → колонка VPS = `deployed`, строка переносится в «Сделано» или помечается ✅.

## Бэклог (не терять)
- Картинки плиток подкатегорий (пустые серые боксы) — фаза CMS; category-grid.tsx деградирует мягко.
- ~~Схлопнуть две папки бэкенда в один git-репо с ветками (убрать merge-боль на cutover).~~ ✅ ЗАКРЫТО 2026-07-06 (унификация U1-U4, `MURU_miniAPP` заморожен `7877be1`, прод на каноне DEP-008).
- `src/lib/content/collections.ts`: `productSlugs` у всех коллекций сейчас `[]` (раньше брались из mock-товаров, разорвано при выносе в статичный модуль) — заполнить реальными SKU после согласования наполнения лендингов с заказчиком.
- CDEK debug-урок для памяти: ранний симптом «расчёт цены не приходит» (сессия 2026-07-02) свёлся к пустым CDEK-кредам в `.env` на момент теста — сам расчётный код был исправен с самого начала. Перед глубоким дебагом смотреть `.env` в первую очередь.
- Rate limiter in-memory per-process — при масштабировании вынести в Redis.
- `getProducts` витрины тянет весь каталог и фильтрует в Node — ок до ~1000 SKU, дальше серверные query-параметры.
- Конвенция: `schema.sql` держим синхронным с миграциями в обоих backend-репо.
- `sitemap.xml` при билде без API — отдельный fallback (follow-up из 2026-07-04-02; `catalog-menu` уже fixed).

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
- **2026-07-04 (сессия 2)**: Полный аудит Claude; ремедиация промпты `2026-07-04-01/-01-fp/-02`; коммиты `84cab5e`/`2c97f0e`/`05f08bf`; VPS deploy + verify (restock 429, build 28/28, browser + mini app OK). DEP-006/007 ✅. muru-docs sync (`2026-07-04-03`).
- **2026-07-06**: Старт фазы «Унификация бэкендов». Промпт `2026-07-05-01` (U1 аудит дрейфа, read-only) выдан исполнителю в `muru-backend-local`; исполнитель совместил его с фиксом (`6c8ba48`) и U2-merge (`storefront-integration→master` FF `56a32c2→6c8ba48`, 6 коммитов). Оркестратор независимо верифицировал: frontend/backend/инфра идентичны канону кроме 2 давно известных косметических отличий; ложный red-flag от `git cherry` (5 коммитов) опровергнут прямым diff содержимого. `master` теперь унифицированный канон.
- **2026-07-06 (сессия 2)**: U3-a (промпт `2026-07-06-01`, коммит `6e7ab0f`) — staging-скаффолдинг (ecosystem/deploy-staging.sh/.env.example/nginx), верифицирован. U3-b/c — staging развёрнут и полностью смоук-протестирован на VPS (`api-staging.murushop.ru`, БД `muru_staging` восстановлена из прод-дампа, 260 products), прод не задет на всём протяжении (103 рестарта без изменений). По пути закрыты 5 несвязанных с кодом инцидентов инфраструктуры (см. блок «Инциденты по пути» выше в разделе U3-b/c): дефолтная ветка GitHub `main` vs `master`, PG15 schema-ownership, случайно непустой `TELEGRAM_BOT_TOKEN`, обязательные YooKassa-переменные при `NODE_ENV=production`, certbot chicken-egg с nginx. U3 закрыт.
  **U4 (cutover прода) выполнен и верифицирован в этой же сессии**: `/var/www/muru` переключён на канон (`origin` → `muru-backend-local`, `reset --hard` на `6e7ab0f`, `deploy.sh` — backend+frontend сборка чисто, один `pm2 reload`). Смоук: health/catalog/root 200, ложная тревога с `yookassa-webhook` 404 разобрана (это штатный `yookassaIpGuard`, не баг), ручной смоук Василия — Mini App чекаут доходит до ЮKassa, `web.murushop.ru` работает. DEP-008 deployed. Точки отката задокументированы. **Унификация бэкендов (U1-U4) полностью завершена за одну сессию.** Следующий шаг — U4-post (заморозка `MURU_miniAPP`) и обновление DEPLOY.md/FORWARD_PORT.md/ORCHESTRATOR.md.
- **2026-07-06 (сессия 3)**: U4-post — `MURU_miniAPP` заморожен (`7877be1`, баннер в README, verified). Обновлены DEPLOY.md/FORWARD_PORT.md (deprecated)/ORCHESTRATOR.md. Собран промпт для внешней проверки унификации (для Claude Fable 5 через desktopcommander). **Унификация полностью закрыта.** Старт фазы «Админка A: фундамент» (CRM v0.1, ТЗ §3): архитектура зафиксирована (admin/ SPA на `/admin/`, email+пароль bcrypt, отдельный `ADMIN_JWT_SECRET`, роли owner/manager, `/api/crm/*`+`/api/admin-auth/*`, workflow через ветку `admin-phase-a` + staging-first для миграции 016). Заведён раздел «Фаза: Админка A». Готовится промпт A1 (миграция + auth-ядро).
- **2026-07-06 (сессия 4)**: Фаза «Админка A» полностью пройдена A1→A6 (executor Codex 5.3, ветка `admin-phase-a`, верификация оркестратора на каждом шаге — чтение кода + самостоятельный `tsc`/`vitest`/`build`). A1 auth-ядро (`8aa5f14`), A2 seed-скрипт (`a1fbf5c`), A3 SPA-каркас `admin/` (`48f62ee`), A4 деплой-интеграция (`154955d`). На A5 (staging) найден и исправлен реальный баг: `npm ci --omit=dev` вычищал `tsx`, на котором был завязан `seed:admin` — фикс на запуск из `dist/` (`17876da`). Полный смоук на staging (7/7 пунктов) прошёл после фикса. A6 (прод, DEP-009): FF-merge в `master`, миграция 016 на прод-БД до деплоя кода, `ADMIN_JWT_SECRET` независимый, seed боевого owner, полный смоук + ручной браузерный вход (скрин Василия) + Mini App без регрессий. Пойман и объяснён инцидент с временным crash-loop (`104→120` рестартов) из-за порядка операций (deploy до правки `.env`) — самостоятельно устранился после донастройки. **Фаза закрыта.**
- **2026-07-08**: **Фаза «Админка B» закрыта (DEP-010).** B1–B6 verified; staging B5; prod deploy+smoke 10/10; storefront `06db73d` на `web.murushop.ru` (502 сразу после PM2 restart — transient, затем 200).
- **2026-07-08 (сессия 2)**: Аудит Fable 5 → B7a/B7b. Verified: `4a0b834` (storefront asset URL resolve + remotePatterns), vitest setup 49/253.
- **2026-07-13**: **DEP-011 deployed** — B7a на `web.murushop.ru`; verify: главная отдаёт абсолютные `https://murushop.ru/uploads/...`, баннеры из CMS.
- **2026-07-13**: **Фаза C закрыта (DEP-012).** Prod `f256a39`: CRM orders API + `murushop.ru/admin/orders`. Staging C3 + prod API smoke OK.
- **2026-07-13 (сессия 2)**: Старт **фазы D** (CRM «Каталог», ТЗ §5). Архитектура зафиксирована (`CATALOG_SOURCE` sheets/crm, миграция 018, guards, staging-first). Выдан промпт **D1** (`2026-07-13-01`) → ветка `admin-phase-d`.
- **2026-07-13 (сессия 3)**: **D1 verified** (оркестратор): миграция 018, env guards, CRM catalog CRUD, публичный `is_archived`, 54/275 тестов. Коммит `6e0d3fe` на `admin-phase-d`.
- **2026-07-13 (сессия 4)**: **D2 verified** + коммит `0d29b46`: categories/characteristics/upload-image, image-proxy CRM bypass, 55 файлов / 284 теста.
- **2026-07-13 (сессия 5)**: **D3 verified** + коммит `484878d`: export xlsx/csv, import dry-run + upsert без purge, 58/294 тестов.
- **2026-07-14 (сессия 4)**: **Invoice hotfix verified + merged.** `a98222e`: `isValidTelegramInvoiceUrl` принимает `https://telegram.me/$` (Telegram API после проблем DNS t.me); без нормализации в t.me. vitest 62/314. DEP-017 обновлён до `a98222e`.
- **2026-07-14 (сессия 3)**: **Cutover prep verified + merged.** `master` @ `da280b2` (4 коммита), push OK; vitest 62/311; DEP-017 pending deploy. `muru-docs` `5cbd631` (CUTOVER.md env consolidation).
- **2026-07-14 (сессия 2)**: Cutover prep — ветка `fix/env-maintenance-cutover`: env observability, `MINIAPP_MAINTENANCE`, invoice URL guard, admin sync 423 в crm, тесты 62/311, `CUTOVER.md` обновлён. Merge/deploy — pending.
- **2026-07-14**: **Фаза D закрыта.** D8.1 FF-merge в `master` (`1e439bf`), push OK, деплой не требуется (только тесты). TD-005 closed. Следующий шаг — по промпту Василия.
- **2026-07-13 (сессия 15)**: **D8.1 verified** (`admin-phase-d81`): mock `env.isCatalogCrmMode` в sync-scheduler.test.ts + тест crm skip. vitest 299/299 green (default + `CATALOG_SOURCE=crm`). TD-005 closed.
- **2026-07-13 (сессия 14)**: **DEP-016 deployed.** D8 @ `0877d6d`: migration 019 + nginx `/img/`; smoke `/img/` → 200. TD-001, TD-004 closed.
- **2026-07-13 (сессия 12)**: **DEP-015 deployed.** D7.1 hotfix `2d20658` на прод. Smoke cat#821: 49/1/2.
- **2026-07-13 (сессия 11)**: **D7.1 verified** (`admin-phase-d71`): `COUNT(DISTINCT p.id)` + test assert. vitest 59/298.
- **2026-07-13 (сессия 10)**: **DEP-014 deployed.** Прод `/var/www/muru` @ `8fa0f38` (D7: categories web tree + safe delete). `git pull` + `deploy.sh` OK.
- **2026-07-13 (сессия 9)**: **D7 verified** (промпт `2026-07-13-02`): `listCrmCategories` 2 SQL + merge; delete cross/FK guards; admin tree UI. vitest 59/298; admin build OK.
