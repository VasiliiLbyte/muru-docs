# EPIC — Раздел «Настройки»: СДЭК · ЮKassa · Документы · Контакты

**Target repos:** `muru-backend-local` (backend + admin SPA), `muru-storefront` (SF)
**Base:** BE `master @ 8e146b9`. SF `main @ 459eb90` (merge gate ЗАКРЫТ — M7+M8+M8-9+kitchen+collections+video влиты в main через FF, local==origin).
**Автор ТЗ:** оркестратор-надзор (Vasilii). **Исполнитель:** Cursor (working orchestrator).
**Режим:** Plan mode → реализация по частям → возврат handoff-отчёта по каждой части. НЕ мерджить в master/main. НЕ деплоить без явного гейта.

---

## 0. Контекст и НЕ-цели (прочитать до старта)

Реализуем то, что уже намечено заглушками в `admin/src/pages/settings/SettingsHubPage.tsx` (карточки «скоро»): Реквизиты и контакты, Доставка (СДЭК), Оплата (ЮКасса/54-ФЗ), Юридические документы.

**Целевая админка — ТОЛЬКО браузерная SPA `muru-backend-local/admin/`** (react-router, cookie-auth `/api/crm/*`, tiptap `RichTextEditor`). Mini-App-админка (`muru-backend-local/frontend/src/admin/*`) в этом эпике НЕ трогается — она выводится из эксплуатации отдельным треком.

**Ключевое решение по секретам (вариант A):** платёжные и API-секреты остаются в env и НЕ переносятся в БД. В БД идут только несекретные бизнес-параметры. Полный список — в каждой части ниже. Причина: security-раздел (W-SEC) ещё не закрыт, канал утечки платёжных ключей через админку недопустим; плюс защита от случайного затирания ключа менеджером.

**НЕ-цели:**
- Не переносить `CDEK_CLIENT_SECRET/CLIENT_ID/WEBHOOK_SECRET`, `YOOKASSA_SECRET_KEY/WEB_SECRET_KEY`, `returnUrl` в БД. Никогда не отдавать их значения в API-ответах.
- Не менять вёрстку/дизайн существующих страниц SF — только источник данных (статика → API с fallback).
- Не расширять видео-баннеры, не трогать Phase W / auth, не трогать каталог/заказы.
- Не вводить новые тяжёлые зависимости — tiptap/prosemirror и весь UI-kit уже есть.

**Общие инварианты проекта (соблюдать):**
- Mini App API contract — additive-only. Публичные `/api/content/*` эндпойнты, которые уже потребляет Mini App/SF, менять только обратно совместимо.
- Все новые CRM-роуты — под тем же cookie-auth middleware, что и остальные `/api/crm/*` (`requireCrmAuth`), а платёжные/доставочные вкладки — дополнительно за owner-ролью (в SPA — `<RequireOwner>`, на бэке — эквивалент, см. как сделано для `/api/crm/users`).
- Каждая новая логика покрывается тестами (BE — рядом с сервисом; фронт — по существующему паттерну). Требуемые тесты должны падать ДО реализации и проходить ПОСЛЕ.
- Русские тексты ошибок для admin-facing (как в DEP-053).

---

## STOP-0 — MERGE GATE SF (ЗАКРЫТ ✅)

`feat/banner-video-sf @ 459eb90` (M7+M8+M8-9+kitchen+collections+video) влит в `main` через FF. `main = 459eb90`, local==origin, verified. Все SF-подчасти базируются от `main @ 459eb90`. BE-части (1, 2, 4-backend) — от `master @ 8e146b9`. Блокер снят, гейт пройден.

**Cursor:** каждую SF-подчасть вести на своей feature-ветке от `main @ 459eb90`. НЕ мерджить обратно в main без гейта Vasilii.

---

## Архитектура данных (общая для всех вкладок)

Две новых сущности в БД:

1. `site_settings` — singleton (`id=1`, паттерн как `bot_welcome_settings`). Плоская запись с namespace-полями. Хранит: контакты, реквизиты, CDEK-бизнес-параметры, YooKassa-бизнес-параметры. Одна строка, один `updated_at`.
2. Документы — переиспользуем СУЩЕСТВУЮЩУЮ `content_pages`. Никакой новой таблицы. Расширяем набор фиксированных slug'ов.

Разбивка полей `site_settings` по группам — в частях 1/3/4.

---

# ЧАСТЬ 1 — КОНТАКТЫ (BE + SF). Низкий риск, делаем первой.

## 1A. Backend (`muru-backend-local/backend`)

Миграция `src/db/migrations/035_site_settings.sql` (+ down-файл):
- Создать таблицу `site_settings` с `id INT PRIMARY KEY DEFAULT 1 CHECK (id = 1)`, `updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`, и колонками группы «контакты»:
  - `contact_phone_display TEXT`, `contact_phone_href TEXT`, `contact_email TEXT`, `contact_address TEXT`, `contact_hours TEXT`, `contact_map_lat DOUBLE PRECISION`, `contact_map_lng DOUBLE PRECISION`, `contact_map_zoom INT`
  - соцсети-заготовки (nullable, пустые не рендерятся на фронте): `social_telegram TEXT`, `social_whatsapp TEXT`, `social_vk TEXT`
  - реквизиты (для страницы /company/requisites, см. Часть 3): `req_full_name TEXT`, `req_short_name TEXT`, `req_inn TEXT`, `req_ogrnip TEXT`, `req_legal_address TEXT`, `req_actual_address TEXT`, `req_phone TEXT`, `req_email TEXT`, `req_site TEXT`, `req_bank_details TEXT`
- Вставить одну строку `id=1` со всеми NULL (или с нейтральными дефолтами из текущего `site.ts`, на усмотрение — но без реальных ПДн клиента).

Сервис `src/services/site-settings.service.ts` (+ тест):
- `SETTINGS_ID = 1`.
- `getSiteSettings()` — читает строку, маппит snake_case→camelCase, при отсутствии строки возвращает объект с null-полями.
- `updateContactSettings(input)` — zod-валидация (телефон/e-mail — мягкая, разрешить пустые; координаты — number|null), UPDATE только полей группы «контакты» + соцсети, `updated_at = NOW()`. Возвращает свежий `getSiteSettings()`.
- Реквизиты вынести в отдельный `updateRequisitesSettings(input)` (используется Частью 3), чтобы вкладки не затирали чужие поля.
- Публичный проекционный геттер `getPublicSiteContacts()` — отдаёт ТОЛЬКО контакты + соцсети + координаты (реквизиты уедут отдельным публичным эндпойнтом в Части 3, т.к. это отдельная страница).

Роутеры:
- CRM: новый `src/routes/crm-settings.routes.ts`, экспорт `crmSettingsRouter`, `use(requireCrmAuth())`. Эндпойнты:
  - `GET /site` → полный `getSiteSettings()` (для форм админки; owner-gated см. ниже)
  - `PUT /site/contacts` → `updateContactSettings`
  - (реквизиты и cdek/yookassa PUT'ы добавят части 3/4; закладывай файл расширяемым)
- Публичный: добавить в существующий `content-public.routes.ts` (или новый `site-public.routes.ts`, смонтированный под `/api/content`) эндпойнт `GET /site-contacts` → `getPublicSiteContacts()`. Additive, без auth, с тем же конвертом `ok()`.
- Монтирование в `src/index.ts`: `app.use('/api/crm/settings', crmSettingsRouter)` рядом со строкой ~92 (`/api/crm/users`). Публичный — рядом с остальными `/api/content/*`.

Owner-gating: вкладки настроек требуют owner-роль. Посмотри, как ограничен `/api/crm/users` (в SPA — `<RequireOwner>`; на бэке — соответствующий guard), и примени тот же механизм к `crm-settings.routes.ts`. По умолчанию все вкладки настроек — owner-only (безопаснее); допустимость доступа менеджера к контактам согласовать с Vasilii.

DoD 1A: миграция применяется и откатывается; 035 up/down чистые; `getSiteSettings`/`updateContactSettings`/`getPublicSiteContacts` покрыты тестами; `GET /api/content/site-contacts` возвращает 200 с публичной проекцией; секретов в ответе нет; `npx tsc --noEmit` зелёный; существующие тесты не падают.

## 1B. Admin SPA (`muru-backend-local/admin`)

- `src/lib/settings-api.ts`: `getSiteSettings()`, `updateContactSettings(input)` через `apiFetch` на `/api/crm/settings/*`. Типы — в `src/types/settings.ts`.
- Страница `src/pages/settings/ContactsSettingsPage.tsx` по образцу `FixedPageEditPage`/`UsersSettingsPage`: форма (телефон display + href, e-mail, адрес, режим работы, координаты+zoom, соцсети), UI-kit (`Card, Field, Input, Button, PageHeader, useToast`), load→edit→save, toast «Сохранено», русские ошибки.
- Роут в `App.tsx`: `<Route path="contacts" element={<ContactsSettingsPage />} />` внутри `settings` (под `<RequireOwner>`).
- `SettingsHubPage.tsx`: карточку «Реквизиты и контакты» превратить в активный `<Link to="/settings/contacts">` (убрать из `soonItems`). Реквизиты пока могут вести на ту же страницу отдельной секцией или на будущую (Часть 3) — согласуй в Plan mode.

DoD 1B: страница открывается только owner; сохранение работает против локального BE; tsc/lint зелёные.

## 1C. Storefront (`muru-storefront`) — ПОСЛЕ STOP-0

Цель — перевести `src/lib/site.ts → siteContacts` со статики на API, сохранив текущую вёрстку. Сейчас `siteContacts` импортируют 7 файлов: `layout/footer.tsx`, `layout/header.tsx`, `layout/mobile-menu.tsx`, `contacts/contacts-details.tsx`, `contacts/contact-map.tsx`, `seo/jsonld.ts`, и сам `site.ts`.

Подход (согласовать точную форму в Plan mode):
- Добавить в `src/lib/api/endpoints.ts` функцию `getSiteContacts()` по образцу `getStaticPage` — с `isContentBackendEnabled()`-гейтом и fallback на текущие статические значения при выключенном backend или ошибке (не ломать сборку/превью). Zod-схема ответа — в `src/lib/schemas/`.
- Текущий статический объект `siteContacts` СОХРАНИТЬ как `SITE_CONTACTS_FALLBACK` (дефолт), чтобы fallback был идентичен нынешнему поведению.
- Точки потребления — серверные компоненты (footer, header, contacts, jsonld) → сделать async и получать контакты через `getSiteContacts()` с ISR (`revalidate` ~300). `mobile-menu.tsx` — если клиентский, пробросить контакты пропсом из ближайшего server-компонента, НЕ тащить fetch в клиент.
- Проверить, что пустые соцсети НЕ рендерят пустые ссылки.

DoD 1C: визуально страницы идентичны (вёрстка не изменена); при включённом backend контакты приходят из API; при выключенном — из fallback; `next build` зелёный; storefront-тесты (29/29) не падают, при изменении контактных компонентов — обновить/добавить тесты.

### === STOP-1 === Вернуть handoff по Части 1 (BE+SPA+SF), дождаться ACCEPT Vasilii перед Частью 2.

---

# ЧАСТЬ 2 — ДОКУМЕНТЫ (BE + admin + SF). Переиспользуем `content_pages`.

## 2A. Backend

Расширить фиксированный набор slug'ов юридических документов. Сейчас `FIXED_PAGE_SLUGS` в `content.service.ts` = `['help','contacts','company','vacancy','partners']`, а на витрине уже рендерятся `privacy`, `offer`, `requisites` (см. SF `getStaticPage`). Инвентарь документов оригинала muru.ru:

| slug | Назначение | URL на SF |
|---|---|---|
| `privacy` | Политика в отношении ПДн (152-ФЗ) | `/legal/privacy/` (уже есть) |
| `offer` | Публичная оферта / условия обслуживания | `/legal/offer/` (уже есть) |
| `delivery` | Доставка | `/help/delivery/` (создать) |
| `refund` | Возврат | `/help/refund/` (создать) |
| `terms` | Условия обслуживания (если отделяем от оферты) | `/help/terms/` (создать) |
| `consent` | Согласие на обработку ПДн (чекбокс регистрации/checkout) | `/legal/consent/` (создать) |
| `requisites` | Реквизиты — НЕ через этот механизм, см. Часть 3 | — |

- Добавить недостающие slug'и (`privacy, offer, delivery, refund, terms, consent`) в фиксированный набор документов. Реши в Plan mode: расширять `FIXED_PAGE_SLUGS` напрямую или завести отдельный `LEGAL_DOC_SLUGS` с той же upsert-механикой (`upsertFixedPage`). Предпочтителен отдельный набор `LEGAL_DOC_SLUGS`, использующий тот же код upsert/get, но с собственными дефолтными заголовками.
- Публичный `getPublicPageBySlug` уже отдаёт любую `is_visible` страницу по slug — новые документы поедут через него без изменений (additive). Проверить, что slug-роутинг не конфликтует.
- Тесты: upsert+get для новых slug'ов, 404 на невидимую страницу, санитайз HTML сохраняется.

DoD 2A: новые slug'и валидны в CRM upsert; публичный geter отдаёт их; sanitize работает; tsc + тесты зелёные.

## 2B. Admin SPA — вкладка «Документы»

- Расширить `FixedPageSlug` / `SECTION_META` в `admin/src/types/content.ts` и `FixedPageEditPage.tsx` новыми документами (title/hint/defaultTitle для каждого). Тексты hint — куда попадает на витрине.
- Страница-хаб `src/pages/settings/DocumentsSettingsPage.tsx` (или под `/content/legal`) со списком документов-ссылок, каждая ведёт на `FixedPageEditPage section="<slug>"` (rich-text редактор + видимость + SEO уже готовы в шаблоне). Именно то, что просил Vasilii: «в каждый текст просто вставлять».
- Роуты в `App.tsx`: по образцу `content/help`, `content/contacts` добавить маршруты для каждого документа.
- `SettingsHubPage.tsx`: карточку «Юридические документы» сделать активным `<Link>`, убрать из `soonItems`.

DoD 2B: каждый документ открывается в редакторе, сохраняется, тумблер видимости работает; owner-gated; tsc/lint зелёные.

## 2C. Storefront — ПОСЛЕ STOP-0

- Создать недостающие страницы-роуты по образцу `app/legal/privacy/page.tsx` (тянет `getStaticPage("privacy")` через `ContentShell`+`StaticProse`): `app/help/delivery/page.tsx`, `app/help/refund/page.tsx`, `app/help/terms/page.tsx`, `app/legal/consent/page.tsx`. Каждая — `getStaticPage("<slug>")`, `generateMetadata`, `revalidate=300`, хлебные крошки по образцу.
- Обновить fallback-плейсхолдеры в `src/lib/content/pages.ts` (`DEFS`): добавить записи для новых slug'ов (нейтральные lorem, как для privacy/offer) — чтобы при выключенном backend страницы не падали 404.
- Ссылки: в `site.ts` `legalNav` уже содержит privacy+offer; при необходимости добавить новые документы в футер/страницу «Клиентам» (`/help`) — согласовать в Plan mode, не плодить лишние ссылки без указания Vasilii.
- SEO/URL при переезде на muru.ru: сверить целевые пути с `muru-docs/PHASE_URL.md` / `URL_MIGRATION_AUDIT.md`. Если оригинал muru.ru отдаёт документы по иным URL (напр. `/help/delivery/`), сохранить эти пути ради 301-парити. Не изобретать новые URL-структуры без сверки с фазой URL.

DoD 2C: новые страницы рендерятся из CMS при включённом backend и из fallback при выключенном; `next build` зелёный; тесты не падают.

### === STOP-2 === Handoff по Части 2, дождаться ACCEPT.

---

# ЧАСТЬ 3 — РЕКВИЗИТЫ (BE + admin + SF). Небольшая, примыкает к контактам.

Реквизиты — структурированные поля (не свободный HTML), потому идут в `site_settings` (колонки `req_*` из миграции 035), а НЕ в `content_pages`. На витрине рендерятся таблицей (`requisites-table.tsx`), сейчас — плейсхолдеры.

## 3A. Backend
- `updateRequisitesSettings(input)` в `site-settings.service.ts` — UPDATE только `req_*` полей (+тест).
- CRM: `PUT /api/crm/settings/requisites`.
- Публичный: `GET /api/content/requisites` → проекция только `req_*` полей.

## 3B. Admin SPA
- В `settings-api.ts` — `updateRequisites`. Страница `RequisitesSettingsPage.tsx` (или секция внутри `ContactsSettingsPage`) с полями реквизитов. Owner-gated.

## 3C. Storefront (после STOP-0)
- `requisites-table.tsx`: заменить хардкод-массив `REQUISITES` на данные из `getRequisites()` (по образцу site-contacts, с fallback на текущие плейсхолдеры). Страница `app/company/requisites/page.tsx` уже существует — только источник таблицы.

DoD 3: реквизиты редактируются в админке и отображаются на `/company/requisites/`; без реальных ПДн в дефолтах/фикстурах; сборки зелёные.

### === STOP-3 === Handoff, ACCEPT.

---

# ЧАСТЬ 4 — СДЭК и ЮKassa (BE + admin). Высокий риск (платежи/доставка). Строго staging-first.

Секреты (вариант A) — НЕ в БД, остаются в env: `CDEK_CLIENT_ID`, `CDEK_CLIENT_SECRET`, `CDEK_WEBHOOK_SECRET`, `YOOKASSA_SECRET_KEY`, `YOOKASSA_WEB_SECRET_KEY`, `YOOKASSA_RETURN_URL`, `YOOKASSA_WEB_RETURN_URL`. Их значения НИКОГДА не отдаются в API и не пишутся в БД.

В БД (`site_settings`, новые колонки — миграция `036_settings_cdek_yookassa.sql`):
- CDEK: `cdek_env TEXT` ('test'|'production'), `cdek_sender_city_code INT`, `cdek_sender_postal_code TEXT`, `cdek_sender_address TEXT`, `cdek_sender_name TEXT`, `cdek_sender_phone TEXT`, `cdek_tariff_door INT`, `cdek_tariff_pvz INT`, `cdek_default_weight_grams INT`, `cdek_default_length_cm INT`, `cdek_default_width_cm INT`, `cdek_default_height_cm INT`
- YooKassa: `yookassa_vat_code INT`, `yookassa_verify_ip BOOLEAN`. (shopId и web_shopId — НЕ секрет; хранить в БД для сверки ИЛИ отдавать read-only из env. Реши в Plan mode; проще — read-only из env, без записи.)

Слияние БД-конфига с env (критично): сейчас `env.ts` — единственный источник для `cdek/client.ts`, `cdek/orders.service.ts`, `yookassa/client.ts`, `yookassa/payments.service.ts`, а дефолтные габариты — в `constants/product-shipping-defaults.ts`. Ввести резолвер (напр. `src/services/runtime-config.service.ts`), который отдаёт эффективную конфигурацию как `БД-значение ?? env-дефолт` для несекретных полей, а секреты берёт всегда из env. Точки чтения перевести на резолвер минимально-инвазивно. НЕ ломать guard'ы в `env.ts` (production requires YOOKASSA_SECRET_KEY и т.п.) — секреты и их проверки остаются как есть.

Важные детали:
- У YooKassa ДВА набора кредов: Mini App (`YOOKASSA_SHOP_ID`) и web-storefront (`YOOKASSA_WEB_SHOP_ID`). Вкладка должна отражать оба контура (хотя бы read-only статус «подключено»).
- `CDEK_ENV=production` в env требует client_id/secret. Если клиент через вкладку переключает `cdek_env` в 'production', а секретов в env нет — не падать процессом (в отличие от env-guard на старте), а вернуть управляемую ошибку/предупреждение в UI: «для production нужны серверные ключи CDEK».
- Кэш конфигурации: если клиенты CDEK/YooKassa кэшируют конфиг в модульных переменных при старте — предусмотреть чтение из резолвера на каждый запрос ИЛИ инвалидацию кэша после сохранения настроек. Проверить в Plan mode.

## 4A. Backend
- Миграция 036 (+down). `updateCdekSettings`, `updateYookassaSettings` в сервисе (+тесты). Резолвер эффективной конфигурации (+тест на приоритет БД над env и на то, что секреты только из env).
- CRM (owner-gated): `PUT /api/crm/settings/cdek`, `PUT /api/crm/settings/yookassa`, и `GET /api/crm/settings/integrations-status` → для каждого секрета возвращать только булев флаг «настроен на сервере» (`cdekConfigured`, `yookassaConfigured`, `yookassaWebConfigured`), НЕ значения.
- Тест-гарантия: ни один settings-эндпойнт не сериализует значения секретов.

## 4B. Admin SPA — вкладки «Доставка (СДЭК)» и «Оплата (ЮКасса/54-ФЗ)»
- `CdekSettingsPage.tsx`, `YookassaSettingsPage.tsx`. Секретные поля показаны как статус-бейдж «✓ настроено на сервере» / «✗ не настроено» (из `integrations-status`), НЕ инпуты. Редактируемые — несекретные поля из БД. Owner-gated (`<RequireOwner>`).
- `SettingsHubPage.tsx`: карточки «Доставка (СДЭК)» и «Оплата (ЮКасса / 54-ФЗ)» → активные `<Link>`, убрать из `soonItems`.

DoD 4: несекретные параметры CDEK/YooKassa редактируются и применяются на бэке через резолвер; секреты не утекают ни в одном ответе (доказать тестом); переключение cdek_env в production без ключей даёт управляемую ошибку, не крэш; tsc/тесты зелёные.

### === STOP-4 === Handoff. Развёртывание Части 4 — ТОЛЬКО через staging-first по чеклисту Vasilii (миграция 036 на staging → smoke calc CDEK + тестовый платёж YooKassa → затем прод). Live-подтверждение, что расчёт доставки и оплата не сломаны, до любого merge в master.

---

## Порядок и зависимости
1. Часть 1 BE+SPA → STOP-1 (SF-часть 1 — после STOP-0 merge gate).
2. Часть 2 → STOP-2.
3. Часть 3 → STOP-3.
4. Часть 4 → STOP-4 (staging-first обязателен).

BE-части 1/2/4-backend можно начинать сразу от `master @ 8e146b9`. Все SF-подчасти — только после STOP-0.

## Что вернуть в каждом handoff-отчёте
- Затронутые файлы (по репо), новые миграции (up/down применены/откачены — подтвердить).
- Список новых/изменённых тестов + факт «падали до, проходят после».
- Результат `tsc --noEmit` и линтера по каждому репо.
- Явное подтверждение: секреты не сериализуются ни в одном новом эндпойнте (для Части 4).
- Точки, требующие решения Vasilii (owner-gating контактов, отдельный LEGAL_DOC_SLUGS vs расширение FIXED, URL документов по PHASE_URL).
- Ветки и SHA каждого коммита. НЕ мерджить, НЕ деплоить без гейта.

## Документация (обновить в muru-docs при закрытии эпика)
- `DECISIONS.md`: ADR о разделении секретов (вариант A) и о переиспользовании `content_pages` для юр-документов.
- `BACKLOG.md`: завести и закрыть эпик.
- `PROGRESS.md`: DEP-записи по деплоям (Часть 4 — с пометкой staging-first).
- `API_CONTRACT.md`: новые публичные `/api/content/site-contacts`, `/api/content/requisites` и новые документные slug'и (additive).
