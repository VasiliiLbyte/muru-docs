## MURU Executor — [ID: 2026-08-01-07]

**Репозиторий:** muru-backend-local  
**Путь:** `/Users/vasilii/Desktop/code /muru-backend-local`  
**Ветка:** `feat/settings-site-contacts` (1A–2B на ней). НЕ merge/deploy.  
**Связь:** EPIC Settings → Часть **3A Backend**. STOP-2 (Часть 2) **ACCEPT**. Колонки `req_*` и `updateRequisitesSettings` уже есть с 1A — добавить CRM/public роуты + public projection + тесты.

### Цель
Публичный и CRM API для реквизитов. Без новой миграции.

### Уже есть
- `updateRequisitesSettings` + `requisitesSettingsInputSchema` в `site-settings.service.ts`
- Колонки `req_*` в `site_settings`
- CRM router: GET `/site`, PUT `/site/contacts` (owner)

### Сделать
1. `getPublicRequisites()` — проекция **только** req-полей (camelCase). Без contacts/socials.
2. CRM: `PUT /api/crm/settings/requisites` → `updateRequisitesSettings` (zod в controller, как contacts)
3. Public: `GET /api/content/requisites` → `getPublicRequisites()`, `ok()`, без auth
4. Тесты:
   - public projection без contact-ключей
   - update requisites не трогает contact SQL (уже есть частично — дополнить route test 401/403/owner 200)
   - GET public 200

### НЕ трогать
- Миграции / CDEK / YK
- Admin SPA / SF (3B/3C отдельно)
- Drive-скрипты

### DoD
- tsc + vitest site-settings + crm-settings (+ public smoke test если паттерн есть) зелёные
- Секретов нет; contacts не в public requisites

### Проверка
```bash
cd backend && npx tsc --noEmit
npx vitest run src/services/site-settings.service.test.ts src/routes/crm-settings.routes.test.ts
```

### Отчёт
файлы, тесты, подтверждение: public без contact_*; PUT только req_*
