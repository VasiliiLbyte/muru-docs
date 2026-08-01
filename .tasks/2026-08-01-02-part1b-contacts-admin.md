## MURU Executor — [ID: 2026-08-01-02]

**Репозиторий:** muru-backend-local  
**Путь:** `/Users/vasilii/Desktop/code /muru-backend-local`  
**Ветка:** продолжать `feat/settings-site-contacts` (1A уже на ней, uncommitted/или закоммить 1A отдельно — на усмотрение; не мешать с Drive-скриптами `backend/scripts/scan-drive-*`).  
**НЕ** мержить в master. **НЕ** деплоить.  
**Связь:** EPIC Settings → Часть **1B Admin SPA**. 1A **ACCEPT** оркестратором (миграция 035, CRM/public API готовы).

### Цель
Страница «Контакты» в CRM-админке: load → edit → save через `/api/crm/settings/*`. Реквизиты и SF — не делать.

### Контекст API (уже есть на ветке)
- `GET /api/crm/settings/site` → полный DTO (camelCase), owner-only
- `PUT /api/crm/settings/site/contacts` → body contact+social поля; **полная замена группы контактов** (опущенные поля → null). Форма обязана слать **все** поля после GET, не частичный PATCH.
- Поля контактов: `contactPhoneDisplay`, `contactPhoneHref`, `contactEmail`, `contactAddress`, `contactHours`, `contactMapLat`, `contactMapLng`, `contactMapZoom`, `socialTelegram`, `socialWhatsapp`, `socialVk`
- Реквизиты `req*` на GET видны, но на этой странице **не редактировать** и **не слать** в PUT contacts (PUT contacts их не трогает на бэке — ок).
- Auth: cookie `admin_token`, весь `/settings/*` уже под `<RequireOwner />`.

### Файлы (ожидаемые)
- `admin/src/types/settings.ts` — типы DTO / contact input
- `admin/src/lib/settings-api.ts` — `getSiteSettings()`, `updateContactSettings(input)` через `apiFetch` на `/api/crm/settings/...` (паттерн `users-api.ts` / `content-api.ts`)
- `admin/src/pages/settings/ContactsSettingsPage.tsx`
- `admin/src/pages/settings/SettingsHubPage.tsx` — карточку «Реквизиты и контакты» → `<Link to="/settings/contacts">`, убрать из `soonItems`
- `admin/src/App.tsx` — `<Route path="contacts" element={<ContactsSettingsPage />} />` внутри `settings`

### НЕ трогать
- `backend/` (кроме случая если нужен крошечный фикс — лучше не надо)
- Реквизиты UI, CDEK/YK, документы
- Mini-App `frontend/`
- Drive-скрипты в `backend/scripts/`
- Дизайн-систему UI-kit (использовать существующие Card, Field, Input, Button, PageHeader, useToast)

### UI требования
- Паттерн страницы: как `FixedPageEditPage` / `UsersSettingsPage` — PageHeader «Контакты», hint: телефон/адрес для шапки, футера и `/company/contacts/` на витрине (не путать с CMS-страницей «Контакты» в Content — там HTML-тело).
- Секции формы (логично сгруппировать):
  1. Телефон (display + href), email, адрес, режим работы
  2. Карта: lat, lng, zoom (number inputs)
  3. Соцсети: telegram, whatsapp, vk (пустое = не показывать на витрине позже)
- Load при монтировании → заполнить state из GET.
- Save: собрать **полный** contact payload (все ключи; пустые строки можно слать — бэк нормализует в null) → PUT → toast «Сохранено» / русская ошибка из `ApiError.message`.
- Loading skeleton / disabled Save while saving.
- Не добавлять поля реквизитов.

### Критерии готовности
- [ ] Owner открывает `/settings/contacts`, видит форму, сохраняет против локального BE (`:4000` + admin Vite)
- [ ] Manager / неавторизованный не видит страницу (уже RequireOwner + 403 на API)
- [ ] Хаб: «Реквизиты и контакты» — активная ссылка без бейджа «скоро»
- [ ] `cd admin && npx tsc -b` (или принятый в репо скрипт) зелёный; lint без новых ошибок
- [ ] Реквизиты / SF / backend не изменены без нужды

### Проверка
```bash
cd "/Users/vasilii/Desktop/code /muru-backend-local/admin"
npx tsc -b
# Ручной: backend :4000 (с кодом 1A) + admin login owner → Настройки → Контакты → save → reload → значения на месте
```

### Отчёт оркестратору
- Список файлов, ветка, коммит если был
- Результат tsc
- Ручной smoke (да/нет)
- Подтверждение: форма шлёт полный contact payload
- Риски

**После ACCEPT 1B** оркестратор выдаст 1C (storefront).
