## MURU Executor — [ID: 2026-08-01-01]

**Репозиторий:** muru-backend-local  
**Путь:** `/Users/vasilii/Desktop/code /muru-backend-local`  
**Ветка:** feature от `master` (напр. `feat/settings-site-contacts`). НЕ мержить в master. НЕ деплоить.  
**База:** `master` (ожидаемый tip около `8e146b9`; если tip ушёл вперёд — от текущего `origin/master`).  
**Связь:** muru-docs `.tasks/EPIC-settings-cdek-yookassa-docs-contacts.md` → Часть 1A; план Settings epic.

### Цель
Добавить singleton `site_settings` + CRM/public API для контактов магазина (секреты в БД не переносить). Только backend.

### Контекст (решения уже зафиксированы оркестратором)
- Паттерн singleton: `sync_schedule_settings` / `bot_welcome_settings` (`id INTEGER PRIMARY KEY CHECK (id = 1)`).
- Auth как у `/api/crm/users`: `crmUsersRouter.use(requireCrmAuth()); use(requireOwner)`.
- Публичный конверт: `ok(res, data)` в `content-public` под `/api/content`.
- Последняя миграция: `034_banner_video.sql` → следующая **`035_site_settings`**.
- Колонки `req_*` создать сразу в 035 (UI/PUT реквизитов — Часть 3; здесь достаточно заготовки `updateRequisitesSettings` + колонок).
- Owner-only для всего `/api/crm/settings`.
- Mini-App admin (`frontend/src/admin`) НЕ трогать.
- Admin SPA и storefront — НЕ в этом промпте (1B/1C отдельно).

### Файлы (ожидаемые)
- `backend/src/db/migrations/035_site_settings.sql`
- `backend/src/db/migrations/035_site_settings.down.sql` (`DROP TABLE IF EXISTS site_settings;`)
- `backend/src/db/schema.sql` — добавить определение `site_settings` рядом с другими singleton-таблицами (как `sync_schedule_settings`)
- `backend/src/services/site-settings.service.ts`
- `backend/src/services/site-settings.service.test.ts`
- `backend/src/controllers/crm-settings.controller.ts` (или handlers в том же стиле, что crm-users)
- `backend/src/routes/crm-settings.routes.ts`
- `backend/src/controllers/content-public.controller.ts` — handler `getSiteContacts`
- `backend/src/routes/content-public.routes.ts` — `GET /site-contacts`
- `backend/src/index.ts` — `app.use('/api/crm/settings', crmSettingsRouter)` рядом с `/api/crm/users`

### НЕ трогать
- `frontend/` (Mini App), `admin/` (SPA — это 1B)
- CDEK / YooKassa / payments / env secrets
- `content.service.ts` FIXED_PAGE_SLUGS (Часть 2)
- Каталог, заказы, auth Phase W

### Схема `site_settings` (миграция 035)

```sql
CREATE TABLE IF NOT EXISTS site_settings (
  id INTEGER PRIMARY KEY DEFAULT 1 CHECK (id = 1),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  -- contacts
  contact_phone_display TEXT,
  contact_phone_href TEXT,
  contact_email TEXT,
  contact_address TEXT,
  contact_hours TEXT,
  contact_map_lat DOUBLE PRECISION,
  contact_map_lng DOUBLE PRECISION,
  contact_map_zoom INT,
  -- socials (nullable)
  social_telegram TEXT,
  social_whatsapp TEXT,
  social_vk TEXT,
  -- requisites (Part 3 UI; columns now)
  req_full_name TEXT,
  req_short_name TEXT,
  req_inn TEXT,
  req_ogrnip TEXT,
  req_legal_address TEXT,
  req_actual_address TEXT,
  req_phone TEXT,
  req_email TEXT,
  req_site TEXT,
  req_bank_details TEXT
);
INSERT INTO site_settings (id) VALUES (1) ON CONFLICT (id) DO NOTHING;
```

Дефолты: все NULL (без реальных ПДн клиента). Не копировать прод-телефон в миграцию.

### Сервис `site-settings.service.ts`
- `SETTINGS_ID = 1`
- `getSiteSettings()` — SELECT; нет строки → объект с null-полями (camelCase)
- `updateContactSettings(input)` — zod `.strict()`; мягкая валидация phone/email (пустые/`""` → null ок); lat/lng/zoom: `number | null`; UPDATE **только** contact_* + social_*; `updated_at = NOW()`; вернуть `getSiteSettings()`
- `updateRequisitesSettings(input)` — заготовка: UPDATE только `req_*` (+ zod). Можно не экспонировать роутом в 1A, но функция + unit-тест обязательны (Часть 3 подключит PUT)
- `getPublicSiteContacts()` — проекция **только**: phoneDisplay, phoneHref, email, address, hours, mapLat, mapLng, mapZoom, socialTelegram, socialWhatsapp, socialVk. **Без** `req_*`.

Маппинг snake→camel единообразно (как в других CRM DTO).

### Роуты CRM (`/api/crm/settings`)
```
use(requireCrmAuth())
use(requireOwner)
GET  /site           → getSiteSettings()
PUT  /site/contacts  → updateContactSettings (body zod; 422 + русские ошибки через fail/zodErrorMessage)
```
Файл роутера закладывать расширяемым (позже: requisites, cdek, yookassa).

### Публичный
`GET /api/content/site-contacts` → `getPublicSiteContacts()`, без auth, `ok()`.

### Тесты (должны падать до реализации / проходить после)
В `site-settings.service.test.ts` (мок `pool.query` по образцу `crm-users.service.test.ts`):
1. `getSiteSettings` — пустой rows → все null
2. `updateContactSettings` — UPDATE затрагивает только contact/social поля; возвращает свежие данные
3. `getPublicSiteContacts` — **не** содержит ключей `req*` / requisites
4. `updateRequisitesSettings` — не затирает contact-поля (отдельный UPDATE только req_*)

Опционально тонкий route-тест: без cookie → 401 на `GET /api/crm/settings/site` (если в репо уже есть паттерн route tests для crm-users; не раздувать).

### Критерии готовности
- [ ] Миграция 035 up применяется локально; down откатывает (`DROP TABLE`)
- [ ] `schema.sql` содержит `site_settings`
- [ ] `GET /api/content/site-contacts` → 200, только публичная проекция
- [ ] `GET /api/crm/settings/site` без auth → 401; с manager → 403; с owner → 200
- [ ] `PUT /api/crm/settings/site/contacts` обновляет контакты, не трогает `req_*`
- [ ] Секреты CDEK/YK нигде не появляются в новых ответах (их и нет в таблице — подтвердить явно в отчёте)
- [ ] `npx tsc --noEmit` в `backend/` зелёный
- [ ] Новые тесты зелёные; существующий vitest не сломан (минимум прогнать `site-settings` + smoke затронутых)

### Проверка
```bash
cd "/Users/vasilii/Desktop/code /muru-backend-local/backend"
# применить 035 к local DB (как принято в репо: psql -f или npm script)
npx tsc --noEmit
npx vitest run src/services/site-settings.service.test.ts
# smoke (сервер :4000):
curl -s http://localhost:4000/api/content/site-contacts | head
curl -s -o /dev/null -w "%{http_code}" http://localhost:4000/api/crm/settings/site   # ожидай 401
```

### Отчёт оркестратору
- Ветка + SHA коммита (если коммитил; коммит только если попросили / принято в репо)
- Список изменённых файлов
- Факт: 035 up/down прогнаны
- Список тестов + «падали до / проходят после» (или red→green в одной сессии)
- Вывод `tsc` и vitest
- Явно: публичный ответ без `req_*` и без любых secret-полей
- Риски / что не сделано (admin/SF = следующие промпты)

**Модель:** для миграций+API предпочтителен более сильный модель; Auto ок если Plan mode жёстко следует этому промпту.
