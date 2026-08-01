## MURU Executor — [ID: 2026-08-01-08]

**Репозиторий:** muru-backend-local  
**Путь:** `/Users/vasilii/Desktop/code /muru-backend-local`  
**Ветка:** `feat/settings-site-contacts`. НЕ merge/deploy. Backend 3A **ACCEPT**.  
**Связь:** EPIC Settings → Часть **3B Admin**.

### Цель
Страница «Реквизиты» в settings: load → edit → save. Отдельная от Контактов.

### API (готово)
- `GET /api/crm/settings/site` — полный DTO (включая req*)
- `PUT /api/crm/settings/site/requisites` — **full replace** группы req (как contacts): форма шлёт все 10 полей
- Поля: `reqFullName`, `reqShortName`, `reqInn`, `reqOgrnip`, `reqLegalAddress`, `reqActualAddress`, `reqPhone`, `reqEmail`, `reqSite`, `reqBankDetails`

### Сделать
- `admin/src/types/settings.ts` — тип `RequisitesSettingsInput` (если ещё нет)
- `admin/src/lib/settings-api.ts` — `updateRequisitesSettings(body)`
- `admin/src/pages/settings/RequisitesSettingsPage.tsx` — паттерн ContactsSettingsPage (`buildRequisitesPayload` со всеми полями; toast «Сохранено»)
  - Labels RU: Полное/сокращённое наименование, ИНН, ОГРНИП, юр./факт. адрес, телефон, email, сайт, банковские реквизиты (textarea для bank ok если в UI-kit есть; иначе Input)
- `App.tsx`: `settings/requisites` → RequisitesSettingsPage (под RequireOwner)
- `SettingsHubPage.tsx`: soon «Реквизиты» → `<Link to="/settings/requisites">`

### НЕ трогать
- backend/, ContactsSettingsPage (кроме случайного shared helper — не обязательно)
- Documents / SF
- Drive-скрипты

### DoD
- [ ] Owner: хаб → Реквизиты → save полного payload
- [ ] `npx tsc -b` зелёный
- [ ] Контакты не затираются при save реквизитов (бэк уже разделяет UPDATE)

### Отчёт
файлы, tsc, подтверждение полного payload (10 полей)
