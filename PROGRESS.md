# MURU — Прогресс и память проекта

Живой рабочий журнал. Обновляется в конце сессий. Версионируется git.
Последнее обновление: 2026-07-15 (Admin Design System **COMPLETE** @ `bcc4be7`; merge/deploy — отдельное решение)

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
- **Cutover Sheets → CRM — ВЫПОЛНЕН 2026-07-14 (DEP-018):** prod `CATALOG_SOURCE=crm`; sync #54 (264 SKU); backup 177K; смоук invoice/витрина/CRM write/archive/`/img/` OK.
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

- **Cat v2 D+E + follow-ups — ВЫПОЛНЕНО и MERGED 2026-07-15** (`origin/master`=`0482f00`, `origin/main`=`34b67bb`):
  - **D** (`d1af333`): admin «Каталог и разделы» — hub, CategoryDetail, GiftGuide, redirects; Content = pages+banners.
  - **E1** (`9aeeb5a`): migration `026`, CRM/public hotspot API, vitest 355→361.
  - **E2** (`84a78d3`): `HotspotEditor` на lookbook edit.
  - **E3** (`d028b74`): storefront hero hotspots + ProductGrid + `generateStaticParams`.
  - **F1** (`3499101`): hero natural aspect ratio (не 21:9 crop).
  - **G1** (`d5c9a3e`): `banner_image` column, migration `027`, cover≠banner API.
  - **G2** (`0482f00`): admin Обложка/Баннер, галерея убрана, hotspots на banner.
  - **G3** (`4383933`): storefront banner, portal popover flip, gallery removed.
  - **H1** (`34b67bb`): compact square cards на `/lookbooks/`.
  - **Prod:** DEP-026/027/028 deployed; browser smoke OK. **Merge:** FF `feature/crm-categories-e`→`master`, `feature/inspiration-hotspots`→`main` (сессия 21). **VPS checkout:** `/var/www/muru`→`master`=`0482f00`, `/var/www/muru-storefront`→`main`=`34b67bb`, migration `027` (`banner_image`) — OK; curl smoke 4×200. **Pending:** mini app regression smoke.

## Следующее
- **Admin Design System merge:** FF `feature/admin-design-system` → `master` (`bcc4be7`, 6 коммитов UI-1…UI-4) — по решению Василия; затем деплой admin bundle (`deploy.sh`, без миграций). **Manual smoke до merge (рекомендуется):** login → dashboard → products (filters+table) → product edit (★+toast) → order confirm → TipTap prompt → hotspot drag → DevTools `prefers-reduced-motion`.
- **Mini App regression smoke (обязательно):** каталог → корзина → checkout TG после merge; записать PASS/FAIL в PROGRESS.
- **Контент вдохновения:** в admin — отдельно обложка (сетка) и баннер (hotspot'ы) для каждого лукбука.
- **Follow-up (не блокеры):** HTML description на storefront; стиль маркеров hotspot; DEP-025 admin preview covers.
- **Follow-up (DEP-025, опц.):** admin preview обложек категорий/подкатегорий через `/img/`; ↑↓ порядок подкатегорий при `sort_order=0`.
- **Боевая валидация** (§4 `CUTOVER.md`): ~30 товаров через CRM; freeze Sheets с клиентом (п.6).
- **Hotfix follow-up:** ~~O-1..O-5~~ ✅ DEP-019/020/021 deployed. Open: `MINIAPP_MAINTENANCE=false`, cutover §3.8 TG sync 423.
- **Операционно:** пре-чеклист п.8 (`backend/.env` → `.unused`); подтвердить п.8 смоука (TG sync 423).

## Фаза: Категории v2 (A/B/C — реструктуризация категорий/подкатегорий CRM)

**Старт:** 2026-07-14. **Цель:** товар — одна основная категория + 0..N подкатегорий-сущностей (не текст); виртуальная «Распродажа» (по `discount_percent>0`, без ручного назначения); чекбокс «Гид по подаркам»; фиксированные характеристики в `specs`. ТЗ — промпт Василия целиком в сессии, детали архитектуры см. коммиты/диффы ниже.

**Репозитории:** `muru-backend-local` (`backend/`+`admin/`), ветки `feature/crm-categories-a`/`-b`/`-c` от `master`; `muru-storefront` — только точечные правки при необходимости. Порядок строго A→B→C, стоп и ревью после каждой фазы. Прод/staging не трогаются до отдельного решения о деплое (после C или по фазам — решить отдельно).

**Merge/push (2026-07-15):** `origin/master` = `0482f00`, `muru-storefront/main` = `34b67bb`. **Prod deployed** DEP-026/027/028 (сессии 19–21).

### Prod-раскатка (2026-07-14) — ВЫПОЛНЕНА и ВЕРИФИЦИРОВАНА

**VPS:** `/var/www/muru` @ `14f438b`, PM2 `muru-backend` ↺139; `/var/www/muru-storefront` @ `58d43dd`, PM2 ↺39. Backup: `/root/backups/muru_db_pre_categories_v2_2026-07-14_1619.dump` (107K). Откат: код `cf5c0a3` + restore дампа.

**Pre-check:** taxonomy `web_sub=243, sub=0` (источник 024 — `web_subcategory_name`); 18 SKU в прямой «Распродажа»; virtual Sale=20; categories=23.

**Миграции 020–025:** все exit 0. 020: 13 junk удалено; 021: 18 SKU → «Без категории»; 024: **17 subcategories / 242 links**; post: categories=10, bez_kategorii=19, direct_sale=0.

**Deploy + API smoke:** health ok; Sale **20**; giftGuide **0**; tree subcategories **17**; admin login **200**; PATCH→Sale **409**.

**Storefront:** build 28/28, `/gifts/` **200**, home/catalog **200**. Build-warn: CMS content 404 → static fallback (ожидаемо, не регрессия).

**Инциденты (не блокеры):** `.env` канон — `/var/www/muru/.env` (не `backend/`); `pg_dump`/`psql` через `DATABASE_URL`; login smoke — JSON без кавычек в пароле давал 500.

### Staging-раскатка (2026-07-14) — ВЫПОЛНЕНА и ВЕРИФИЦИРОВАНА

**VPS:** `api-staging.murushop.ru` / `/var/www/muru-staging` @ `14f438b`, PM2 `muru-backend-staging` (:4001), БД `muru_staging`. Backup: `/root/backups/muru_staging_pre_categories_v2_2026-07-14_1417.dump` (106K). Прод `muru-backend` ↺138 — не тронут.

**Миграции 020–025:** идемпотентно OK. Ключевой гейт 024 на staging: источник taxonomy = **`web_subcategory_name`** (238 товаров, `subcategory` пуст); **16 subcategories / 238 links**. 021: **18 SKU** из прямой «Распродажа» → «Без категории»: MU0175–MU0190, MU0193, MU0200. 020: удалены 13 мусорных категорий-dубликатов.

**API smoke (curl с VPS :4001):** Sale listing 20, giftGuide=0→после пометки; login 200; guards 409 (PATCH→Sale, rename/delete Sale); tree subcategories=16; `--data-urlencode` обязателен для кириллицы в query.

**Admin smoke (local admin + SSH DB tunnel → staging БД):** 1✅ категории/подкатегории; 2✅ CRUD (preview обложек в таблице — битые `<img>`, данные в API OK, follow-up); 3✅ пустая подкатегория в дереве (`Тест304` под «Флористика») → **решение A: оставить** пустые в меню; 4✅ MU0185 `subcategoryIds [4,9]`, write-through `Сухоцветы`, листинги «Натуральный декор»+«Интерьер»; 5✅ specs фиксированные + «Тип»; 6✅ «Распродажа» нет в селекте + 409; 7✅ giftGuide `['MU0185']`; 8✅ Sale 20→19 при снятии скидки.

**Инциденты по пути (не блокеры):** (1) миграции до `git pull` — файлов не было (HEAD `1392cb7`), исправлено порядком pull→migrate; (2) `source .env` на VPS ломается на скобках в `CDEK_SENDER_ADDRESS` — использовать `export DATABASE_URL=$(grep …)`; (3) local admin: `.env` перебивает tunnel URL — временная правка `DATABASE_URL` на `:5433` + restore из `.env.backup-local`.

**STOP-гейт:** прод не тронут. Прод-раскатка — отдельный промпт (DEP-022…025).

### Статус
- **Фаза A (чистка категорий + виртуальная Распродажа) ВЫПОЛНЕНА и ВЕРИФИЦИРОВАНА 2026-07-14** (ветка `feature/crm-categories-a`, коммит `dc04b70`, не пушена/не смержена):
  - Миграции `020_catalog_categories_cleanup.sql` (удаляет категории вне `TOP_LEVEL_CATEGORIES`+«Без категории» без активных товаров/крос-размещений, идемпотентно, `RAISE NOTICE`) и `021_catalog_sale_direct_reassign.sql` (переносит прямых товаров «Распродажа» → «Без категории», чистит крос-размещения на неё).
  - `constants/catalog-top-level.ts`: `SALE_CATEGORY_NAME`. `catalog.service.ts`: виртуальное членство «Распродажа» в `getCatalogTree`/`getCatalogProducts` = активные товары с `discount_percent>0` (публичный контракт не менялся, только состав выборки).
  - **Доп. гварды (решение оркестратора, приняты и подтверждены Василием):** `crm-catalog.service.ts` — 409 на прямое назначение `categoryId`=«Распродажа» при create/update товара; `crm-catalog-categories.service.ts` — 409 на удаление и на переименование/смену slug категории «Распродажа» (cover-патч разрешён).
  - `admin/ProductEditPage.tsx`: «Распродажа» скрыта из select категории товара.
  - **Верификация оркестратора (независимая, полная):** диффы всех 10 файлов прочитаны построчно и соответствуют промпту; `tsc --noEmit` (backend) и `tsc -b` (admin) — чисто (самостоятельный прогон); `vitest run` — **62 файла / 332 теста зелёные** (было 321, +11; 2 ложных фейла на первом прогоне — `connect EPERM`, сетевой шум сэндбокса на supertest-портах, не код, подтверждено повторным прогоном с `full_network` и изолированными прогонами каждого файла); БД `muru_local` после миграций — 10 категорий (удалена тестовая «Тратата»), у «Распродажа» прямых товаров 0, `discount_percent>0` активных — 21; живой `GET /api/catalog/tree` — узел «Распродажа» присутствует; `GET /api/catalog/products?categorySlug=распродажа` — 21 товар, у товаров в ответе их настоящая категория («Без категории», «Вазы и аксессуары», «Текстиль»), не «Распродажа».
  - **Известное ограничение:** `category=Распродажа` + `subcategory`/`subcategorySlug` одновременно — фильтр по подкатегории не применяется (у виртуальной Распродажи подкатегорий нет до фазы B).
  - Storefront: код не менялся (публичный контракт формы ответа не изменился); визуальная проверка `/catalog/rasprodazha/` — не прогонялась, следующий шаг по желанию.
- **Фаза B (подкатегории как сущности) ВЫПОЛНЕНА и ВЕРИФИЦИРОВАНА 2026-07-14** (ветка `feature/crm-categories-b` от `feature/crm-categories-a`, коммит `f9e2327`, не пушена/не смержена):
  - Миграции `022_subcategories.sql`/`023_product_subcategories.sql` (новые таблицы) + `024_subcategories_backfill.sql` (бэкфилл, идемпотентно).
  - **Критичная находка при верификации, исправлено оркестратором на месте:** исходное ТЗ фазы B указывало источником бэкфилла `web_subcategory_name` — но прямой SQL-запрос к `muru_local` показал: это поле **пусто у всех 265 активных товаров** (легаси-фича «Web catalog F/G/H», DEP-004 2026-07-04, судя по всему клиент так и не заполнил эти столбцы в таблице), а реальные taxonomy-данные (243/265 товаров: «Посуда», «Сервировка», «Вазы и кувшины» и т.д.) лежат в другой легаси-колонке — `subcategory` (migration 019, D-фаза). Бэкфилл строго по ТЗ дал `Created 0 subcategory row(s)` — технически верно исполнено агентом, но результат бесполезен на реальных данных (публичное дерево с `?subcategories=1` отдавало пустые `children` везде). Оркестратор переписал миграцию 024 на `COALESCE(NULLIF(web_subcategory_name,''), NULLIF(subcategory,''))` (приоритет — «правильному» полю, если оно когда-то заполнится; фоллбэк — на легаси, где реально есть данные) — после фикса: **16 подкатегорий / 243 связки**, живой `GET /api/catalog/tree?subcategories=1` показывает реальные подкатегории по всем 7 обычным категориям («Посуда», «Сервировка», «Флористический инструмент», «Свет», «Постеры» и т.д.), «Распродажа» — пусто (верно, виртуальная). Идемпотентность повторного прогона подтверждена (0/0 на втором запуске).
  - Write-through: `crm-catalog.service.ts` (`createCrmCatalogProduct`/`updateCrmCatalogProduct`) — транзакции (BEGIN/COMMIT/ROLLBACK), `subcategoryIds[0]` → денормализация в `web_subcategory_name/slug`+`subcategory/subcategory_slug`; `[]` → NULL; поле не передано → связки не трогаются. Новый файл `catalog-membership.helpers.ts` — переиспользуемые SQL-фрагменты OR-membership (прямая категория ИЛИ подкатегория-сущность ИЛИ cross-placement), использованы единообразно в `catalog.service.ts` (публичное дерево/листинг/счётчики) и `crm-catalog-categories.service.ts` (счётчики/delete-guard). Новый `crm-catalog-subcategories.service.ts` — CRUD под `/api/crm/categories/:id/subcategories` (409 на дубль slug и на удаление занятой подкатегории).
  - Admin: `ProductEditPage.tsx` — мультиселект подкатегорий **по всем категориям** (не только по выбранной у товара), первая = основная (★), порядок кнопками ↑↓, текстовый инпут `webSubcategoryName` убран; `CategoriesPage.tsx` — полноценный CRUD подкатегорий (создание/переименование/обложка через существующий upload/порядок/удаление).
  - **Верификация оркестратора (независимая, полная):** диффы всех 19 файлов прочитаны построчно; `tsc --noEmit`/`tsc -b` — чисто; `vitest run` — **63 файла / 341 тест зелёные** (было 332, +9), самостоятельный прогон совпал с отчётом; прямые SQL-запросы к `muru_local` до и после фикса миграции; живые curl к `/api/catalog/tree?subcategories=1` до и после. Storefront: код не менялся (публичный контракт формы не изменился, `coverImageUrl` у подкатегорий уже поддерживался типом `Category`/`CategoryGrid`) — визуальная проверка не прогонялась.
  - **Риск на будущее:** на проде/staging нужно повторно проверить, действительно ли `subcategory` (не `web_subcategory_name`) — верный источник; если там иначе — миграция 024 всё равно корректна благодаря `COALESCE` (не потребует правок).
- **Фаза C (характеристики + «Гид по подаркам») ВЫПОЛНЕНА и ВЕРИФИЦИРОВАНА 2026-07-14** (backend/admin: ветка `feature/crm-categories-c` от `feature/crm-categories-b`, коммит `14f438b`; storefront: `main`, коммит `58d43dd`, не пушен):
  - Миграция `025_product_gift_guide.sql` (`is_gift_guide` + частичный индекс), применена к `muru_local`.
  - `crm-catalog.service.ts`/`catalog.service.ts`: `isGiftGuide`/`giftGuide` в create/update/list/detail (CRM+публичный), аддитивный фильтр `giftGuide=true` в обоих API.
  - Admin `ProductEditPage.tsx`: свободный редактор specs заменён на 4 фиксированных поля (Материал/Страна производитель/Размер/Бренд) + read-only «Тип» (авто из первой подкатегории). `buildSpecs()` берёт `product.specs` целиком за основу, удаляет только управляемые ключи — легаси-ключи (`Плотность ткани` и т.п.) не теряются (бэкенд по-прежнему делает полную замену `specs`, merge — целиком на фронте). Легаси `Страна` консолидируется в `Страна производитель` на сохранении. `size`-колонка синхронизируется со `specs['Размер']`.
  - **Важная деталь верификации:** отчёт исполнителя явно указал 3 «ручные проверки» как невыполненные («осталось в UI») — оркестратор не принял отчёт на слово и прогнал их сам напрямую через реальный CRM API (не юнит-тестами) на живом `muru_local`: сброшен пароль локального owner (`seed:admin`), логин, `GET` детали MU0176 (id 165, реальный товар с `specs = {Тип, Бренд, Страна, Материал}` + временно инжектированный тестовый ключ `Плотность ткани` — удалён обратно после проверки), `PATCH` с телом, идентичным тому, что реально отправит `buildBody()`/`buildSpecs()` при изменении только «Материала». Результат совпал с ожиданием на 100%: `Плотность ткани` сохранился, `Страна` заменён на `Страна производитель`, `Тип` удалён (у MU0176 нет подкатегории — один из 18 SKU, переназначенных фазой A на «Без категории»), `Материал` обновлён, `Бренд`/`Размер` не потеряны. Далее: `isGiftGuide=true` на MU0176 → подтверждено оба фильтра (`GET /api/catalog/products?giftGuide=true` и `GET /api/crm/catalog/products?giftGuide=true` вернули ровно MU0176). Тестовые данные восстановлены в исходное состояние после проверки.
  - Storefront: `/gifts/page.tsx` — реальная выборка `getProducts({ giftGuide: true, page, pageSize: 24 })` (client-side фильтр, как `onSale` — публичный каталог-бэк тянется целиком и фильтруется в Node, см. существующий паттерн) + `CatalogPagination` + пустое состояние. Живая проверка: локальный dev-сервер (`next dev`, порт 3000) против локального бэка — `/gifts/` → 200, товар с `isGiftGuide=true` (MU0176, «Керамический подсвечник») отрендерен.
  - **Верификация оркестратора (независимая, полная):** диффы всех 17 файлов (12 backend/admin + 5 storefront) прочитаны построчно; `tsc --noEmit`/`tsc -b` (backend+admin) и `tsc --noEmit`+`next build` (storefront) — чисто, самостоятельный прогон; `vitest run` — **63 файла / 345 тестов** зелёные (было 341, +4).
  - **Подтверждённый эффект по ТЗ (не баг):** первое сохранение через новую форму товара без подкатегории удаляет `specs['Тип']`, даже если там было sheet-импортированное значение — так и задумано (п.5 промпта).
- **ИТОГ ТРИПТИХА: фазы A/B/C «Категории v2» + staging + prod — полностью выполнены и верифицированы за 2026-07-14.** Git: `14f438b` / `58d43dd`; prod live (DEP-022…024).

### Категории v2 — оставшиеся фазы (план, НЕ путать с глобальной «Фазой D» админки / Sheets cutover)

> **Именование:** «**Категории v2 фаза D/E**» — только реорганизация каталога и hotspot-вдохновение. Глобальная **Фаза D** (CRM «Каталог», migration 018, Sheets cutover) — **закрыта** 2026-07-14.

| Фаза | Скоуп | Оценка | Статус |
|------|--------|--------|--------|
| **A/B/C** | Подкатегории-сущности, виртуальная Распродажа, specs, giftGuide | 3 сессии | ✅ done + staging-verified |
| **D (Cat v2)** | **Реорганизация админки:** единый раздел «Каталог и разделы» | ~1–2 сессии | ✅ done + merged (`0482f00`) |
| **E1 (Cat v2)** | Миграция `026`, CRM/public hotspot API | ~1 сессия | ✅ done + merged |
| **E2 (Cat v2)** | Admin `HotspotEditor` | ~1 сессия | ✅ done + merged |
| **E3 (Cat v2)** | Storefront hero + hotspots + ProductGrid | ~1 сессия | ✅ done + merged (`34b67bb`) |
| **F1/G1–G3/H1** | Aspect fix, cover/banner, popover, index cards | follow-ups | ✅ done + merged |

**Порядок:** D → E. **Деплой:** миграция → backend → admin → storefront (аддитивно).

### Cat v2 D+E — merged + prod (2026-07-15)

**Git канон:** `origin/master`=`0482f00`, `origin/main`=`34b67bb`. Feature-ветки смержены FF.

**VPS (до checkout):** `/var/www/muru` на `feature/crm-categories-e`; `/var/www/muru-storefront` на `feature/inspiration-hotspots` — SHA совпадают с каноном. **Следующий шаг:** `git checkout master/main` на VPS (тривиально).

**Откат кода:** backend `14f438b`, storefront `58d43dd`. Backup: `muru_db_pre_cat_v2_de_2026-07-15_0613.dump`.

## Cutover (операционный, 2026-07-14) — ВЫПОЛНЕН

**Prod @ `1392cb7`:** `CATALOG_SOURCE=crm`, `ENABLE_SHEETS_STOCK_WRITE=false`. Финальный sync #54: 264 SKU. Бэкап `muru_db_pre_cutover_2026-07-14_0843.dump` (177K). Смоук: invoice, витрина, CRM write, `/img/crm_*` 200, архив/разархив OK. Откат доступен 48 ч (§5 `CUTOVER.md`).

**Follow-up (не блокируют cutover):** create product UI bug; storefront CRM images; category tile images on web.

## Cutover prep (код + prod, 2026-07-14)

**Prod @ `1392cb7` (DEP-017 deployed):** `[env] loaded`, `MINIAPP_MAINTENANCE`, sync 423 crm, invoice: accept `telegram.me/$` + normalize → `t.me/$` для `openInvoice`. **Staging @ `1392cb7` (TD-002 closed 2026-07-14):** `reset --hard origin/master` + `deploy-staging.sh`; health 200, db connected; prod `muru-backend` не тронут (↺131). Ранбук cutover — `CUTOVER.md`.

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

- **ИТОГ ФАЗЫ: «Админка D: CRM Каталог» (D1–D8.1) полностью завершена и верифицирована 2026-07-14.** Live: `https://murushop.ru/admin/catalog`, API `/api/crm/catalog/*`, nginx `/img/`, migration 018/019 на прод-БД. **Cutover 2026-07-14:** прод `CATALOG_SOURCE=crm` (DEP-018). Откат кода: `git reset --hard f256a39` + `deploy.sh` (миграции 018/019 аддитивны); откат режима: §5 `CUTOVER.md`.

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
| DEP-017 | Cutover prep + invoice hotfix: env log, `MINIAPP_MAINTENANCE`, sync 423 crm, invoice `telegram.me`→`t.me` | `muru-backend-local` / `master` (`1392cb7`) | verified | **deployed** (Василий, 2026-07-14) | `git pull` + `deploy.sh`; без миграции | Prod smoke: mini app invoice OK |
| DEP-018 | **Cutover:** `CATALOG_SOURCE=crm`, final sync 264, backup, smoke | `muru-backend-local` / `master` (`1392cb7`) | verified | **deployed** (Василий, 2026-07-14) | env flip + `pm2 reload`; backup `/root/backups/muru_db_pre_cutover_2026-07-14_0843.dump` | Смоук §3; follow-up hotfix → DEP-019 |
| DEP-019 | Hotfix O-1+O-3: maintenance unless no initData (web OK); admin create product | `muru-backend-local` / `master` (`23edcaa`) | verified | **deployed** (Василий, 2026-07-14) | `deploy.sh` OK; `/api/health` 200; `/api/catalog/tree` без initData → 200 | vitest 62/321 |
| DEP-020 | Hotfix O-2+O-4: storefront `/img/` proxy + category covers | `muru-storefront` / `main` (`04d5222`) | verified | **deployed** (Василий, 2026-07-14) | recovery `npm ci --include=dev` → build → restart | curl `/catalog/`: 8× cover via `murushop.ru/img/`; smoke O-4/6 ✅ |
| DEP-021 | Hotfix O-5: admin CRM photo preview on re-edit | `muru-backend-local` / `master` (`cf5c0a3`) | verified | **deployed** (Василий, 2026-07-14) | `deploy.sh` OK; admin rebuild | `productToImageSlots` → `/img/crm_*/600.webp` |
| DEP-022 | **Категории v2:** миграции `020`–`025` на prod-БД | `muru-backend-local` / `master` (`14f438b`) | verified | **deployed** (Василий, 2026-07-14) | — | Prod: 17/242 links, 13 junk del, 18 SKU из Sale; backup `...1619.dump` 107K |
| DEP-023 | **Категории v2:** backend + admin @ `14f438b` | `muru-backend-local` / `master` | verified | **deployed** (Василий, 2026-07-14) | `deploy.sh`; PM2 ↺139 | API smoke 5/5 + login 409 guard |
| DEP-024 | **Категории v2:** storefront `giftGuide` @ `58d43dd` | `muru-storefront` / `main` | verified | **deployed** (Василий, 2026-07-14) | FF `04d5222→58d43dd`; build 28/28 | `web.murushop.ru/gifts/` 200 |
| DEP-025 | **Follow-up (опц.):** admin cover preview + subcategory sort | `muru-backend-local` / `master` | not started | — | после прода или hotfix | Preview: `proxyPath` в `CategoriesPage`; sort: swap при equal `sort_order` |
| DEP-026 | **Cat v2 D+E:** миграция `026` + backend/admin + storefront E3 | `feature/*` → **merged** `master`/`main` | verified | **deployed** | psql 026; DEP-026 smoke | merged @ `0482f00`/`34b67bb` |
| DEP-027 | **Inspiration G1–G3:** migration `027` + cover/banner + popover | merged | verified | **deployed** | psql 027; browser smoke OK | manual cover+banner per lookbook |
| DEP-028 | **H1:** compact square lookbook index cards | `34b67bb` on `main` | verified | **deployed** | storefront rebuild | `aspect-square`, `max-w-[1140px]` |

**Как обновлять:** оркестратор добавляет строку при verify prod-затрагивающей задачи; после деплоя Василий сообщает → колонка VPS = `deployed`, строка переносится в «Сделано» или помечается ✅.

## Бэклог (не терять)
- Картинки плиток подкатегорий (пустые серые боксы) — фаза CMS; category-grid.tsx деградирует мягко. **+ cutover 2026-07-14:** топ-категории на `web.murushop.ru` — `adaptTree()` не мапит `coverImageUrl`; CRM-фото товаров на витрине — нужен `/img/` proxy как в mini app.
- **Admin create product:** ~~route `products/new` bug~~ ✅ fix verified (`isNew = !id || id === 'new'`), DEP-019 deployed.
- **Категории v2 admin follow-up:** preview обложек категорий/подкатегорий в `CategoriesPage` (raw Drive/CRM URL → `/img/`); ↑↓ reorder не меняет порядок, если все `sort_order=0` после бэкфилла (DEP-025).
- **Пустые подкатегории в публичном дереве:** принято на staging-smoke — **оставляем** (решение A); фильтр пустых — отдельная правка по запросу.
- ~~Схлопнуть две папки бэкенда в один git-репо с ветками (убрать merge-боль на cutover).~~ ✅ ЗАКРЫТО 2026-07-06 (унификация U1-U4, `MURU_miniAPP` заморожен `7877be1`, прод на каноне DEP-008).
- `src/lib/content/collections.ts`: `productSlugs` у всех коллекций сейчас `[]` (раньше брались из mock-товаров, разорвано при выносе в статичный модуль) — заполнить реальными SKU после согласования наполнения лендингов с заказчиком.
- CDEK debug-урок для памяти: ранний симптом «расчёт цены не приходит» (сессия 2026-07-02) свёлся к пустым CDEK-кредам в `.env` на момент теста — сам расчётный код был исправен с самого начала. Перед глубоким дебагом смотреть `.env` в первую очередь.
- Rate limiter in-memory per-process — при масштабировании вынести в Redis.
- `getProducts` витрины тянет весь каталог и фильтрует в Node — ок до ~1000 SKU, дальше серверные query-параметры.
- Конвенция: `schema.sql` держим синхронным с миграциями в обоих backend-репо.
- `sitemap.xml` при билде без API — отдельный fallback (follow-up из 2026-07-04-02; `catalog-menu` уже fixed).

## Крупное расширение скоупа
Заказчик прислал рецензию ТЗ + Арина 6 пунктов маркетинга (30.06.2026). Это **отдельный оплачиваемый этап, НЕ входит в 470k v1**. Детали и триаж — в `SPEC.md` → блок v0.2.

## Фаза: Admin Design System (редизайн CRM admin/)

**Старт:** 2026-07-15. **Репо:** `muru-backend-local`, ветка `feature/admin-design-system` от `master` (`0482f00`). **Скоуп:** только `admin/`; backend не трогаем. **Деплой:** нет до отдельного решения.

| Фаза | Статус |
|------|--------|
| UI-1 Foundation (tokens, ui kit, layout, login) | ✅ verified `946ff15` |
| UI-2 Catalog + data components | ✅ verified `f44f04c` |
| UI-2.1 Subcategory CSS + sr-only | ✅ verified `58dfb4f` |
| UI-3 Orders/Content/Inspiration/Dashboard | ✅ verified `d5a3814` |
| UI-3.1 Hotspot CSS + quiet drag toast | ✅ verified `539a075` |
| UI-4 Polish + DESIGN.md | ✅ verified `bcc4be7` |

**ИТОГ:** design system **COMPLETE** на ветке (6 коммитов от `946ff15`). Legacy `styles.css`/`pages.css` удалены; CSS chain `tokens→base→components→index` (1525 строк); `admin/DESIGN.md` + ссылка в `55-executor.mdc`. `lib/*-api.ts` без диффа (только `cn.ts`). Forward-port не нужен. Деплой — после merge.

## Лог сессий
- **2026-07-15 (сессия 25)**: **Admin Design System UI-4 verified — фаза COMPLETE.** `@ bcc4be7`: удалены `styles.css`/`pages.css` (−1075 строк net); живые правила в `components.css`; RichTextEditor → `SkeletonText`; ProductsListPage → `filters-panel`; WCAG fix `.orders-status-tab--active` (olive bg + white text); `prefers-reduced-motion` guard в base + tab underline + spinners/shimmer. Оркестратор: `tsc -b`+`build` OK; confirm/alert/prompt=0; hex только tokens; legacy classes=0; diff scope `admin/**`+`55-executor.mdc`. Manual smoke не прогонялся. Следующее: решение о merge → deploy.
- **2026-07-15 (сессия 24b)**: **Admin Design System UI-3.1 verified.** `@ 539a075`: удалены дубли hotspot/tiptap из pages.css (accent marker живёт в components.css); drag-save без success toast. `tsc`+`build` OK. Следующее: UI-4.
- **2026-07-15 (сессия 24)**: **Admin Design System UI-3 verified.** `@ d5a3814`: PromptProvider, orders/content/dashboard/hotspot/RichTextEditor; confirm/alert/prompt=0; file inputs only ui-kit; lib без диффа. Находки: `pages.css` переопределяет `.hotspot-marker` (accent→dark gray); drag-end toast шумный. Следующее: UI-3.1 → UI-4.
- **2026-07-15 (сессия 23b)**: **Admin Design System UI-2.1 verified.** `@ 58dfb4f`: BEM `catalog-subcategory-order__*` aligned TSX↔CSS; `.sr-only` in base.css. Diff: 2 CSS files only; `tsc`+`build` OK. Следующее: UI-3.
- **2026-07-15 (сессия 23)**: **Admin Design System UI-2 verified.** `@ f44f04c`: Table/Skeleton/EmptyState/PageHeader/Toast/Confirm/ImageUploader/FileDropzone; App providers; catalog+sections pages на новой системе; ProductEdit subcategory grouped Cards + ★ Star; API libs без диффа; confirm в catalog/sections=0. Оркестратор: `tsc`+`build` OK; находки (не блокеры): BEM mismatch ProductEdit subcategory order (`__row` vs `-row` в CSS), `.sr-only` не в base.css; Lookbooks delete без toast. Следующее: UI-3.
- **2026-07-15 (сессия 22)**: **Admin Design System UI-1 verified.** Ветка `feature/admin-design-system` @ `946ff15`: CSS split (`styles/tokens|base|components|pages`), ui kit (Button/Field/Input/Checkbox/Badge/Card/Tabs), ProtectedLayout+CatalogLayout+ContentLayout+LoginPage, `lucide-react`. Оркестратор: `tsc -b`+`build` OK, hex только в `tokens.css`, `lib/*-api.ts` без диффа (только новый `cn.ts`). Следующее: UI-2.
- **2026-07-15 (сессия 21)**: **FF-merge Cat v2 D+E в канон.** `feature/crm-categories-e`→`master` (`14f438b`→`0482f00`, push OK); `feature/inspiration-hotspots`→`main` (`58d43dd`→`34b67bb`, push OK). PROGRESS: D/E→Сделано, DEP-027/028 deployed. **VPS checkout (Василий):** backend FF `14f438b`→`0482f00` on `master`, storefront FF `58d43dd`→`34b67bb` on `main`; `banner_image` в `content_lookbooks` — есть; curl 4×200. **Pending:** mini app regression smoke. G1 backend `d5c9a3e` (migration 027, `banner_image`, vitest 361/361). G2 admin `0482f00` (Обложка/Баннер, галерея убрана, hotspots на banner). G3 storefront (uncommitted): portal popover flip, `banner??cover`, gallery removed, `typecheck`+`build` 28/28. F1 `3499101` pushed ранее. **Следующее:** commit+push G3 + backend G1/G2 → VPS DEP-027.
- **2026-07-15 (сессия 20)**: **DEP-026 deployed на prod.** Backend/admin `84a78d3` (миграция 026, backup 114K); storefront `d028b74` build 26/26, PM2 ↺40. Smoke: CRM+public hotspots API; live HTML hero+«Предметы на фото»+MU0150 на `web.murushop.ru/lookbooks/vaupaupau/`. Build-warn CMS 404 → static fallback (не регрессия). Следующее: merge в master/main, браузерный клик по маркеру, mini app regression.
- **2026-07-15 (сессия 19, продолжение)**: **Cat v2 E3 storefront verified.** `feature/inspiration-hotspots` @ `d028b74`: hero hotspots, popover, ProductGrid hydration, generateStaticParams API+fallback. Оркестратор: `tsc`+`build` 28/28. **ИТОГ Cat v2 D+E: dev-verified, not deployed.** Commits: `d1af333` D, `9aeeb5a` E1, `84a78d3` E2, `d028b74` E3. Следующее: совместный smoke → DEP-026.
- **2026-07-15 (сессия 19)**: **Cat v2 D+E1+E2 dev-verified.** D: admin «Каталог и разделы» (`d1af333`) — hub, CategoryDetail, GiftGuide, redirects. E1: migration 026 + hotspot API (`9aeeb5a`), vitest 355. E2: `HotspotEditor` (`84a78d3`), tsc OK. Ручной admin smoke отложен (нет 017+026 на `muru_local`). Follow-up E3: `discountPercent`/`oldPrice` в public hotspot product DTO; CRM hotspot N+1 `getProduct`. Следующее: E3 storefront.
- **2026-07-14 (сессия 18)**: **Prod-cutover «Категории v2» DEP-022…024 deployed.** Pre-check prod `muru_db`: web_sub=243/sub=0, 18 SKU в Sale, virtual Sale=20. Backup `muru_db_pre_categories_v2_2026-07-14_1619.dump` (107K). FF pull `cf5c0a3→14f438b`; миграции 020–025 (13 junk, 18 reassigned, 17/242 subcats); `deploy.sh` PM2 ↺139; API smoke green (Sale 20, tree 17, login 200, guard 409). Storefront FF `04d5222→58d43dd`, build 28/28, `/gifts/` 200. DEP-025 optional остаётся.
- **2026-07-14 (сессия 17)**: **Категории v2 staging-verified, STOP перед продом.**
- **2026-07-14 (сессия 16)**: **Push+merge выполнены оркестратором напрямую** (проверил доступ: GitHub reachable, SSH до staging VPS — `Permission denied`, значит staging только руками Василия). `git push` 3 веток `feature/crm-categories-*`, FF-merge `-c → master` (`cf5c0a3→14f438b`, 26 файлов, чисто), `git push origin master` + `git push origin main` (storefront). Гейт подтверждён: `origin/master`==`local master`==`14f438b`. Подготовлен runbook для Василия: миграции 020–025 на staging + деплой + смоук-чеклист (12 пунктов из промпта пользователя) — прод остаётся untouched, STOP-гейт после staging.
- **2026-07-14 (сессия 15)**: **Фаза C verified + принята.** Отчёт исполнителя явно оставил 3 ключевые ручные проверки невыполненными («осталось в UI») — оркестратор не поверил на слово, прогнал их сам напрямую через реальный CRM API на `muru_local` (сброс пароля owner, логин, PATCH с телом, идентичным реальной форме): консолидация `Страна→Страна производитель`, сохранность легаси-ключей specs (`Плотность ткани`), удаление `Тип` без подкатегории, `isGiftGuide` фильтр в обоих API, живой `/gifts/` через `next dev` — всё подтвердилось на 100%. `tsc`×3/`vitest` (345/345)/`next build` — чисто. Закоммитил `14f438b` (backend/admin, `feature/crm-categories-c`) и `58d43dd` (storefront, `main`, без push, с явным подтверждением пользователя перед коммитом в чужой main). **Триптих A/B/C завершён.** Следующий шаг — решить порядок merge веток `feature/crm-categories-*` в `master` и план деплоя (staging сначала, миграции 020–025).
- **2026-07-14 (сессия 14)**: **Фаза B verified + исправлена.** Промпт фазы B выполнен в `feature/crm-categories-b`. Оркестратор нашёл критичную проблему бэкфилла (миграция 024 источником указывала пустое `web_subcategory_name`, реальные данные — в легаси `subcategory`, 243/265 товаров) прямым SQL-запросом к `muru_local` — согласовал с Василием фикс на `COALESCE`, переписал миграцию 024 сам, применил к `muru_local` (16 подкатегорий/243 связки), подтвердил идемпотентность и живым curl `/api/catalog/tree?subcategories=1`. Полная верификация диффов (19 файлов)/`tsc`×2/`vitest` (341/341). Закоммитил (`f9e2327`, без push/merge). Следующий шаг — промпт фазы C (характеристики + «Гид по подаркам»).
- **2026-07-14 (сессия 13)**: Старт фазы «Категории v2» (A/B/C, реструктуризация категорий/подкатегорий CRM). Промпт фазы A выдан и выполнен в `muru-backend-local` (`feature/crm-categories-a`). Оркестратор независимо верифицировал: диффы, `tsc` x2, `vitest` 332/332 (изолировано от сетевого шума сэндбокса), реальные SQL-запросы к `muru_local`, живые curl к `/api/catalog/tree`/`products`. Добавил и согласовал с Василием 2 доп. гварда (защита виртуальной «Распродажа» от удаления/переименования через CRM), не входивших в исходное ТЗ. Закоммитил фазу A (`dc04b70`, без push/merge). Следующий шаг — промпт фазы B (подкатегории-сущности).
- **2026-07-14 (сессия 9)**: **Hotfix verified (2026-07-14-03 + storefront O-2/O-4).** Backend: O-3 `isNew` fix; O-1 `miniappMaintenanceUnlessNoTelegramInitData` on catalog/cdek; vitest **62/321** (оркестратор). Storefront: `resolveCatalogImageUrl`, `adaptTree` coverImageUrl; tsc OK. Commit/deploy pending → DEP-019, DEP-020.
- **2026-07-14 (сессия 10)**: **Deploy verify.** DEP-019 **deployed** @ `23edcaa`. DEP-020 **failed** build → 502; fix `npm ci --include=dev` (DEPLOY.md).
- **2026-07-14 (сессия 11)**: **Storefront recovery + post-deploy smoke.** DEP-020 **deployed** after recovery; `web.murushop.ru` 200. Smoke: CRM фото/описание на web ✅; create product ✅; O-4/6 covers ✅; `MINIAPP_MAINTENANCE=true` ещё не снят.
- **2026-07-14 (сессия 12)**: **O-5 verified + deployed (DEP-021).** `cf5c0a3` — `admin/src/lib/images.ts`, `productToImageSlots`; `deploy.sh` OK.
- **2026-07-14 (сессия 8)**: **CUTOVER ВЫПОЛНЕН (DEP-018).** Финальный sync #54 (264 SKU); backup 177K; `CATALOG_SOURCE=crm` + reload; смоук: invoice ✅, витрина ✅, CRM edit ✅, `/img/crm_*` 200, archive/unarchive ✅. П.8 TG sync 423 — pending confirm.
- **2026-07-14 (сессия 7)**: **TD-003 closed.** `/var/www/muru/.env`: `CDEK_SENDER_ADDRESS` в двойных кавычках; `source .env` + `echo` — адрес со скобками без literal `"`; `pm2 reload ecosystem.config.js --update-env` OK.
- **2026-07-14 (сессия 6)**: **TD-002 closed.** Staging `/var/www/muru-staging` `1e439bf` → `1392cb7` (`reset --hard origin/master`, `deploy-staging.sh`); backend+admin build OK; `api-staging.murushop.ru/api/health` 200 db connected; prod PM2 ↺131 без изменений. Оркестратор: независимый curl health 200.
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
- **2026-07-14 (сессия 5)**: **DEP-017 deployed @ `1392cb7`.** Invoice normalize hotfix: `telegram.me/$` → `t.me/$` для `WebApp.openInvoice`. Prod smoke: оплата mini app OK. Cutover prep live.
- **2026-07-14 (сессия 4)**: Invoice hotfix chain: `da280b2` guard (t.me only) → симптом invalid URL; `a98222e` accept telegram.me → `WebAppInvoiceUrlInvalid`; `1392cb7` normalize → оплата OK.
- **2026-07-14 (сессия 3)**: **Cutover prep verified + merged.** `master` @ `da280b2` (4 коммита), push OK; vitest 62/311; DEP-017 pending deploy. `muru-docs` `5cbd631` (CUTOVER.md env consolidation).
- **2026-07-14 (сессия 2)**: Cutover prep — ветка `fix/env-maintenance-cutover`: env observability, `MINIAPP_MAINTENANCE`, invoice URL guard, admin sync 423 в crm, тесты 62/311, `CUTOVER.md` обновлён. Merge/deploy — pending.
- **2026-07-14**: **Фаза D закрыта.** D8.1 FF-merge в `master` (`1e439bf`), push OK, деплой не требуется (только тесты). TD-005 closed. Следующий шаг — по промпту Василия.
- **2026-07-13 (сессия 15)**: **D8.1 verified** (`admin-phase-d81`): mock `env.isCatalogCrmMode` в sync-scheduler.test.ts + тест crm skip. vitest 299/299 green (default + `CATALOG_SOURCE=crm`). TD-005 closed.
- **2026-07-13 (сессия 14)**: **DEP-016 deployed.** D8 @ `0877d6d`: migration 019 + nginx `/img/`; smoke `/img/` → 200. TD-001, TD-004 closed.
- **2026-07-13 (сессия 12)**: **DEP-015 deployed.** D7.1 hotfix `2d20658` на прод. Smoke cat#821: 49/1/2.
- **2026-07-13 (сессия 11)**: **D7.1 verified** (`admin-phase-d71`): `COUNT(DISTINCT p.id)` + test assert. vitest 59/298.
- **2026-07-13 (сессия 10)**: **DEP-014 deployed.** Прод `/var/www/muru` @ `8fa0f38` (D7: categories web tree + safe delete). `git pull` + `deploy.sh` OK.
- **2026-07-13 (сессия 9)**: **D7 verified** (промпт `2026-07-13-02`): `listCrmCategories` 2 SQL + merge; delete cross/FK guards; admin tree UI. vitest 59/298; admin build OK.
